---
created: 2026-02-17
updated: 2026-02-17
tags:
  - nestjs
  - prisma
  - database
  - orm
  - typescript
aliases:
  - NestJS Prisma
  - Prisma 7 Integration
  - Prisma ORM NestJS
related:
  - NestJS-MOC
  - NestJS-TypeORM
  - NestJS-Mongoose
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

## 📦 Prisma 7 Breaking Changes

| Изменение | Prisma 6 | Prisma 7 |
|-----------|----------|----------|
| **Runtime** | Rust + TypeScript | Только TypeScript |
| **Driver Adapters** | Опционально | **Обязательно** |
| **Provider** | `prisma-client-js` | `prisma-client` |
| **Output** | `node_modules` | Проект (`./src/generated`) |
| **Bundle size** | 100% | **10%** (90% меньше) |
| **Query execution** | 100% | **33%** (3x быстрее) |

---

## 🔧 Schema Configuration

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"  // Prisma 7: было "prisma-client-js"
  output   = "./src/generated/prisma"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Модели
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  password  String
  role      Role     @default(USER)
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?  // Soft delete

  @@index([email])
  @@index([role])
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([authorId])
  @@index([published])
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

---

## 🏛️ Prisma Module

```typescript
// prisma/prisma.module.ts
import { Module, Global, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule implements OnModuleInit, OnModuleDestroy {
  constructor(private prisma: PrismaService) {}

  async onModuleInit() {
    await this.prisma.$connect();
  }

  async onModuleDestroy() {
    await this.prisma.$disconnect();
  }
}
```

---

## 🔧 Prisma Service (Driver Adapters)

```typescript
// prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '../generated/prisma/client.js';
import { PrismaPg } from '@prisma/adapter-pg';
import { Pool } from 'pg';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    // Prisma 7: Driver Adapter обязателен!
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 20,                // Max connections
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    });

    const adapter = new PrismaPg(pool);

    // Prisma 7: нельзя создать без адаптера
    super({
      adapter,
      log: [
        { level: 'query', emit: 'event' },
        { level: 'error', emit: 'stdout' },
        { level: 'warn', emit: 'stdout' },
      ],
    });
  }

  async onModuleInit() {
    await this.$connect();

    // Query logging
    this.$on('query', ({ query, params, duration }) => {
      console.log(`Query: ${query}`);
      console.log(`Params: ${params}`);
      console.log(`Duration: ${duration}ms`);
    });
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

---

## 🔄 Prisma Extensions

```typescript
// prisma/prisma.extension.ts
import { Prisma } from '../generated/prisma/client.js';

// Soft Delete Extension
export const softDeleteExtension = Prisma.defineExtension((client) => {
  return client.$extends({
    model: {
      $allModels: {
        async softDelete(where: { id: number }) {
          const context = Prisma.getExtensionContext(this);
          return (context as any).update({
            where,
            data: { deletedAt: new Date() },
          });
        },
        async findNotDeleted(where: any = {}) {
          const context = Prisma.getExtensionContext(this);
          return (context as any).findMany({
            where: { ...where, deletedAt: null },
          });
        },
      },
    },
  });
});

// Pagination Extension
export const paginationExtension = Prisma.defineExtension((client) => {
  return client.$extends({
    model: {
      $allModels: {
        async paginate(options: {
          page: number;
          limit: number;
          where?: any;
          orderBy?: any;
          include?: any;
        }) {
          const { page, limit, where, orderBy, include } = options;
          const skip = (page - 1) * limit;

          const context = Prisma.getExtensionContext(this);
          const [data, total] = await Promise.all([
            (context as any).findMany({
              skip,
              take: limit,
              where,
              orderBy,
              include,
            }),
            (context as any).count({ where }),
          ]);

          return {
            data,
            meta: {
              total,
              page,
              limit,
              totalPages: Math.ceil(total / limit),
              hasNext: page * limit < total,
              hasPrev: page > 1,
            },
          };
        },
      },
    },
  });
});

// Usage in PrismaService
@Injectable()
export class PrismaService extends PrismaClient {
  constructor() {
    const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });

    super({ adapter });

    // Apply extensions
    this.$extends(softDeleteExtension);
    this.$extends(paginationExtension);
  }
}
```

---

## 🛠️ Repository Pattern

```typescript
// users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class UsersRepository {
  constructor(private prisma: PrismaService) {}

  async findAll(skip: number, take: number, where?: any) {
    return this.prisma.user.findMany({
      skip,
      take,
      where,
      orderBy: { createdAt: 'DESC' },
    });
  }

  async findById(id: number) {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string) {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(data: any) {
    return this.prisma.user.create({ data });
  }

  async update(id: number, data: any) {
    return this.prisma.user.update({ where: { id }, data });
  }

  async delete(id: number) {
    return this.prisma.user.delete({ where: { id } });
  }

  async softDelete(id: number) {
    return this.prisma.user.softDelete({ id });
  }

  async findNotDeleted(where?: any) {
    return this.prisma.user.findNotDeleted(where);
  }

  async paginate(page: number, limit: number, where?: any) {
    return this.prisma.user.paginate({
      page,
      limit,
      where,
      orderBy: { createdAt: 'DESC' },
    });
  }
}
```

---

## 🛠️ Service Layer

```typescript
// users/users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
} from '@nestjs/common';
import { UsersRepository } from './users.repository';

@Injectable()
export class UsersService {
  constructor(private usersRepository: UsersRepository) {}

  // ─────────────────────────────────────────────────────
  // FIND ALL с пагинацией
  // ─────────────────────────────────────────────────────
  async findAll(page = 1, limit = 10, search?: string) {
    const where: any = { deletedAt: null };

    if (search) {
      where.OR = [
        { name: { contains: search } },
        { email: { contains: search } },
      ];
    }

    return this.usersRepository.paginate(page, limit, where);
  }

  // ─────────────────────────────────────────────────────
  // FIND ONE
  // ─────────────────────────────────────────────────────
  async findOne(id: number) {
    const user = await this.usersRepository.findById(id);

    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }

    return user;
  }

  // ─────────────────────────────────────────────────────
  // CREATE
  // ─────────────────────────────────────────────────────
  async create(createUserDto: CreateUserDto) {
    // Check for duplicates
    const existing = await this.usersRepository.findByEmail(createUserDto.email);
    if (existing) {
      throw new ConflictException('Email already exists');
    }

    // Hash password
    const hashedPassword = await this.hashPassword(createUserDto.password);

    return this.usersRepository.create({
      ...createUserDto,
      password: hashedPassword,
    });
  }

  // ─────────────────────────────────────────────────────
  // UPDATE
  // ─────────────────────────────────────────────────────
  async update(id: number, updateUserDto: UpdateUserDto) {
    await this.findOne(id); // Check exists

    // If email changes, check for duplicates
    if (updateUserDto.email) {
      const existing = await this.usersRepository.findByEmail(updateUserDto.email);
      if (existing && existing.id !== id) {
        throw new ConflictException('Email already exists');
      }
    }

    return this.usersRepository.update(id, updateUserDto);
  }

  // ─────────────────────────────────────────────────────
  // DELETE (Soft)
  // ─────────────────────────────────────────────────────
  async remove(id: number) {
    await this.findOne(id);
    return this.usersRepository.softDelete(id);
  }

  // ─────────────────────────────────────────────────────
  // TRANSACTION
  // ─────────────────────────────────────────────────────
  async transferCredits(fromId: number, toId: number, amount: number) {
    return this.usersRepository.prisma.$transaction(async (tx) => {
      const [fromUser, toUser] = await Promise.all([
        tx.user.findUnique({ where: { id: fromId } }),
        tx.user.findUnique({ where: { id: toId } }),
      ]);

      if (!fromUser || !toUser) {
        throw new NotFoundException('User not found');
      }

      if (fromUser.credits < amount) {
        throw new ConflictException('Insufficient credits');
      }

      await Promise.all([
        tx.user.update({
          where: { id: fromId },
          data: { credits: { decrement: amount } },
        }),
        tx.user.update({
          where: { id: toId },
          data: { credits: { increment: amount } },
        }),
      ]);
    });
  }

  // ─────────────────────────────────────────────────────
  // HELPER
  // ─────────────────────────────────────────────────────
  private async hashPassword(password: string): Promise<string> {
    const bcrypt = require('bcrypt');
    return bcrypt.hash(password, 10);
  }
}
```

---

## 📋 Environment

```bash
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# Prisma Postgres (Prisma Cloud)
# DATABASE_URL="prisma://accelerate.prisma-data.net/?api_key=xxx"

# Vercel/Neon (Serverless)
# DATABASE_URL="postgresql://user:password@us-east-1.aws.neon.tech/mydb?sslmode=require"

# Connection pool settings
DATABASE_POOL_MAX=20
DATABASE_POOL_MIN=5
DATABASE_TIMEOUT=2000
```

---

## 🚀 Scripts

```json
{
  "scripts": {
    "db:migrate:dev": "prisma migrate dev",
    "db:migrate:deploy": "prisma migrate deploy",
    "db:generate": "prisma generate",
    "db:studio": "prisma studio",
    "db:seed": "prisma db seed",
    "db:reset": "prisma migrate reset",
    "db:format": "prisma format",
    "db:validate": "prisma validate"
  }
}
```

---

## ✅ Best Practices

```typescript
// ✅ Prisma types вместо DTO
import { Prisma } from '../generated/prisma/client.js';
async create(data: Prisma.UserCreateInput) { ... }

// ✅ Transaction для критичных операций
await prisma.$transaction(async (tx) => { ... });

// ✅ Include для связанных данных
const user = await prisma.user.findUnique({
  where: { id },
  include: { posts: true },
});

// ✅ Select для производительности
const users = await prisma.user.findMany({
  select: { id: true, email: true, name: true },
});

// ✅ Pagination с cursor для больших таблиц
const posts = await prisma.post.findMany({
  take: 10,
  skip: 1,
  cursor: { id: lastPostId },
});

// ✅ Prisma 7: Driver Adapter обязателен
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
const prisma = new PrismaClient({ adapter });

// ❌ N+1 проблема - используйте Promise.all
const users = await prisma.user.findMany();
const posts = await Promise.all(
  users.map(user => prisma.post.findMany({ where: { authorId: user.id } }))
);

// ❌ Prisma 7: нельзя без адаптера
const prisma = new PrismaClient(); // TypeError!
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-TypeORM]] — TypeORM
- [[NestJS-Mongoose]] — MongoDB/Mongoose
- [[PostgreSQL-Cheatsheet]] — PostgreSQL
