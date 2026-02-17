---
created: 2026-02-17
tags:
  - nestjs
  - interceptors
  - cache
  - transform
  - logging
aliases:
  - NestJS Interceptors
  - NestJS Cache Interceptor
related:
  - NestJS-MOC
  - NestJS-Guards
  - NestJS-Filters
---

# NestJS — Interceptors

> [!SUMMARY] Обзор
> Interceptors изменяют request/response перед/после обработки. Transform, cache, logging, timeout.

---

## 🔄 Basic Interceptor

```typescript
// interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface ApiResponse<T> {
  data: T;
  meta: {
    timestamp: string;
    path: string;
    method: string;
  };
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<ApiResponse<T>> {
    const request = context.switchToHttp().getRequest();

    return next.handle().pipe(
      map(data => ({
        data,
        meta: {
          timestamp: new Date().toISOString(),
          path: request.url,
          method: request.method,
        },
      })),
    );
  }
}

// Usage
@UseInterceptors(TransformInterceptor)
@Get('users')
findAll() {
  return this.usersService.findAll();
}
```

---

## 📝 Logging Interceptor

```typescript
// interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url, ip } = request;
    const startTime = Date.now();

    return next
      .handle()
      .pipe(
        tap(() => {
          const response = context.switchToHttp().getResponse();
          const { statusCode } = response;
          const duration = Date.now() - startTime;

          this.logger.log(
            `${method} ${url} ${statusCode} ${duration}ms [${ip}]`,
          );
        }),
      );
  }
}

// Usage
@UseInterceptors(LoggingInterceptor)
@Get('users')
findAll() {
  return this.usersService.findAll();
}
```

---

## 💾 Cache Interceptor

```typescript
// interceptors/cache.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(private redisService: RedisService) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = `cache:${request.url}`;

    // Check cache
    const cached = await this.redisService.get(cacheKey);
    if (cached) {
      return of(cached);
    }

    // Execute and cache
    return next.handle().pipe(
      tap(data => {
        this.redisService.set(cacheKey, data, 300); // 5 minutes
      }),
    );
  }
}

// Usage with built-in CacheInterceptor
import { CacheInterceptor, CacheKey, CacheTTL } from '@nestjs/common';

@UseInterceptors(CacheInterceptor)
@Get('users')
@CacheKey('users_all')
@CacheTTL(300)  // 5 minutes
findAll() {
  return this.usersService.findAll();
}
```

---

## ⏱️ Timeout Interceptor

```typescript
// interceptors/timeout.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { timeout, catchError } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<any> {
    return next.handle().pipe(
      timeout(5000), // 5 second timeout
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}

// Usage
@UseInterceptors(TimeoutInterceptor)
@Get('slow-endpoint')
async slowEndpoint() {
  // Will timeout after 5 seconds
  return this.slowService.getData();
}
```

---

## 📊 Response Class Serializer Interceptor

```typescript
// interceptors/class-serializer.interceptor.ts
import { UseInterceptors } from '@nestjs/common';
import { ClassSerializerInterceptor } from '@nestjs/common';

// DTO with exclusion
export class UserResponseDto {
  id: number;
  email: string;
  name: string;
  
  @Exclude()  // Exclude from response
  password: string;

  @Exclude()
  createdAt: Date;

  @Expose()  // Explicitly include
  @Transform(({ value }) => value?.toUpperCase())
  role: string;
}

// Usage
@UseInterceptors(ClassSerializerInterceptor)
@Get('users')
findAll(): UserResponseDto[] {
  return this.usersService.findAll();
}

// Or at controller level
@Controller('users')
@UseInterceptors(ClassSerializerInterceptor)
export class UsersController {}
```

---

## 🌍 Global Interceptors

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { TransformInterceptor } from './interceptors/transform.interceptor';
import { LoggingInterceptor } from './interceptors/logging.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global interceptors
  app.useGlobalInterceptors(new TransformInterceptor());
  app.useGlobalInterceptors(new LoggingInterceptor());

  await app.listen(3000);
}
bootstrap();
```

```typescript
// app.module.ts
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: TransformInterceptor,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

---

## 🎯 File Interceptor

```typescript
// upload/upload.controller.ts
import {
  Controller,
  Post,
  Body,
  UseInterceptors,
  UploadedFile,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname } from 'path';

@Controller('upload')
export class UploadController {
  @Post('file')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './uploads',
        filename: (req, file, cb) => {
          const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
          const ext = extname(file.originalname);
          cb(null, `${file.fieldname}-${uniqueSuffix}${ext}`);
        },
      }),
      fileFilter: (req, file, cb) => {
        if (!file.mimetype.match(/\/(jpg|jpeg|png|gif)$/)) {
          return cb(new Error('Only image files are allowed!'), false);
        }
        cb(null, true);
      },
      limits: {
        fileSize: 5 * 1024 * 1024, // 5MB
      },
    }),
  )
  uploadFile(@UploadedFile() file: Express.Multer.File) {
    return {
      message: 'File uploaded successfully',
      filename: file.filename,
      size: file.size,
    };
  }
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Guards]] — Guards
- [[NestJS-Filters]] — Filters
- [[NestJS-Pipes]] — Pipes
