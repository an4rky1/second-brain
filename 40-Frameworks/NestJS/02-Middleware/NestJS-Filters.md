---
created: 2026-02-17
tags:
  - nestjs
  - filters
  - exception
  - error-handling
aliases:
  - NestJS Filters
  - NestJS Exception Filters
related:
  - NestJS-MOC
  - NestJS-Interceptors
  - NestJS-Guards
---

# NestJS — Filters & Error Handling

> [!SUMMARY] Обзор
> Exception filters обрабатывают ошибки централизованно. Custom filters, error responses, logging.

---

## 🚨 Basic Exception Filter

```typescript
// filters/all-exceptions.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()  // Catch all exceptions
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger('Exceptions');

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    // Determine status
    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    // Determine message
    const message =
      exception instanceof HttpException
        ? exception.getMessage()
        : 'Internal server error';

    // Log error
    if (status >= 500) {
      this.logger.error(
        `${request.method} ${request.url}`,
        exception instanceof Error ? exception.stack : '',
      );
    }

    // Response to client
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message,
    });
  }
}
```

---

## 📋 HTTP Exception Filter

```typescript
// filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    // Format response
    const errorResponse = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      error: typeof exceptionResponse === 'string' 
        ? exceptionResponse 
        : (exceptionResponse as any).message,
      details: typeof exceptionResponse === 'object' 
        ? (exceptionResponse as any).errors 
        : undefined,
    };

    response.status(status).json(errorResponse);
  }
}
```

---

## 🛠️ Custom Exception Filter

```typescript
// filters/validation-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  BadRequestException,
} from '@nestjs/common';
import { Response } from 'express';

@Catch(BadRequestException)
export class ValidationExceptionFilter implements ExceptionFilter {
  catch(exception: BadRequestException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest();

    const exceptionResponse = exception.getResponse() as any;

    response.status(400).json({
      statusCode: 400,
      timestamp: new Date().toISOString(),
      path: request.url,
      error: 'Validation failed',
      details: exceptionResponse.message || exceptionResponse,
    });
  }
}
```

---

## 🏛️ Custom Error Classes

```typescript
// exceptions/app.exception.ts
export class AppException extends Error {
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

export class BadRequestException extends AppException {
  constructor(message = 'Bad Request') {
    super(400, message);
  }
}

export class UnauthorizedException extends AppException {
  constructor(message = 'Unauthorized') {
    super(401, message);
  }
}

export class ForbiddenException extends AppException {
  constructor(message = 'Forbidden') {
    super(403, message);
  }
}

export class NotFoundException extends AppException {
  constructor(message = 'Not Found') {
    super(404, message);
  }
}

export class ConflictException extends AppException {
  constructor(message = 'Conflict') {
    super(409, message);
  }
}

export class TooManyRequestsException extends AppException {
  constructor(message = 'Too Many Requests') {
    super(429, message);
  }
}

export class InternalServerException extends AppException {
  constructor(message = 'Internal Server Error') {
    super(500, message, false);
  }
}
```

---

## 🎯 Usage in Services

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import {
  NotFoundException,
  ConflictException,
  BadRequestException,
} from '../exceptions/app.exception';

@Injectable()
export class UsersService {
  async findOne(id: number) {
    const user = await this.prisma.user.findUnique({ where: { id } });
    
    if (!user) {
      throw new NotFoundException('User not found');
    }
    
    return user;
  }

  async create(createUserDto: CreateUserDto) {
    const existing = await this.prisma.user.findUnique({
      where: { email: createUserDto.email },
    });

    if (existing) {
      throw new ConflictException('Email already exists');
    }

    if (createUserDto.password.length < 8) {
      throw new BadRequestException('Password must be at least 8 characters');
    }

    return this.prisma.user.create({ data: createUserDto });
  }

  async update(id: number, updateUserDto: UpdateUserDto) {
    await this.findOne(id);  // Throws NotFoundException if not found

    if (updateUserDto.email) {
      const existing = await this.prisma.user.findUnique({
        where: { email: updateUserDto.email },
      });

      if (existing && existing.id !== id) {
        throw new ConflictException('Email already exists');
      }
    }

    return this.prisma.user.update({
      where: { id },
      data: updateUserDto,
    });
  }
}
```

---

## 🌍 Global Filters

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AllExceptionsFilter } from './filters/all-exceptions.filter';
import { HttpExceptionFilter } from './filters/http-exception.filter';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global exception filters
  app.useGlobalFilters(new AllExceptionsFilter());
  app.useGlobalFilters(new HttpExceptionFilter());

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

```typescript
// app.module.ts
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: AllExceptionsFilter,
    },
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

---

## 📝 Error Response Format

```typescript
// Standard error response
{
  "statusCode": 400,
  "timestamp": "2026-02-17T12:00:00.000Z",
  "path": "/api/users",
  "method": "POST",
  "error": "Validation failed",
  "details": [
    {
      "field": "email",
      "message": "Invalid email format"
    },
    {
      "field": "password",
      "message": "Password must be at least 8 characters"
    }
  ]
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Interceptors]] — Interceptors
- [[NestJS-Guards]] — Guards
- [[NestJS-DTO-Validation]] — DTO & Validation
