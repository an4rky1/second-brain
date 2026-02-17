---
created: 2026-02-17
tags:
  - express
  - logging
  - morgan
  - winston
aliases:
  - Express Logging
  - Express Morgan Winston
related:
  - Express-MOC
  - Express-Middleware-Core
---

# Express.js — Logging

> [!SUMMARY] Обзор
> Логирование в Express: Morgan (HTTP logs), Winston (application logs).

---

## Morgan (HTTP Logger)

### Установка

```bash
npm install morgan
npm install -D @types/morgan
```

### Basic Usage

```typescript
// middleware/logger.middleware.ts
import morgan from 'morgan';
import { logger } from '../utils/logger';

// Custom stream for Winston
const stream = {
  write: (message: string) => {
    logger.info(message.trim());
  },
};

// Development format (colored)
export const morganDev = morgan('dev', { stream });

// Combined format (production)
export const morganCombined = morgan('combined', { stream });

// Custom format
export const morganCustom = morgan(
  ':method :url :status :res[content-length] - :response-time ms',
  { stream }
);

// Custom tokens
morgan.token('user-id', (req) => (req as any).user?.id || 'anonymous');
morgan.token('response-time', (req, res) => {
  return String(Date.now() - (req as any).startTime);
});

export const morganCustomWithUser = morgan(
  ':method :url :status :res[content-length] - :response-time ms - user::user-id',
  { stream }
);

// Middleware to track start time
export const trackStartTime = (req: any, res: any, next: any) => {
  req.startTime = Date.now();
  next();
};

// Usage
app.use(trackStartTime);
app.use(morganCustomWithUser);
```

### Formats

```typescript
// Available formats
morgan('tiny');     // GET /users 200 - 5ms
morgan('short');    // GET /users 200 - 5ms
morgan('common');   // Combined log format
morgan('combined'); // Apache combined log format
morgan('dev');      // Colored output for dev

// Custom format
morgan(':remote-addr - :remote-user [:date[clf]] ":method :url HTTP/:http-version" :status :res[content-length] ":referrer" ":user-agent"');
```

---

## Winston (Application Logger)

### Установка

```bash
npm install winston
npm install -D @types/winston
```

### Logger Config

```typescript
// utils/logger.ts
import winston from 'winston';
import path from 'path';

const { combine, timestamp, printf, colorize, errors } = winston.format;

// Custom format
const logFormat = printf(({ level, message, timestamp, stack, ...metadata }) => {
  let msg = `${timestamp} [${level}]: ${message}`;
  
  if (Object.keys(metadata).length > 0) {
    msg += ` ${JSON.stringify(metadata)}`;
  }
  
  if (stack) {
    msg += `\nStack: ${stack}`;
  }
  
  return msg;
});

// Logger instance
export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(
    errors({ stack: true }),
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    logFormat
  ),
  defaultMeta: { service: 'express-app' },
  transports: [
    // Console
    new winston.transports.Console({
      format: combine(colorize(), logFormat),
    }),
    
    // Error log
    new winston.transports.File({
      filename: path.join('logs', 'error.log'),
      level: 'error',
      maxsize: 5242880, // 5MB
      maxFiles: 5,
    }),
    
    // Combined log
    new winston.transports.File({
      filename: path.join('logs', 'combined.log'),
      maxsize: 5242880,
      maxFiles: 5,
    }),
  ],
  
  exceptionHandlers: [
    new winston.transports.File({
      filename: path.join('logs', 'exceptions.log'),
    }),
  ],
  
  rejectionHandlers: [
    new winston.transports.File({
      filename: path.join('logs', 'rejections.log'),
    }),
  ],
});
```

### Usage in Controllers

```typescript
// controllers/users.controller.ts
import { Request, Response } from 'express';
import { logger } from '../utils/logger';

export const usersController = {
  getAll: async (req: Request, res: Response) => {
    const start = Date.now();
    
    try {
      logger.info('Fetching all users', { 
        method: req.method, 
        url: req.url,
        userId: (req as any).user?.id,
      });
      
      const users = await UserService.findAll();
      
      const duration = Date.now() - start;
      logger.info('Users fetched successfully', { 
        count: users.length, 
        duration: `${duration}ms`,
      });
      
      res.json({ data: users });
    } catch (error: any) {
      logger.error('Failed to fetch users', { 
        error: error.message, 
        stack: error.stack,
      });
      
      res.status(500).json({ error: 'Failed to fetch users' });
    }
  },

  create: async (req: Request, res: Response) => {
    try {
      logger.info('Creating new user', { data: req.body });
      
      const user = await UserService.create(req.body);
      
      logger.info('User created', { userId: user.id });
      
      res.status(201).json({ data: user });
    } catch (error: any) {
      logger.error('Failed to create user', { 
        error: error.message,
        data: req.body,
      });
      
      res.status(500).json({ error: 'Failed to create user' });
    }
  },
};
```

---

## Request Logging Middleware

```typescript
// middleware/requestLogger.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { logger } from '../utils/logger';

export const requestLogger = (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    logger.info('Request completed', {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      ip: req.ip,
      userAgent: req.get('user-agent'),
      userId: (req as any).user?.id,
    });
  });
  
  next();
};

// Usage
app.use(requestLogger);
```

---

## Environment

```bash
# .env
LOG_LEVEL=info
LOG_FORMAT=json
LOG_FILE_PATH=./logs
LOG_MAX_SIZE=5242880
LOG_MAX_FILES=5
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Middleware-Core]] — Middleware
- [[Express-Error-Handling]] — Error handling
