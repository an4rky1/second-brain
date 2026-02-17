---
created: 2026-02-16
tags:
  - moc
  - databases
related:
  - MOC-Backend
  - MOC-Infrastructure
---

# 🗄️ MOC — Базы данных

> [!ABSTRACT] Обзор
> SQL и NoSQL базы данных, паттерны работы с данными.

---

## 🗂️ Навигация

### SQL
| БД | Файл | Статус |
|----|------|--------|
| PostgreSQL | [[PostgreSQL-Cheatsheet]] | 🟢 Готов |

### NoSQL
| БД | Файл | Статус |
|----|------|--------|
| Redis | [[Redis-Cheatsheet]] | 🟢 Готов |
| MongoDB | [[NoSQL-Cheatsheet]] | 🟢 Обзор NoSQL |

---

## 📊 Выбор базы данных

```
                        ┌─────────────────┐
                        │  Выбор БД       │
                        └────────┬────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
    ┌───────▼───────┐    ┌──────▼──────┐    ┌───────▼───────┐
    │  Реляционные  │    │ Документные │    │   Key-Value   │
    │   (SQL)       │    │  (NoSQL)    │    │   (Cache)     │
    └───────┬───────┘    └──────┬──────┘    └───────┬───────┘
            │                   │                   │
    ┌───────▼───────┐    ┌──────▼──────┐    ┌───────▼───────┐
    │  PostgreSQL   │    │  MongoDB    │    │    Redis      │
    │    MySQL      │    │             │    │   Memcached   │
    └───────────────┘    └─────────────┘    └───────────────┘
```

### Критерии выбора

| Требование | Рекомендация |
|------------|--------------|
| ACID транзакции | PostgreSQL |
| Гибкая схема | MongoDB |
| Кэширование | Redis |
| Full-text search | PostgreSQL + Elasticsearch |
| Time-series | TimescaleDB / InfluxDB |
| Graph | Neo4j |

---

## 🏗️ Паттерны работы с данными

### Connection Pool
```typescript
// Всегда используйте pooling
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
});
```

### Repository Pattern
```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Service   │ →  │  Repository  │ →  │   Database  │
└─────────────┘    └──────────────┘    └─────────────┘
```

### CQRS
```
┌─────────────────────────────────────────────────────┐
│                    Command Side                      │
│  Write → Validate → Event → Project → Read Model    │
└─────────────────────────────────────────────────────┘
```

---

## 📈 Индексы и производительность

### Типы индексов

| Тип | Когда использовать |
|-----|-------------------|
| B-Tree | По умолчанию, range queries |
| Hash | Точные совпадения |
| GIN | JSONB, full-text |
| GiST | Геоданные, ranges |
| BRIN | Большие таблицы, time-series |

---

## 🔗 Связанные заметки

- [[MOC-Backend]] — Backend разработка
- [[MOC-Infrastructure]] — Инфраструктура
- [[NestJS-Cheatsheet]] — NestJS
- [[PostgreSQL-Cheatsheet]] — PostgreSQL
- [[Redis-Cheatsheet]] — Redis
