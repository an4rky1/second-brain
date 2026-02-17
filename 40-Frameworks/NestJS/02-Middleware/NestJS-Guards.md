---
created: 2026-02-17
tags:
  - nestjs
  - guards
  - auth
  - authorization
aliases:
  - NestJS Guards
  - NestJS Authentication
related:
  - NestJS-MOC
  - NestJS-Passport
  - NestJS-Interceptors
---

# NestJS — Guards

> [!SUMMARY] Обзор
> Guards определяют может ли запрос быть обработан контроллером. Authentication, authorization, roles.

---

## 🛡️ Basic Auth Guard

```typescript
// auth/guards/jwt-auth.guard.ts
import {
  Injectable,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';
import { IS_PUBLIC_KEY } from '../../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Проверка на @Public() декоратор
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    return super.canActivate(context);
  }

  handleRequest(err: any, user: any, info: any) {
    if (err || !user) {
      throw err || new UnauthorizedException('Invalid token');
    }
    return user;
  }
}
```

---

## 👑 Roles Guard

```typescript
// auth/guards/roles.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();

    if (!user) {
      throw new ForbiddenException('User not found');
    }

    const hasRole = requiredRoles.some(role => user.roles?.includes(role));

    if (!hasRole) {
      throw new ForbiddenException(
        `Required roles: ${requiredRoles.join(', ')}. User has: ${user.roles?.join(', ')}`,
      );
    }

    return true;
  }
}
```

---

## 🏷️ Custom Decorators

### Public Decorator

```typescript
// decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// Usage
@Public()
@Post('login')
login() { ... }

@Public()
@Get('health')
health() { ... }
```

### Roles Decorator

```typescript
// decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// Usage
@Roles('admin')
@Get('admin/stats')
getAdminStats() { ... }

@Roles('admin', 'moderator')
@Post('posts/moderate')
moderatePost() { ... }
```

### User Decorator

```typescript
// decorators/user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);

// Usage
@Get('profile')
@UseGuards(JwtAuthGuard)
getProfile(@User() user: UserEntity) {
  return user;
}

@Get('profile/email')
@UseGuards(JwtAuthGuard)
getEmail(@User('email') email: string) {
  return email;
}
```

---

## 🔐 Usage Examples

### Controller Level

```typescript
// users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  UseGuards,
} from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles } from '../decorators/roles.decorator';
import { Public } from '../decorators/public.decorator';
import { UsersService } from './users.service';

@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)  // Apply to all routes
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // ─────────────────────────────────────────────────────
  // PUBLIC ENDPOINTS
  // ─────────────────────────────────────────────────────
  @Public()
  @Post('register')
  register(@Body() registerDto: RegisterDto) {
    return this.usersService.register(registerDto);
  }

  @Public()
  @Post('login')
  login(@Body() loginDto: LoginDto) {
    return this.usersService.login(loginDto);
  }

  // ─────────────────────────────────────────────────────
  // AUTHENTICATED ENDPOINTS
  // ─────────────────────────────────────────────────────
  @Get('profile')
  getProfile(@User() user: UserEntity) {
    return user;
  }

  @Get()
  findAll(@Query() query: QueryDto) {
    return this.usersService.findAll(query);
  }

  // ─────────────────────────────────────────────────────
  // ADMIN ONLY ENDPOINTS
  // ─────────────────────────────────────────────────────
  @Roles('admin')
  @Get('admin/stats')
  getAdminStats() {
    return this.usersService.getStats();
  }

  @Roles('admin')
  @Delete(':id')
  remove(@Param('id') id: number) {
    return this.usersService.remove(id);
  }

  // ─────────────────────────────────────────────────────
  // ADMIN OR MODERATOR ENDPOINTS
  // ─────────────────────────────────────────────────────
  @Roles('admin', 'moderator')
  @Post(':id/ban')
  banUser(@Param('id') id: number) {
    return this.usersService.ban(id);
  }
}
```

### Method Level

```typescript
@Controller('posts')
export class PostsController {
  @Get()
  @UseGuards(JwtAuthGuard)  // Only for this route
  findAll() {
    return this.postsService.findAll();
  }

  @Post()
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Roles('admin', 'moderator')
  create(@Body() createPostDto: CreatePostDto) {
    return this.postsService.create(createPostDto);
  }
}
```

### Global Guards

```typescript
// main.ts
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,  // Global auth guard
    },
    {
      provide: APP_GUARD,
      useClass: RolesGuard,    // Global roles guard
    },
  ],
})
export class AppModule {}
```

---

## 🎯 Throttler Guard (Rate Limiting)

```bash
npm install @nestjs/throttler
```

```typescript
// app.module.ts
import { ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,      // 1 minute
      limit: 10,       // 10 requests per minute
    }]),
  ],
})
export class AppModule {}

// Controller
@Controller('api')
@UseGuards(ThrottlerGuard)
export class ApiController {
  @Get('heavy')
  @SkipThrottle()  // Skip rate limiting for this route
  heavyOperation() {
    return { message: 'Done' };
  }
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Passport]] — Passport.js
- [[NestJS-Interceptors]] — Interceptors
- [[NestJS-JWT-Auth]] — JWT Authentication
