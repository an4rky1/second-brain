---
created: 2026-02-17
tags:
  - express
  - middleware
  - core
  - custom
aliases:
  - Express Middleware Core
  - Express Custom Middleware
related:
  - Express-MOC
  - Express-Request-Response
---

# Express.js — Middleware

> [!SUMMARY] Обзор
> Middleware в Express: built-in, custom, third-party, error handling.

---

## 🔌 Built-in Middleware

```typescript
import express from 'express';

const app = express();

// Parse JSON bodies
app.use(express.json({ limit: '10mb' }));

// Parse URL-encoded bodies
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Serve static files
app.use(express.static('public', {
  maxAge: '1d',
  etag: true,
  lastModified: true,
}));

// Text parser
app.use(express.text());

// Raw buffer parser
app.use(express.raw());
```

---

## 📝 Custom Middleware

### Logger Middleware

```typescript
// middleware/logger.middleware.ts
import { Request, Response, NextFunction } from 'express';

export const loggerMiddleware = (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const start = Date.now();
  
  console.log(`${req.method} ${req.url}`);
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} ${res.statusCode} ${duration}ms`);
  });
  
  next();
};

// Usage
app.use(loggerMiddleware);
```

### Auth Middleware

```typescript
// middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

export interface AuthRequest extends Request {
  user?: { id: number; email: string; role: string };
}

export const authMiddleware = (
  req: AuthRequest,
  res: Response,
  next: NextFunction,
) => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  const token = authHeader.split(' ')[1];
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    req.user = decoded as any;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Role-based middleware
export const authorize = (...roles: string[]) => {
  return (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    next();
  };
};

// Usage
app.use('/api', authMiddleware);
app.delete('/api/users/:id', authorize('admin'), deleteUser);
```

### Validation Middleware

```typescript
// middleware/validation.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export const validate = (schema: AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      await schema.parseAsync(req.body);
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors.map(err => ({
            field: err.path.join('.'),
            message: err.message,
          })),
        });
      }
      next(error);
    }
  };
};

// Usage
app.post('/users', validate(registerSchema), createUser);
```

---

## 🚨 Error Handling Middleware

```typescript
// middleware/errorHandler.middleware.ts
import { Request, Response, NextFunction } from 'express';

export class AppError extends Error {
  constructor(
    public status: number,
    public message: string,
  ) {
    super(message);
  }
}

export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  console.error(err.stack);

  if (err instanceof AppError) {
    return res.status(err.status).json({
      error: {
        message: err.message,
        status: err.status,
      },
    });
  }

  // Unknown error
  res.status(500).json({
    error: {
      message: process.env.NODE_ENV === 'development' 
        ? err.message 
        : 'Internal Server Error',
      status: 500,
    },
  });
};

// Usage (must be last!)
app.use(errorHandler);
```

---

## 📦 Third-party Middleware

```bash
npm install cors helmet morgan compression rate-limiter-flexible
npm install -D @types/cors @types/morgan
```

```typescript
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import rateLimit from 'express-rate-limit';

// CORS
app.use(cors({
  origin: ['https://example.com'],
  credentials: true,
}));

// Security headers
app.use(helmet());

// HTTP logger
app.use(morgan('combined'));

// Compression
app.use(compression());

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests',
});
app.use('/api/', limiter);
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Error-Handling]] — Error handlers
- [[Express-Validation]] — Validation middleware
