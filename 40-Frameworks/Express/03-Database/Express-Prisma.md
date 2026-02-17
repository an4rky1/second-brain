---
created: 2026-02-17
tags:
  - nestjs
  - prisma
  - database
  - orm
aliases:
  - NestJS Prisma
  - Prisma 7 Integration
related:
  - NestJS-MOC
  - Database-TypeORM
  - PostgreSQL-Cheatsheet
---

# NestJS — Prisma 7 Integration

> [!SUMMARY] Обзор
> Prisma 7 — современная ORM для Node.js с полной типизацией. Driver adapters, TypeScript runtime, extensions.

---

## ⚡ Быстрый старт

```bash
# Установка
npm install prisma --save-dev
npm install @prisma/client @prisma/adapter-pg pg

# Инициализация
npx prisma init --db --output ./src/generated/prisma

# Миграции
npx prisma migrate dev --name init
npx prisma generate
```

---

## 📚 Теория

### Prisma 7 Breaking Changes

| Изменение | Prisma 6 | Prisma 7 |
|-----------|----------|----------|
| **Runtime** | Rust + TypeScript | Только TypeScript |
| **Driver Adapters** | Опционально | **Обязательно** |
| **Provider** | `prisma-client-js` | `prisma-client` |
| **Output** | `node_modules` | Проект (`./src/generated`) |

### Производительность

| Метрика | Улучшение |
|---------|-----------|
| Bundle size | **90% меньше** |
| Query execution | **3x быстрее** |
| Type evaluation | **98% меньше типов** |
| Full type check | **70% быстрее** |

---

## 🔧 Практические примеры

Смотрите подробную документацию:

- [[Database-Prisma-Full]] — полная версия с примерами кода

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[Database-TypeORM]] — TypeORM интеграция
- [[PostgreSQL-Cheatsheet]] — PostgreSQL
