---
created: 2026-02-17
tags:
  - express
  - error-handling
  - errors
  - middleware
aliases:
  - Express Error Handling
  - Express Error Middleware
related:
  - Express-MOC
  - Express-Middleware-Core
---

# Express.js — Error Handling

> [!SUMMARY] Обзор
> Обработка ошибок в Express: error middleware, custom errors, async handlers.

---

## 🚨 Error Middleware

```typescript
// middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';

export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  console.error(err.stack);

  const status = (err as any).status || 500;
  const message = err.message || 'Internal Server Error';

  res.status(status).json({
    error: {
      message,
      status,
      stack: process.env.NODE_ENV === 'development' ? err.stack : undefined,
    },
  });
};

// Usage (must be last!)
app.use(errorHandler);
```

---

## 📋 Custom Error Classes

```typescript
// utils/AppError.ts
export class AppError extends Error {
  public status: number;
  public isOperational: boolean;

  constructor(
    status: number,
    message: string,
    isOperational = true,
  ) {
    super(message);
    this.status = status;
    this.isOperational = isOperational;
    
    Object.setPrototypeOf(this, new.target.prototype);
    Error.captureStackTrace(this);
  }
}

export class BadRequestError extends AppError {
  constructor(message = 'Bad Request') {
    super(400, message);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(403, message);
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Not Found') {
    super(404, message);
  }
}

export class ConflictError extends AppError {
  constructor(message = 'Conflict') {
    super(409, message);
  }
}

export class TooManyRequestsError extends AppError {
  constructor(message = 'Too Many Requests') {
    super(429, message);
  }
}

export class InternalError extends AppError {
  constructor(message = 'Internal Server Error') {
    super(500, message, false);
  }
}
```

---

## 🔄 Async Handler Wrapper

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
router.get(
  '/users',
  asyncHandler(async (req, res) => {
    const users = await User.findAll();
    res.json({ data: users });
  })
);

router.post(
  '/users',
  asyncHandler(async (req, res) => {
    const user = await User.create(req.body);
    res.status(201).json({ data: user });
  })
);
```

---

## 📝 Error Handling Examples

### Controller Example

```typescript
// controllers/users.controller.ts
import { Request, Response } from 'express';
import { asyncHandler } from '../utils/asyncHandler';
import { NotFoundError, BadRequestError, ConflictError } from '../utils/AppError';

export const usersController = {
  getAll: asyncHandler(async (req: Request, res: Response) => {
    const users = await UserService.findAll();
    res.json({ data: users });
  }),

  getOne: asyncHandler(async (req: Request, res: Response) => {
    const { id } = req.params;
    
    const user = await UserService.findById(Number(id));
    
    if (!user) {
      throw new NotFoundError('User not found');
    }
    
    res.json({ data: user });
  }),

  create: asyncHandler(async (req: Request, res: Response) => {
    const { email } = req.body;
    
    // Check for existing email
    const existing = await UserService.findByEmail(email);
    if (existing) {
      throw new ConflictError('Email already exists');
    }
    
    // Validate
    if (!email) {
      throw new BadRequestError('Email is required');
    }
    
    const user = await UserService.create(req.body);
    res.status(201).json({ data: user });
  }),

  update: asyncHandler(async (req: Request, res: Response) => {
    const { id } = req.params;
    
    const user = await UserService.update(Number(id), req.body);
    
    if (!user) {
      throw new NotFoundError('User not found');
    }
    
    res.json({ data: user });
  }),

  delete: asyncHandler(async (req: Request, res: Response) => {
    const { id } = req.params;
    
    const deleted = await UserService.delete(Number(id));
    
    if (!deleted) {
      throw new NotFoundError('User not found');
    }
    
    res.status(204).send();
  }),
};
```

### Service Example

```typescript
// services/users.service.ts
import { NotFoundError, ConflictError } from '../utils/AppError';

export class UserService {
  async findById(id: number) {
    const user = await prisma.user.findUnique({ where: { id } });
    
    if (!user) {
      throw new NotFoundError('User not found');
    }
    
    return user;
  }

  async findByEmail(email: string) {
    return prisma.user.findUnique({ where: { email } });
  }

  async create(data: CreateUserDto) {
    const existing = await this.findByEmail(data.email);
    
    if (existing) {
      throw new ConflictError('Email already exists');
    }
    
    return prisma.user.create({ data });
  }

  async update(id: number, data: UpdateUserDto) {
    const user = await this.findById(id);
    
    if (data.email && data.email !== user.email) {
      const existing = await this.findByEmail(data.email);
      if (existing) {
        throw new ConflictError('Email already exists');
      }
    }
    
    return prisma.user.update({ where: { id }, data });
  }

  async delete(id: number) {
    await this.findById(id);
    await prisma.user.delete({ where: { id } });
    return true;
  }
}
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Middleware-Core]] — Middleware
- [[Express-Validation]] — Validation
