---
created: 2026-02-17
tags:
  - nestjs
  - sessions
  - express-session
  - redis
aliases:
  - NestJS Sessions
  - NestJS Session Store
related:
  - NestJS-MOC
  - NestJS-JWT-Auth
  - NestJS-Passport
---

# NestJS — Sessions

> [!SUMMARY] Обзор
> Sessions в NestJS: express-session, Redis store, cookie configuration.

---

## 📦 Установка

```bash
npm install express-session connect-redis redis
npm install -D @types/express-session @types/connect-redis
```

---

## 🔧 Session Module

```typescript
// session/session.module.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class SessionMiddleware implements NestMiddleware {
  private sessionMiddleware: ReturnType<typeof session>;

  constructor(private configService: ConfigService) {
    // Redis client
    const redisClient = createClient({
      url: this.configService.get('REDIS_URL') || 'redis://localhost:6379',
    });
    redisClient.connect();

    // Session middleware
    this.sessionMiddleware = session({
      store: new RedisStore({ client: redisClient }),
      secret: this.configService.get('SESSION_SECRET'),
      resave: false,
      saveUninitialized: false,
      name: 'sessionId',
      cookie: {
        secure: this.configService.get('NODE_ENV') === 'production',
        httpOnly: true,
        maxAge: 24 * 60 * 60 * 1000, // 1 day
        sameSite: 'lax',
        path: '/',
      },
    });
  }

  use(req: Request, res: Response, next: NextFunction) {
    this.sessionMiddleware(req, res, next);
  }
}

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

## 🏛️ App Module Setup

```typescript
// app.module.ts
import { Module, MiddlewareConsumer, RequestMethod } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { SessionMiddleware } from './session/session.middleware';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
    AuthModule,
    UsersModule,
  ],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(SessionMiddleware)
      .forRoutes({
        path: '*',
        method: RequestMethod.ALL,
      });
  }
}
```

---

## 🔐 Session Auth Controller

```typescript
// auth/auth.controller.ts
import {
  Controller,
  Post,
  Get,
  Body,
  UseGuards,
  Request,
  Response,
  Session,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { SessionRequest } from '../common/interfaces/session-request.interface';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  async register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }

  @UseGuards(AuthGuard('local'))
  @Post('login')
  async login(
    @Request() req: SessionRequest,
    @Session() session: Record<string, any>,
  ) {
    // Set session
    session.userId = req.user.id;
    session.email = req.user.email;
    session.role = req.user.role;

    return {
      message: 'Login successful',
      user: {
        id: req.user.id,
        email: req.user.email,
        name: req.user.name,
      },
    };
  }

  @Post('logout')
  async logout(@Session() session: Record<string, any>) {
    return new Promise((resolve, reject) => {
      session.destroy((err) => {
        if (err) {
          reject(err);
        } else {
          resolve({ message: 'Logout successful' });
        }
      });
    });
  }

  @Get('profile')
  @UseGuards(AuthGuard('jwt'))  // Or custom session guard
  async getProfile(@Request() req: SessionRequest) {
    return { user: req.user };
  }

  @Get('me')
  async getMe(@Session() session: Record<string, any>) {
    if (!session.userId) {
      return { authenticated: false };
    }

    const user = await this.authService.findUserById(session.userId);
    return { authenticated: true, user };
  }
}
```

---

## 🛡️ Session Guard

```typescript
// auth/guards/session-auth.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';

@Injectable()
export class SessionAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const session = request.session;

    if (!session || !session.userId) {
      throw new UnauthorizedException('Session not found');
    }

    // Attach user to request
    request.user = {
      id: session.userId,
      email: session.email,
      role: session.role,
    };

    return true;
  }
}

// Usage
@Controller('profile')
@UseGuards(SessionAuthGuard)
export class ProfileController {
  @Get()
  getProfile(@Request() req: any) {
    return { user: req.user };
  }
}
```

---

## 🔄 Session Service

```typescript
// session/session.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class SessionService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  // Get session data
  async getSession(sessionId: string): Promise<any> {
    return this.cacheManager.get(`session:${sessionId}`);
  }

  // Set session data
  async setSession(sessionId: string, data: any, ttl?: number): Promise<void> {
    await this.cacheManager.set(`session:${sessionId}`, data, ttl);
  }

  // Destroy session
  async destroySession(sessionId: string): Promise<void> {
    await this.cacheManager.del(`session:${sessionId}`);
  }

  // Update session
  async updateSession(sessionId: string, data: Partial<any>): Promise<void> {
    const session = await this.getSession(sessionId);
    await this.setSession(sessionId, { ...session, ...data });
  }

  // Get user from session
  async getUserFromSession(sessionId: string): Promise<any> {
    const session = await this.getSession(sessionId);
    return session?.user;
  }
}
```

---

## 📋 Environment

```bash
# .env
SESSION_SECRET=your-super-secret-session-key
REDIS_URL=redis://localhost:6379
NODE_ENV=production  # Enables secure cookies
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-JWT-Auth]] — JWT Authentication
- [[NestJS-Passport]] — Passport.js
- [[NestJS-Redis]] — Redis integration
