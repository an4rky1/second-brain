---
created: 2026-02-17
tags:
  - express
  - architecture
  - patterns
  - structure
aliases:
  - Express Architecture
  - Express Project Structure
related:
  - Express-MOC
  - Express-Middleware-Core
---

# Express.js — Architecture & Patterns

> [!SUMMARY] Обзор
> Архитектурные паттерны и структура проекта для Express.js приложений.

---

## 🏗️ Структура проекта

### Classic MVC

```
src/
├── index.ts              # Entry point
├── app.ts                # App configuration
├── config/
│   ├── database.ts
│   ├── app.config.ts
│   └── env.config.ts
├── routes/
│   ├── index.ts          # Routes aggregator
│   ├── users.routes.ts
│   └── auth.routes.ts
├── controllers/
│   ├── users.controller.ts
│   └── auth.controller.ts
├── services/
│   ├── users.service.ts
│   └── auth.service.ts
├── models/
│   └── user.model.ts
├── middleware/
│   ├── auth.middleware.ts
│   ├── error.middleware.ts
│   └── validation.middleware.ts
├── utils/
│   ├── AppError.ts
│   ├── asyncHandler.ts
│   └── logger.ts
└── types/
    └── express.d.ts
```

### Feature-based Structure

```
src/
├── index.ts
├── app.ts
├── config/
│   └── ...
├── common/
│   ├── middleware/
│   ├── decorators/
│   └── utils/
├── modules/
│   ├── users/
│   │   ├── users.routes.ts
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.model.ts
│   │   └── users.validator.ts
│   ├── auth/
│   │   ├── auth.routes.ts
│   │   ├── auth.controller.ts
│   │   └── auth.service.ts
│   └── posts/
│       └── ...
└── app.ts
```

---

## 📐 Паттерны

### Controller Pattern

```typescript
// controllers/users.controller.ts
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../services/users.service';

export class UsersController {
  constructor(private userService: UserService) {}

  getAll = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const users = await this.userService.findAll();
      res.json({ data: users });
    } catch (error) {
      next(error);
    }
  };

  getOne = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.userService.findById(Number(req.params.id));
      res.json({ data: user });
    } catch (error) {
      next(error);
    }
  };

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.userService.create(req.body);
      res.status(201).json({ data: user });
    } catch (error) {
      next(error);
    }
  };

  update = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.userService.update(
        Number(req.params.id),
        req.body
      );
      res.json({ data: user });
    } catch (error) {
      next(error);
    }
  };

  delete = async (req: Request, res: Response, next: NextFunction) => {
    try {
      await this.userService.delete(Number(req.params.id));
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  };
}
```

### Service Layer Pattern

```typescript
// services/users.service.ts
import { UserRepository } from '../repositories/users.repository';
import { NotFoundError, ConflictError } from '../utils/AppError';

export class UserService {
  constructor(private userRepository: UserRepository) {}

  async findAll() {
    return this.userRepository.findAll();
  }

  async findById(id: number) {
    const user = await this.userRepository.findById(id);
    
    if (!user) {
      throw new NotFoundError('User not found');
    }
    
    return user;
  }

  async create(data: CreateUserDto) {
    const existing = await this.userRepository.findByEmail(data.email);
    if (existing) {
      throw new ConflictError('Email already exists');
    }

    const hashedPassword = await this.hashPassword(data.password);

    return this.userRepository.create({
      ...data,
      password: hashedPassword,
    });
  }

  async update(id: number, data: UpdateUserDto) {
    await this.findById(id);

    if (data.email) {
      const existing = await this.userRepository.findByEmail(data.email);
      if (existing && existing.id !== id) {
        throw new ConflictError('Email already exists');
      }
    }

    return this.userRepository.update(id, data);
  }

  async delete(id: number) {
    await this.findById(id);
    await this.userRepository.delete(id);
  }

  private async hashPassword(password: string): Promise<string> {
    const bcrypt = require('bcrypt');
    return bcrypt.hash(password, 10);
  }
}
```

### Repository Pattern

```typescript
// repositories/users.repository.ts
import { Pool } from 'pg';

export class UsersRepository {
  constructor(private pool: Pool) {}

  async findAll() {
    const { rows } = await this.pool.query('SELECT * FROM users');
    return rows;
  }

  async findById(id: number) {
    const { rows } = await this.pool.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return rows[0];
  }

  async findByEmail(email: string) {
    const { rows } = await this.pool.query(
      'SELECT * FROM users WHERE email = $1',
      [email]
    );
    return rows[0];
  }

  async create(data: CreateUserDto) {
    const { rows } = await this.pool.query(
      `INSERT INTO users (email, password, name)
       VALUES ($1, $2, $3)
       RETURNING *`,
      [data.email, data.password, data.name]
    );
    return rows[0];
  }

  async update(id: number, data: UpdateUserDto) {
    const { rows } = await this.pool.query(
      `UPDATE users 
       SET email = COALESCE($1, email),
           name = COALESCE($2, name)
       WHERE id = $3
       RETURNING *`,
      [data.email, data.name, id]
    );
    return rows[0];
  }

  async delete(id: number) {
    await this.pool.query('DELETE FROM users WHERE id = $1', [id]);
  }
}
```

---

## 🔧 Utils

### Async Handler

```typescript
// utils/asyncHandler.ts
import { Request, Response, NextFunction } from 'express';

export const asyncHandler = (
  fn: (req: Request, res: Response, next: NextFunction) => Promise<any>
) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// Usage
router.get('/', asyncHandler((req, res) => usersController.getAll(req, res)));
```

### App Error

```typescript
// utils/AppError.ts
export class AppError extends Error {
  constructor(
    public status: number,
    public message: string,
    public isOperational = true,
  ) {
    super(message);
    Object.setPrototypeOf(this, new.target.prototype);
    Error.captureStackTrace(this);
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Not Found') {
    super(404, message);
  }
}

export class BadRequestError extends AppError {
  constructor(message = 'Bad Request') {
    super(400, message);
  }
}

export class ConflictError extends AppError {
  constructor(message = 'Conflict') {
    super(409, message);
  }
}
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Middleware-Core]] — Middleware
- [[Express-Testing]] — Тестирование
