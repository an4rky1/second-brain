---
created: 2026-02-17
tags:
  - nestjs
  - pipes
  - validation
  - transformation
aliases:
  - NestJS Pipes
  - NestJS Validation Pipe
related:
  - NestJS-MOC
  - NestJS-DTO-Validation
  - NestJS-Filters
---

# NestJS — Pipes

> [!SUMMARY] Обзор
> Pipes обрабатывают данные перед передачей контроллеру. Validation, transformation, custom pipes.

---

## 📮 Built-in Pipes

### ValidationPipe

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // Remove extra properties
      forbidNonWhitelisted: true,   // Throw error on extra properties
      transform: true,              // Transform to DTO class
      transformOptions: {
        enableImplicitConversion: true,
      },
      validationError: {
        target: false,              // Don't include target in error
        value: false,               // Don't include value in error
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

### ParseIntPipe

```typescript
// users/users.controller.ts
import { Controller, Get, Param, ParseIntPipe } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    // id is guaranteed to be a number
    return this.usersService.findOne(id);
  }
}
```

### ParseFloatPipe

```typescript
import { Controller, Post, Body, ParseFloatPipe } from '@nestjs/common';

@Controller('products')
export class ProductsController {
  @Post()
  create(@Body('price', ParseFloatPipe) price: number) {
    return this.productsService.create({ price });
  }
}
```

### ParseBoolPipe

```typescript
import { Controller, Get, Query, ParseBoolPipe } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get()
  findAll(@Query('active', ParseBoolPipe) active: boolean) {
    return this.usersService.findActive(active);
  }
}
```

### ParseEnumPipe

```typescript
import {
  Controller,
  Get,
  Query,
  ParseEnumPipe,
  ValidationPipeOptions,
} from '@nestjs/common';

enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
}

@Controller('users')
export class UsersController {
  @Get()
  findAll(
    @Query('role', new ParseEnumPipe(UserRole)) role: UserRole,
  ) {
    return this.usersService.findByRole(role);
  }
}
```

### ParseArrayPipe

```typescript
import { Controller, Get, Query, ParseArrayPipe } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get()
  findAll(
    @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
    ids: number[],
  ) {
    return this.usersService.findByIds(ids);
  }

  @Get('bulk')
  findBulk(
    @Query(
      'users',
      new ParseArrayPipe({
        items: CreateUserDto,
        whitelist: true,
        forbidNonWhitelisted: true,
      }),
    )
    users: CreateUserDto[],
  ) {
    return this.usersService.bulkCreate(users);
  }
}
```

### ParseUUIDPipe

```typescript
import { Controller, Get, Param, ParseUUIDPipe } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    // id is guaranteed to be a valid UUID
    return this.usersService.findOne(id);
  }

  @Get(':id')
  findOneCustom(
    @Param(
      'id',
      new ParseUUIDPipe({
        version: '4',  // Only UUID v4
        errorMsg: 'Invalid UUID v4',
      }),
    )
    id: string,
  ) {
    return this.usersService.findOne(id);
  }
}
```

---

## 🔧 Custom Pipes

### Custom Validation Pipe

```typescript
// pipes/validate-user.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class ValidateUserPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // Check age
    if (value.age && value.age < 18) {
      throw new BadRequestException('User must be at least 18 years old');
    }

    // Check email domain
    if (value.email && !value.email.endsWith('@company.com')) {
      throw new BadRequestException('Email must be @company.com domain');
    }

    return value;
  }
}

// Usage
@Controller('users')
export class UsersController {
  @Post()
  create(@Body(ValidateUserPipe) createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

### Custom ParseIntPipe

```typescript
// pipes/parse-positive-int.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class ParsePositiveIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);

    if (isNaN(val)) {
      throw new BadRequestException(`Validation failed: "${value}" is not a number`);
    }

    if (val <= 0) {
      throw new BadRequestException(`Validation failed: "${value}" must be positive`);
    }

    return val;
  }
}

// Usage
@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id', ParsePositiveIntPipe) id: number) {
    return this.usersService.findOne(id);
  }
}
```

### File Parse Pipe

```typescript
// pipes/parse-file.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class ParseFilePipe implements PipeTransform {
  constructor(
    private maxSize: number,
    private allowedTypes: string[],
  ) {}

  transform(value: any, metadata: ArgumentMetadata) {
    if (!value) {
      throw new BadRequestException('File is required');
    }

    // Check size
    if (value.size > this.maxSize) {
      throw new BadRequestException(
        `File size exceeds ${this.maxSize / 1024 / 1024}MB limit`,
      );
    }

    // Check type
    const fileType = value.mimetype;
    if (!this.allowedTypes.includes(fileType)) {
      throw new BadRequestException(
        `File type ${fileType} not allowed. Allowed: ${this.allowedTypes.join(', ')}`,
      );
    }

    return value;
  }
}

// Usage
@Controller('upload')
export class UploadController {
  @Post('file')
  @UseInterceptors(FileInterceptor('file'))
  uploadFile(
    @UploadedFile(ParseFilePipe) file: Express.Multer.File,
  ) {
    return { filename: file.originalname };
  }
}
```

---

## 🎯 Pipe Precedence

```typescript
// Pipes are executed in order:
// 1. Method-level pipes
// 2. Controller-level pipes
// 3. Global pipes
// 4. Param-specific pipes

@Controller('users')
@UsePipes(new ValidationPipe())  // Controller-level
export class UsersController {
  @Get(':id')
  findOne(
    @Param('id', ParseIntPipe) id: number,  // Param-specific
  ) {
    return this.usersService.findOne(id);
  }

  @Post()
  @UsePipes(new ValidateUserPipe())  // Method-level
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

---

## 🌍 Global Pipes

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

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
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

---

## 📋 Pipe Options

```typescript
// ValidationPipe options
new ValidationPipe({
  transform: true,              // Transform to DTO class
  whitelist: true,              // Strip extra properties
  forbidNonWhitelisted: true,   // Throw error on extra properties
  disableErrorMessages: false,  // Show error details
  validateCustomDecorators: true,
  transformOptions: {
    enableImplicitConversion: true,
    excludeExtraneousValues: true,
  },
  validationError: {
    target: false,
    value: false,
  },
});
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-DTO-Validation]] — DTO & Validation
- [[NestJS-Filters]] — Exception filters
- [[NestJS-Interceptors]] — Interceptors
