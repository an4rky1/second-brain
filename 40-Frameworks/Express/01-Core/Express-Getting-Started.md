---
created: 2026-02-17
tags:
  - express
  - getting-started
  - basics
  - server
aliases:
  - Express Getting Started
  - Express First Server
related:
  - Express-MOC
  - Express-Routing
---

# Express.js — Getting Started

> [!SUMMARY] Обзор
> Быстрый старт с Express.js: установка, первый сервер, базовая конфигурация.

---

## ⚡ Установка

```bash
# Инициализация проекта
mkdir my-app && cd my-app
npm init -y

# Установка Express
npm install express

# TypeScript (опционально)
npm install -D typescript ts-node @types/node @types/express
npx tsc --init
```

---

## 📦 Первый сервер

```typescript
// src/index.ts
import express, { Request, Response } from 'express';

const app = express();
const PORT = process.env.PORT || 3000;

// Базовый route
app.get('/', (req: Request, res: Response) => {
  res.json({ message: 'Hello World!' });
});

// Запуск сервера
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

---

## 🔧 Конфигурация

### Basic Setup

```typescript
// src/app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';

const app = express();

// Security
app.use(helmet());
app.use(cors());

// Body parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Logging
app.use(morgan('dev'));

// Routes
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

export default app;
```

### Environment Config

```typescript
// src/config/env.ts
import dotenv from 'dotenv';
dotenv.config();

export const config = {
  port: process.env.PORT || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  dbUrl: process.env.DATABASE_URL || '',
  jwtSecret: process.env.JWT_SECRET || '',
};
```

### Server Entry Point

```typescript
// src/index.ts
import app from './app';
import { config } from './config/env';

const PORT = config.port;

app.listen(PORT, () => {
  console.log(`
    ╔════════════════════════════════════╗
    ║   Server running on port ${PORT}    ║
    ║   Environment: ${config.nodeEnv}           ║
    ║   http://localhost:${PORT}          ║
    ╚════════════════════════════════════╝
  `);
});
```

---

## 🎮 Basic Routing

```typescript
// src/routes/index.ts
import { Router } from 'express';

const router = Router();

// GET /
router.get('/', (req, res) => {
  res.json({ message: 'Welcome to API' });
});

// GET /about
router.get('/about', (req, res) => {
  res.json({ name: 'My API', version: '1.0.0' });
});

// GET /health
router.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

export default router;
```

---

## 📁 Структура проекта

```
project/
├── src/
│   ├── index.ts          # Entry point
│   ├── app.ts            # App configuration
│   ├── config/
│   │   └── env.ts        # Environment config
│   ├── routes/
│   │   └── index.ts      # Routes
│   ├── controllers/
│   │   └── ...
│   ├── services/
│   │   └── ...
│   ├── middleware/
│   │   └── ...
│   └── utils/
│       └── ...
├── .env
├── .gitignore
├── package.json
└── tsconfig.json
```

---

## 🏃 Scripts

```json
{
  "name": "my-express-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "ts-node-dev --respawn src/index.ts",
    "start": "node dist/index.js",
    "build": "tsc",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "express": "^4.18.0",
    "cors": "^2.8.5",
    "helmet": "^7.0.0",
    "morgan": "^1.10.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/node": "^20.0.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Routing]] — Роутинг
- [[Express-Request-Response]] — Request/Response объекты
