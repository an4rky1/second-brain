---
created: 2026-02-17
tags:
  - database
  - optimization
  - performance
  - indexing
aliases:
  - Database Optimization
  - Database Performance Tuning
related:
  - PostgreSQL-Cheatsheet
  - MySQL-Cheatsheet
  - MOC-Databases
---

# Database Optimization

> [!SUMMARY] Обзор
> Оптимизация баз данных: индексы, query optimization, caching, partitioning, connection pooling.

---

## 📊 Query Optimization

### EXPLAIN / EXPLAIN ANALYZE

```sql
-- PostgreSQL
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';

-- MySQL
EXPLAIN
SELECT * FROM users WHERE email = 'test@example.com';

-- Output interpretation
Seq Scan on users  -- ❌ Full table scan
Index Scan using idx_email on users  -- ✅ Using index

-- Cost estimation
-- cost=0.43..8.45  (startup..total)
-- rows=1  (estimated rows)
-- width=32  (average row width)
-- actual time=0.021..0.021  (actual execution time)
-- rows=1  (actual rows returned)
-- loops=1  (number of times executed)
```

### Common Issues

```sql
-- ❌ SELECT * (fetches all columns)
SELECT * FROM users;

-- ✅ Select only needed columns
SELECT id, email, name FROM users;

-- ❌ Function on indexed column
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- ✅ Use functional index or store normalized
CREATE INDEX idx_email_lower ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- ❌ Leading wildcard
SELECT * FROM users WHERE name LIKE '%john%';

-- ✅ Trailing wildcard only
SELECT * FROM users WHERE name LIKE 'john%';

-- ❌ OR on non-indexed columns
SELECT * FROM users WHERE email = 'test@example.com' OR name = 'John';

-- ✅ UNION ALL with indexed columns
SELECT * FROM users WHERE email = 'test@example.com'
UNION ALL
SELECT * FROM users WHERE name = 'John';

-- ❌ N+1 queries
-- Application code:
for user in users:
    posts = db.query("SELECT * FROM posts WHERE user_id = ?", user.id)

-- ✅ JOIN or batch loading
SELECT u.*, p.* FROM users u
LEFT JOIN posts p ON u.id = p.user_id;
```

---

## 🗂️ Indexing Strategies

### Index Types

```sql
-- B-Tree (default, range queries)
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_created ON users(created_at);

-- Hash (exact match only)
CREATE INDEX idx_email_hash ON users USING HASH(email);

-- Composite (multiple columns)
CREATE INDEX idx_role_created ON users(role, created_at);
-- Order matters! Can be used for:
-- WHERE role = ? AND created_at > ?
-- WHERE role = ?
-- But NOT for WHERE created_at > ?

-- Partial/Filtered (PostgreSQL)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = TRUE;

-- Expression Index (PostgreSQL)
CREATE INDEX idx_email_lower ON users (LOWER(email));

-- Covering Index (includes extra columns)
CREATE INDEX idx_email_covering ON users(email) INCLUDE (name, role);

-- Full-text (text search)
CREATE FULLTEXT INDEX idx_content ON posts(title, content);
```

### When to Index

```sql
-- ✅ Good candidates for indexing
-- Columns in WHERE clauses
-- Columns in JOIN conditions
-- Columns in ORDER BY
-- Columns in GROUP BY
-- Foreign keys

-- ❌ Bad candidates
-- Low cardinality (boolean, gender)
-- Small tables (< 1000 rows)
-- Frequently updated columns
-- Columns with many NULLs (unless partial index)
```

### Index Maintenance

```sql
-- PostgreSQL
REINDEX TABLE users;
REINDEX DATABASE mydb;

-- MySQL
OPTIMIZE TABLE users;
ANALYZE TABLE users;

-- Check index usage (PostgreSQL)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Find unused indexes (PostgreSQL)
SELECT indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE '%_pkey';
```

---

## 💾 Connection Pooling

### PgBouncer (PostgreSQL)

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = userlist.txt

# Pool settings
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 5

# Timeouts
server_lifetime = 3600
server_idle_timeout = 600
server_connect_timeout = 15
```

### Application Pool Settings

```typescript
// Node.js (pg)
const pool = new Pool({
  max: 20,              // Max connections
  min: 5,               // Min connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Node.js (mysql2)
const pool = mysql.createPool({
  connectionLimit: 20,
  maxIdle: 10,
  idleTimeout: 60000,
  enableKeepAlive: true,
});

// Prisma
// datasource.db {
//   provider = "postgresql"
//   url      = env("DATABASE_URL")
//   connectionLimit = 20
// }
```

---

## 📦 Caching Strategies

### Query Cache (Application Level)

```typescript
// Redis cache
async function getCachedUser(id: string) {
  const cacheKey = `user:${id}`;
  
  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Query database
  const user = await db.user.findUnique({ where: { id } });
  
  // Cache for 5 minutes
  await redis.setex(cacheKey, 300, JSON.stringify(user));
  
  return user;
}

// Cache-aside pattern
async function getWithCache(key: string, queryFn: () => Promise<any>, ttl = 300) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  
  const data = await queryFn();
  await redis.setex(key, ttl, JSON.stringify(data));
  return data;
}
```

### Materialized Views (PostgreSQL)

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW user_stats AS
SELECT 
  u.id,
  u.email,
  COUNT(p.id) as post_count,
  COUNT(c.id) as comment_count,
  MAX(p.created_at) as last_post_at
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
LEFT JOIN comments c ON u.id = c.user_id
GROUP BY u.id, u.email;

-- Create index on materialized view
CREATE INDEX idx_user_stats_post_count ON user_stats(post_count DESC);

-- Refresh materialized view
REFRESH MATERIALIZED VIEW user_stats;

-- Refresh concurrently (allows reads during refresh)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;

-- Schedule refresh (cron)
CREATE EXTENSION pg_cron;
SELECT cron.schedule('refresh-user-stats', '0 * * * *', 
  'REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats');
```

---

## 📊 Partitioning

### Range Partitioning (PostgreSQL 10+)

```sql
-- Create partitioned table
CREATE TABLE logs (
  id SERIAL,
  message TEXT,
  created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE logs_2026_01 PARTITION OF logs
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE logs_2026_02 PARTITION OF logs
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Query automatically uses correct partition
SELECT * FROM logs WHERE created_at > '2026-01-15';

-- Drop old partition (fast!)
DROP TABLE logs_2026_01;
```

### Hash Partitioning

```sql
CREATE TABLE users (
  id SERIAL,
  email TEXT,
  data JSONB
) PARTITION BY HASH (id);

CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## 🔧 Configuration Tuning

### PostgreSQL (postgresql.conf)

```conf
# Memory
shared_buffers = 4GB              # 25% of RAM
effective_cache_size = 12GB       # 75% of RAM
work_mem = 64MB                   # Per-operation memory
maintenance_work_mem = 1GB        # For VACUUM, CREATE INDEX

# Connections
max_connections = 200
superuser_reserved_connections = 3

# WAL
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

# Query planning
random_page_cost = 1.1            # For SSD
effective_io_concurrency = 200    # For SSD
```

### MySQL (my.cnf)

```conf
[mysqld]
# Memory
innodb_buffer_pool_size = 4G      # 70% of RAM
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2

# Connections
max_connections = 200
thread_cache_size = 50

# Query cache (MySQL 5.7, removed in 8.0)
query_cache_type = 1
query_cache_size = 64M

# InnoDB
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1
```

---

## 📈 Monitoring

### PostgreSQL

```sql
-- Slow queries
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Table sizes
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Index usage
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Locks
SELECT 
  pid,
  usename,
  query,
  state,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE state != 'idle';
```

### MySQL

```sql
-- Slow queries
SELECT * FROM mysql.slow_log;
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- Table sizes
SELECT 
  table_name,
  ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.TABLES
WHERE table_schema = 'mydb'
ORDER BY (data_length + index_length) DESC;

-- Index usage
SHOW INDEX FROM users;

-- Active connections
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';
```

---

## 🔗 Связанные заметки

- [[PostgreSQL-Cheatsheet]] — PostgreSQL
- [[MySQL-Cheatsheet]] — MySQL
- [[Redis-Cheatsheet]] — Redis caching
- [[MOC-Databases]] — Databases MOC
