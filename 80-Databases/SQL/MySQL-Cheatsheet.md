---
created: 2026-02-17
tags:
  - database
  - mysql
  - sql
  - rdbms
aliases:
  - MySQL Cheatsheet
  - MySQL SQL Reference
related:
  - PostgreSQL-Cheatsheet
  - SQL-Cheatsheet
  - Database-Design
---

# MySQL — Полная шпаргалка

> [!SUMMARY] Обзор
> MySQL — популярная реляционная СУБД. SQL запросы, индексы, транзакции, оптимизация.

---

## 📦 Установка

```bash
# Docker
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_USER=user \
  -e MYSQL_PASSWORD=password \
  -p 3306:3306 \
  mysql:8

# Connect
mysql -u root -p
mysql -u user -p mydb
```

---

## 📝 Basic SQL

### CREATE

```sql
-- Database
CREATE DATABASE IF NOT EXISTS mydb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- Table
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(100) NOT NULL,
  password VARCHAR(255) NOT NULL,
  role ENUM('user', 'admin', 'moderator') DEFAULT 'user',
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  INDEX idx_email (email),
  INDEX idx_role (role)
) ENGINE=InnoDB;

-- Table with foreign key
CREATE TABLE posts (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  user_id INT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE,
  
  INDEX idx_user_id (user_id)
) ENGINE=InnoDB;
```

### SELECT

```sql
-- Basic
SELECT id, email, name FROM users;
SELECT * FROM users;

-- WHERE
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE email = 'test@example.com';
SELECT * FROM users WHERE role IN ('admin', 'moderator');
SELECT * FROM users WHERE created_at > '2026-01-01';
SELECT * FROM users WHERE name LIKE '%john%';
SELECT * FROM users WHERE email LIKE 'john%@example.com';

-- ORDER BY
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users ORDER BY role, name ASC;

-- LIMIT / OFFSET
SELECT * FROM users LIMIT 10;
SELECT * FROM users LIMIT 10 OFFSET 20;  -- Page 3 with 10 per page

-- DISTINCT
SELECT DISTINCT role FROM users;
SELECT DISTINCT role, is_active FROM users;
```

### JOIN

```sql
-- INNER JOIN
SELECT u.name, p.title
FROM users u
INNER JOIN posts p ON u.id = p.user_id;

-- LEFT JOIN
SELECT u.name, p.title
FROM users u
LEFT JOIN posts p ON u.id = p.user_id;

-- RIGHT JOIN
SELECT u.name, p.title
FROM users u
RIGHT JOIN posts p ON u.id = p.user_id;

-- Multiple JOINs
SELECT u.name, p.title, c.content
FROM users u
JOIN posts p ON u.id = p.user_id
JOIN comments c ON p.id = c.post_id
WHERE u.role = 'admin';

-- Subquery
SELECT * FROM users
WHERE id IN (SELECT user_id FROM posts WHERE created_at > '2026-01-01');
```

### INSERT

```sql
-- Single row
INSERT INTO users (email, name, password)
VALUES ('test@example.com', 'John', 'hashed_password');

-- Multiple rows
INSERT INTO users (email, name, password) VALUES
  ('test1@example.com', 'John', 'hash1'),
  ('test2@example.com', 'Jane', 'hash2'),
  ('test3@example.com', 'Bob', 'hash3');

-- INSERT ... SELECT
INSERT INTO users_backup (email, name, password)
SELECT email, name, password FROM users WHERE is_active = TRUE;

-- INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO users (email, name)
VALUES ('test@example.com', 'John')
ON DUPLICATE KEY UPDATE name = 'John', updated_at = CURRENT_TIMESTAMP;
```

### UPDATE

```sql
-- Single row
UPDATE users SET name = 'Jane' WHERE id = 1;

-- Multiple rows
UPDATE users SET is_active = FALSE WHERE role = 'banned';

-- Multiple columns
UPDATE users 
SET name = 'Jane', email = 'jane@example.com'
WHERE id = 1;

-- With JOIN
UPDATE users u
JOIN posts p ON u.id = p.user_id
SET u.role = 'active'
WHERE p.created_at > '2026-01-01';
```

### DELETE

```sql
-- Single row
DELETE FROM users WHERE id = 1;

-- Multiple rows
DELETE FROM users WHERE is_active = FALSE;

-- With JOIN
DELETE u FROM users u
JOIN posts p ON u.id = p.user_id
WHERE p.created_at < '2025-01-01';

-- TRUNCATE (faster, resets auto-increment)
TRUNCATE TABLE users;
```

---

## 📊 Aggregation

```sql
-- COUNT
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT role) FROM users;

-- SUM / AVG / MIN / MAX
SELECT SUM(price) FROM orders;
SELECT AVG(price) FROM orders;
SELECT MIN(price), MAX(price) FROM orders;

-- GROUP BY
SELECT role, COUNT(*) as count
FROM users
GROUP BY role;

SELECT role, COUNT(*) as count, AVG(created_at) as avg_date
FROM users
GROUP BY role
HAVING COUNT(*) > 10;

-- GROUP BY with multiple columns
SELECT role, is_active, COUNT(*) as count
FROM users
GROUP BY role, is_active
WITH ROLLUP;
```

---

## 🔑 Indexes

```sql
-- Create index
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_role_active ON users(role, is_active);

-- Unique index
CREATE UNIQUE INDEX idx_unique_email ON users(email);

-- Full-text index
CREATE FULLTEXT INDEX idx_content ON posts(title, content);

-- Show indexes
SHOW INDEX FROM users;

-- Drop index
DROP INDEX idx_email ON users;

-- Analyze index usage
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

---

## 🔒 Transactions

```sql
-- Start transaction
START TRANSACTION;

-- Multiple statements
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Commit
COMMIT;

-- Rollback
ROLLBACK;

-- Savepoint
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT sp1;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
ROLLBACK TO sp1;  -- Rollback to savepoint
COMMIT;

-- Set isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- Default
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## 👤 Users & Permissions

```sql
-- Create user
CREATE USER 'app'@'localhost' IDENTIFIED BY 'password';
CREATE USER 'app'@'%' IDENTIFIED BY 'password';  -- Any host

-- Grant privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app'@'localhost';
GRANT ALL PRIVILEGES ON mydb.* TO 'app'@'localhost';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost';

-- Revoke privileges
REVOKE DELETE ON mydb.* FROM 'app'@'localhost';

-- Show privileges
SHOW GRANTS FOR 'app'@'localhost';

-- Drop user
DROP USER 'app'@'localhost';

-- Flush privileges
FLUSH PRIVILEGES;
```

---

## 🔧 Administration

```sql
-- Show databases
SHOW DATABASES;

-- Show tables
SHOW TABLES;
SHOW TABLES FROM mydb;

-- Describe table
DESCRIBE users;
SHOW CREATE TABLE users;

-- Show status
SHOW STATUS;
SHOW VARIABLES;

-- Process list
SHOW PROCESSLIST;
KILL 123;  -- Kill process ID

-- Backup
mysqldump -u root -p mydb > backup.sql
mysqldump -u root -p --all-databases > all.sql

-- Restore
mysql -u root -p mydb < backup.sql

-- Optimize tables
OPTIMIZE TABLE users;
ANALYZE TABLE users;
CHECK TABLE users;
```

---

## 📋 Environment

```bash
# .env
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_DATABASE=mydb
MYSQL_USER=user
MYSQL_PASSWORD=password
MYSQL_ROOT_PASSWORD=root

# Connection pool
DB_POOL_MAX=20
DB_POOL_MIN=5
```

---

## 🔗 Связанные заметки

- [[PostgreSQL-Cheatsheet]] — PostgreSQL
- [[SQL-Cheatsheet]] — SQL basics
- [[Database-Design]] — Database design
- [[Database-Optimization]] — Database optimization
