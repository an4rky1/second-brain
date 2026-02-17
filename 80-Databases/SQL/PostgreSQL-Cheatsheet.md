---
created: 2026-02-16
tags:
  - cheat-sheet
  - database
  - sql
  - postgresql
aliases:
  - SQL Cheatsheet
  - PostgreSQL Reference
related:
  - Database-Design
  - NestJS-Cheatsheet
  - Performance-Optimization
---

# SQL & PostgreSQL — Полная шпаргалка

> [!SUMMARY] Обзор
> SQL и реляционные базы данных. PostgreSQL — продвинутая open-source СУБД с поддержкой JSON, полнотекстового поиска, гео-данных. Основы проектирования и оптимизации.

---

## 📚 Теория

### ACID

```
┌─────────────────────────────────────────────────────┐
│                  ACID Properties                      │
│                                                      │
│  Atomicity      →  Всё или ничего (транзакции)       │
│  Consistency    →  Валидное состояние (constraints)  │
│  Isolation      →  Параллельность без конфликтов     │
│  Durability     →  Сохранение после commit           │
└─────────────────────────────────────────────────────┘
```

### Уровни изоляции

```
┌─────────────────────────────────────────────────────────────────┐
│  Isolation Level    │ Dirty Read │ Non-Repeatable │ Phantom    │
├─────────────────────┼────────────┼────────────────┼────────────┤
│  Read Uncommitted   │     ✓      │       ✓        │     ✓      │
│  Read Committed     │     ✗      │       ✓        │     ✓      │
│  Repeatable Read    │     ✗      │       ✗        │     ✓      │
│  Serializable       │     ✗      │       ✗        │     ✗      │
└─────────────────────────────────────────────────────────────────┘

PostgreSQL default: Read Committed
```

---

## ⚡ Быстрый старт

```bash
# Установка PostgreSQL (Ubuntu)
sudo apt install postgresql postgresql-contrib

# Запуск
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Подключение
sudo -i -u postgres
psql

# Создание пользователя и БД
CREATE USER myuser WITH PASSWORD 'mypass';
CREATE DATABASE mydb OWNER myuser;
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

# Connection string
postgresql://user:password@localhost:5432/database
```

---

## 🔧 Практические примеры

### Основные команды

```sql
-- DDL (Data Definition Language)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

ALTER TABLE users ADD COLUMN age INTEGER;
ALTER TABLE users ALTER COLUMN age SET NOT NULL;
ALTER TABLE users DROP COLUMN age;

DROP TABLE IF EXISTS users CASCADE;

-- DML (Data Manipulation Language)
INSERT INTO users (email, name) 
VALUES ('john@example.com', 'John')
RETURNING id;

INSERT INTO users (email, name)
VALUES ('jane@example.com', 'Jane'),
       ('bob@example.com', 'Bob');

UPDATE users 
SET name = 'John Doe', updated_at = CURRENT_TIMESTAMP
WHERE id = 1
RETURNING *;

DELETE FROM users WHERE id = 1;

-- DQL (Data Query Language)
SELECT id, email, name 
FROM users 
WHERE created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 10 OFFSET 0;

-- DISTINCT
SELECT DISTINCT email FROM users;

-- COUNT
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT email) FROM users;
```

### JOINs

```sql
-- INNER JOIN (только совпадения)
SELECT u.id, u.name, o.id as order_id
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN (все из левой + совпадения)
SELECT u.id, u.name, o.id as order_id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN (все из правой + совпадения)
SELECT u.id, u.name, o.id as order_id
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN (все записи)
SELECT u.id, u.name, o.id as order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN (декартово произведение)
SELECT * FROM table1 CROSS JOIN table2;

-- SELF JOIN
SELECT e.name as employee, m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### Агрегация

```sql
-- GROUP BY
SELECT 
    status,
    COUNT(*) as count,
    AVG(total) as avg_total,
    SUM(total) as sum_total,
    MIN(total) as min_total,
    MAX(total) as max_total
FROM orders
GROUP BY status;

-- HAVING (фильтр после агрегации)
SELECT status, COUNT(*) as count
FROM orders
GROUP BY status
HAVING COUNT(*) > 10;

-- GROUPING SETS
SELECT 
    department, 
    role, 
    COUNT(*) as count
FROM employees
GROUP BY GROUPING SETS (
    (department, role),
    (department),
    (role),
    ()
);

-- CUBE (все комбинации)
SELECT department, role, COUNT(*)
FROM employees
GROUP BY CUBE (department, role);

-- ROLLUP (иерархия)
SELECT department, role, COUNT(*)
FROM employees
GROUP BY ROLLUP (department, role);
```

### Подзапросы и CTE

```sql
-- Подзапрос в WHERE
SELECT * FROM users
WHERE id IN (
    SELECT user_id FROM orders WHERE total > 100
);

-- Подзапрос в SELECT
SELECT 
    u.*,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u;

-- CTE (Common Table Expression)
WITH active_users AS (
    SELECT id, name FROM users WHERE active = true
),
user_orders AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
)
SELECT au.name, COALESCE(uo.order_count, 0) as orders
FROM active_users au
LEFT JOIN user_orders uo ON au.id = uo.user_id;

-- RECURSIVE CTE (иерархии)
WITH RECURSIVE hierarchy AS (
    SELECT id, name, parent_id, 0 as level
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id, h.level + 1
    FROM categories c
    INNER JOIN hierarchy h ON c.parent_id = h.id
)
SELECT * FROM hierarchy;
```

### Индексы

```sql
-- B-Tree (по умолчанию)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Hash (только =)
CREATE INDEX idx_sessions_token ON sessions USING HASH(token);

-- GIN (JSONB, массивы, full-text)
CREATE INDEX idx_users_data ON users USING GIN(data);
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);

-- GiST (гео, ranges)
CREATE INDEX idx_locations_geom ON locations USING GiST(geom);

-- BRIN (большие таблицы)
CREATE INDEX idx_logs_timestamp ON logs USING BRIN(timestamp);

-- Partial Index (условный)
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Expression Index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- ANALYZE (обновление статистики)
ANALYZE users;
ANALYZE VERBOSE;

-- EXPLAIN (план запроса)
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users;
```

### Транзакции

```sql
-- Базовая транзакция
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
-- или
ROLLBACK;

-- SAVEPOINT (точка сохранения)
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;

SAVEPOINT my_savepoint;

UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Ошибка!
ROLLBACK TO my_savepoint;

UPDATE accounts SET balance = balance + 100 WHERE id = 3;

COMMIT;

-- SERIALIZABLE (максимальная изоляция)
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ...
COMMIT;

-- SELECT FOR UPDATE (блокировка строк)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Другие транзакции не могут изменить эту строку
COMMIT;

-- SELECT FOR SHARE (блокировка на чтение)
SELECT * FROM posts WHERE id = 1 FOR SHARE;
```

### JSONB

```sql
-- Создание таблицы с JSONB
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes JSONB
);

-- Вставка
INSERT INTO products (name, attributes)
VALUES ('Laptop', '{"brand": "Apple", "price": 1000, "specs": {"ram": 16, "storage": 512}}');

-- Query
SELECT * FROM products
WHERE attributes->>'brand' = 'Apple';

SELECT * FROM products
WHERE attributes @> '{"brand": "Apple"}';  -- Contains

SELECT * FROM products
WHERE attributes ? 'price';  -- Has key

-- Извлечение
SELECT attributes->'specs'->>'ram' as ram FROM products;
SELECT attributes#>>'{specs,storage}' as storage FROM products;

-- Агрегация
SELECT jsonb_agg(name) FROM products;
SELECT jsonb_object_agg(name, attributes) FROM products;

-- Индекс
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);
```

### Window Functions

```sql
-- ROW_NUMBER
SELECT 
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;

-- RANK / DENSE_RANK
SELECT 
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) as rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;

-- LAG / LEAD
SELECT 
    date,
    revenue,
    LAG(revenue) OVER (ORDER BY date) as prev_revenue,
    LEAD(revenue) OVER (ORDER BY date) as next_revenue,
    revenue - LAG(revenue) OVER (ORDER BY date) as growth
FROM daily_revenue;

-- Running Total
SELECT 
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date) as running_total
FROM daily_revenue;

-- Moving Average
SELECT 
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7d
FROM daily_revenue;

-- FIRST_VALUE / LAST_VALUE
SELECT 
    name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) as highest_salary
FROM employees;
```

### Full-Text Search

```sql
-- Базовый поиск
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');

-- tsvector и tsquery
SELECT to_tsvector('english', 'The quick brown fox');
-- 'brown':3 'fox':4 'quick':2

SELECT to_tsquery('english', 'database & search');
-- 'databas' & 'search'

-- Операторы
'database & search'     -- AND
'database | search'     -- OR
'database & !search'    -- AND NOT
'database <-> search'   -- FOLLOWED BY

-- Индекс
CREATE INDEX idx_articles_content ON articles 
USING GIN(to_tsvector('english', content));

-- Ранжирование
SELECT 
    title,
    content,
    ts_rank(to_tsvector('english', content), query) as rank
FROM articles, to_tsquery('english', 'database') query
WHERE to_tsvector('english', content) @@ query
ORDER BY rank DESC;

-- Highlight
SELECT ts_headline(
    'english',
    content,
    to_tsquery('english', 'database'),
    'StartSel=<b>, StopSel=</b>'
) as highlighted
FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');
```

---

## 🎯 Best Practices

### ✅ Делать

```sql
-- 1. Использовать подготовленные выражения
-- В коде:
const result = await db.query('SELECT * FROM users WHERE id = $1', [id]);

-- 2. Правильные типы данных
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email CITEXT UNIQUE NOT NULL,  -- Case-insensitive
    created_at TIMESTAMPTZ DEFAULT NOW(),
    status SMALLINT DEFAULT 0,
    data JSONB DEFAULT '{}'::jsonb
);

-- 3. Индексы для частых запросов
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- 4. Foreign Keys
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total DECIMAL(10, 2) NOT NULL
);

-- 5. CHECK constraints
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    price DECIMAL(10, 2) CHECK (price >= 0),
    stock INTEGER CHECK (stock >= 0)
);
```

### ❌ Не делать

```sql
-- 1. SELECT *
SELECT * FROM users;  -- ❌
SELECT id, email, name FROM users;  -- ✅

-- 2. N+1 запросы
-- В коде:
for user in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)  -- ❌

-- Используйте JOIN или IN:
SELECT * FROM orders WHERE user_id IN (1, 2, 3);  -- ✅

-- 3. Функции в WHERE
WHERE LOWER(email) = 'test@example.com';  -- ❌ (не использует индекс)
WHERE email = 'test@example.com';  -- ✅ (или expression index)

-- 4. Отсутствие LIMIT
SELECT * FROM logs;  -- ❌
SELECT * FROM logs ORDER BY timestamp DESC LIMIT 100;  -- ✅

-- 5. Игнорирование EXPLAIN
-- Всегда проверяйте план запроса для медленных запросов
```

---

## 🔗 Связанные заметки

- [[Database-Design]] — Проектирование БД
- [[NestJS-Cheatsheet]] — NestJS + TypeORM
- [[Performance-Optimization]] — Оптимизация

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **EXPLAIN ANALYZE** — ваш друг для оптимизации
> 2. **Индексы не бесплатны** — замедляют запись
> 3. **VACUUM ANALYZE** — регулярно для production
> 4. **Connection Pool** — обязательно (PgBouncer)
> 5. **Репликация** — для чтения и отказоустойчивости

> [!INFO] Полезные команды
> ```bash
> # Backup
> pg_dump -U user dbname > backup.sql
> pg_dump -Fc dbname > backup.dump  # Custom format
> 
> # Restore
> psql -U user dbname < backup.sql
> pg_restore -d dbname backup.dump
> 
> # PSQL команды
> \l              # Список БД
> \c dbname       # Подключиться
> \dt             # Список таблиц
> \d table        # Описание таблицы
> \di             # Список индексов
> \du             # Список пользователей
> \q              # Выход
> ```
