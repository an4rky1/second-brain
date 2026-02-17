---
created: 2026-02-17
updated: 2026-02-17
tags:
  - express
  - moс
  - backend
  - nodejs
aliases:
  - Express.js Index
  - Express MOC
related:
  - NodeJS-Cheatsheet
  - TypeScript-Cheatsheet
  - NestJS-MOC
---

# 🚂 Express.js — Индекс

> [!SUMMARY] Обзор
> Express.js — минималистичный веб-фреймворк для Node.js. Гибкий, мощный, с огромной экосистемой middleware.

---

## 🗂️ Навигация

### 01 — Core (Основы)

| Тема | Файл | Описание |
|------|------|----------|
| 📖 Getting Started | [[01-Core/Express-Getting-Started]] | Установка, первый сервер, роутинг |
| 🎮 Routing | [[01-Core/Express-Routing]] | Routes, params, query, handlers |
| 📝 Request/Response | [[01-Core/Express-Request-Response]] | req, res объекты, методы |

### 02 — Middleware

| Тема | Файл | Описание |
|------|------|----------|
| 🔌 Core Middleware | [[02-Middleware/Express-Middleware-Core]] | Built-in, custom, third-party |
| 🚨 Error Handling | [[02-Middleware/Express-Error-Handling]] | Error handlers, AppError |
| 📋 Validation | [[02-Middleware/Express-Validation]] | Joi, Zod, express-validator |
| 📝 Logging | [[02-Middleware/Express-Logging]] | Morgan, Winston |

### 03 — Database

| Тема | Файл | Описание |
|------|------|----------|
| 🔷 Prisma | [[03-Database/Express-Prisma]] | Prisma 7 ORM integration |
| 🗄️ TypeORM | [[03-Database/Express-TypeORM]] | TypeORM integration |
| 🔴 Redis | [[03-Database/Express-Redis]] | Кэширование, сессии |

### 04 — Authentication

| Тема | Файл | Описание |
|------|------|----------|
| 🔐 Passport.js | [[04-Auth/Express-Passport]] | JWT, Local, OAuth стратегии |
| 🛡️ JWT Auth | [[04-Auth/Express-JWT-Auth]] | JWT без Passport |
| 🍪 Sessions | [[04-Auth/Express-Sessions]] | express-session, Redis store |
| 🔗 OAuth | [[04-Auth/Express-OAuth]] | Google, GitHub, Facebook |

### 05 — Advanced

| Тема | Файл | Описание |
|------|------|----------|
| 🏗️ Architecture | [[05-Advanced/Express-Architecture]] | Структура, паттерны, best practices |
| 🧪 Testing | [[05-Advanced/Express-Testing]] | Jest, Supertest |
| 🚀 Deployment | [[05-Advanced/Express-Deployment]] | Docker, PM2, production |
| 📧 Mail | [[05-Advanced/Express-Mail]] | Email отправка |

---

## 📚 Быстрый старт

```bash
# Создание проекта
mkdir my-app && cd my-app
npm init -y
npm install express

# TypeScript
npm install -D typescript ts-node @types/node @types/express
npx tsc --init

# Запуск
npx ts-node src/index.ts
```

---

## 🔗 Связанные заметки

- [[NodeJS-Cheatsheet]] — Node.js основы
- [[TypeScript-Cheatsheet]] — TypeScript
- [[NestJS-MOC]] — NestJS (фреймворк на Express)
- [[Docker-Cheatsheet]] — Контейнеризация
