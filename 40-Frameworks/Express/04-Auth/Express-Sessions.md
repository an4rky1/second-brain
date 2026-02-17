---
created: 2026-02-17
tags:
  - express
  - sessions
  - express-session
  - redis
aliases:
  - Express Sessions
  - Express Session Store
related:
  - Express-MOC
  - Express-Passport
---

# Express.js — Sessions

> [!SUMMARY] Обзор
> Sessions в Express: express-session, Redis store, cookie configuration.

---

## Установка

```bash
npm install express-session connect-redis redis
npm install -D @types/express-session @types/connect-redis
```

---

## Session Config

```typescript
// middleware/session.middleware.ts
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';

// Redis client
const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});
redisClient.connect();

// Session middleware
export const sessionMiddleware = session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET!,
  resave: false,          // Don't save session if unmodified
  saveUninitialized: false, // Don't create session until something stored
  name: 'sessionId',      // Custom cookie name
  cookie: {
    secure: process.env.NODE_ENV === 'production', // HTTPS only in production
    httpOnly: true,       // Prevent XSS
    maxAge: 24 * 60 * 60 * 1000, // 1 day
    sameSite: 'lax',      // CSRF protection
    path: '/',
  },
});

// Session type augmentation
declare module 'express-session' {
  interface SessionData {
    userId: number;
    email: string;
    role: string;
  }
}
```

---

## Session Auth

```typescript
// middleware/sessionAuth.middleware.ts
import { Request, Response, NextFunction } from 'express';

export interface SessionRequest extends Request {
  session: any;
}

export const sessionAuth = (
  req: SessionRequest,
  res: Response,
  next: NextFunction,
) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  req.user = {
    id: req.session.userId,
    email: req.session.email,
    role: req.session.role,
  };
  
  next();
};
```

---

## Session Controller

```typescript
// controllers/sessionAuth.controller.ts
import { Response } from 'express';
import { SessionRequest } from '../middleware/sessionAuth.middleware';
import { AuthService } from '../services/auth.service';

export class SessionAuthController {
  constructor(private authService: AuthService) {}

  // Login
  async login(req: SessionRequest, res: Response) {
    const { email, password } = req.body;

    const user = await this.authService.validateUser(email, password);
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Set session
    req.session.userId = user.id;
    req.session.email = user.email;
    req.session.role = user.role;

    res.json({
      message: 'Login successful',
      data: { user },
    });
  }

  // Logout
  async logout(req: SessionRequest, res: Response) {
    req.session.destroy((err) => {
      if (err) {
        return res.status(500).json({ error: 'Logout failed' });
      }
      res.json({ message: 'Logout successful' });
    });
  }

  // Get current user
  async getMe(req: SessionRequest, res: Response) {
    const user = await this.authService.findUserById(req.session.userId);
    res.json({ data: user });
  }

  // Update session (e.g., role change)
  async updateSession(req: SessionRequest, res: Response) {
    const { role } = req.body;
    req.session.role = role;
    res.json({ message: 'Session updated' });
  }
}
```

---

## Session Routes

```typescript
// routes/session.routes.ts
import { Router } from 'express';
import { SessionAuthController } from '../controllers/sessionAuth.controller';
import { sessionAuth } from '../middleware/sessionAuth.middleware';

const router = Router();
const controller = new SessionAuthController();

router.post('/login', (req, res) => controller.login(req, res));
router.post('/logout', sessionAuth, (req, res) => controller.logout(req, res));
router.get('/me', sessionAuth, (req, res) => controller.getMe(req, res));

export default router;
```

---

## Memory Store (Development Only)

```typescript
// ⚠️ Don't use in production!
import session from 'express-session';

app.use(
  session({
    secret: 'your-secret-key',
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: false, // Set to true in production with HTTPS
      maxAge: 24 * 60 * 60 * 1000,
    },
  })
);
```

---

## Environment

```bash
# .env
SESSION_SECRET=your-super-secret-session-key
REDIS_URL=redis://localhost:6379
NODE_ENV=production  # Enables secure cookies
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Passport]] — Passport.js
- [[Express-JWT-Auth]] — JWT authentication
