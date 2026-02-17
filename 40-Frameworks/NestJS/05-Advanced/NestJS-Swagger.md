---
created: 2026-02-17
tags:
  - nestjs
  - swagger
  - openapi
  - documentation
aliases:
  - NestJS Swagger
  - NestJS API Documentation
related:
  - NestJS-MOC
  - NestJS-DTO-Validation
  - NestJS-Controllers
---

# NestJS — Swagger (OpenAPI)

> [!SUMMARY] Обзор
> Swagger/OpenAPI документация для NestJS API. Автоматическая генерация, decorators, custom schemas.

---

## 📦 Установка

```bash
npm install @nestjs/swagger swagger-ui-express
```

---

## 🔧 Setup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger config
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API documentation')
    .setVersion('1.0')
    .addTag('users', 'Users management')
    .addTag('auth', 'Authentication')
    .addBearerAuth()
    .addServer('http://localhost:3000', 'Development')
    .addServer('https://api.example.com', 'Production')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
```

Swagger UI: `http://localhost:3000/api`
JSON spec: `http://localhost:3000/api-json`

---

## 📝 DTO Decorators

```typescript
// users/dto/create-user.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  @ApiProperty({
    description: 'User name',
    example: 'John Doe',
    minLength: 2,
    maxLength: 50,
  })
  name: string;

  @ApiProperty({
    description: 'User email',
    example: 'john@example.com',
    format: 'email',
  })
  email: string;

  @ApiProperty({
    description: 'User password',
    minLength: 8,
    format: 'password',
    writeOnly: true,
  })
  password: string;

  @ApiPropertyOptional({
    description: 'User age',
    minimum: 18,
    maximum: 120,
    default: 18,
  })
  age?: number;

  @ApiPropertyOptional({
    description: 'User role',
    enum: UserRole,
    default: UserRole.USER,
  })
  role?: UserRole;

  @ApiPropertyOptional({
    description: 'User tags',
    type: [String],
    example: ['tag1', 'tag2'],
  })
  tags?: string[];
}
```

---

## 🎮 Controller Decorators

```typescript
// users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  ParseIntPipe,
  HttpStatus,
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiParam,
  ApiQuery,
  ApiBody,
  ApiResponse,
  ApiBearerAuth,
  ApiExcludeEndpoint,
} from '@nestjs/swagger';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { UserResponseDto } from './dto/user-response.dto';

@ApiTags('users')  // Group in Swagger
@ApiBearerAuth()   // Require auth
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @ApiOperation({ summary: 'Get all users' })
  @ApiQuery({ name: 'page', required: false, type: Number, default: 1 })
  @ApiQuery({ name: 'limit', required: false, type: Number, default: 10 })
  @ApiQuery({ name: 'search', required: false, type: String })
  @ApiResponse({
    status: 200,
    description: 'Users retrieved successfully',
    type: [UserResponseDto],
  })
  @ApiResponse({ status: 401, description: 'Unauthorized' })
  findAll(
    @Query('page', new ParseIntPipe({ optional: true })) page = 1,
    @Query('limit', new ParseIntPipe({ optional: true })) limit = 10,
    @Query('search') search?: string,
  ) {
    return this.usersService.findAll({ page, limit, search });
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', description: 'User ID', example: 1 })
  @ApiResponse({
    status: 200,
    description: 'User retrieved successfully',
    type: UserResponseDto,
  })
  @ApiResponse({ status: 404, description: 'User not found' })
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }

  @Post()
  @ApiOperation({ summary: 'Create new user' })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({
    status: 201,
    description: 'User created successfully',
    type: UserResponseDto,
  })
  @ApiResponse({ status: 400, description: 'Bad request' })
  @ApiResponse({ status: 409, description: 'Email already exists' })
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Put(':id')
  @ApiOperation({ summary: 'Update user' })
  @ApiParam({ name: 'id', description: 'User ID', example: 1 })
  @ApiBody({ type: UpdateUserDto })
  @ApiResponse({
    status: 200,
    description: 'User updated successfully',
    type: UserResponseDto,
  })
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Delete user' })
  @ApiParam({ name: 'id', description: 'User ID', example: 1 })
  @ApiResponse({ status: 204, description: 'User deleted' })
  @ApiResponse({ status: 404, description: 'User not found' })
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }

  @Get('admin/stats')
  @ApiExcludeEndpoint()  // Exclude from Swagger
  getAdminStats() {
    return this.usersService.getStats();
  }
}
```

---

## 📊 Response Model

```typescript
// users/dto/user-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { Exclude, Expose } from 'class-transformer';

export class UserResponseDto {
  @ApiProperty({ example: 1 })
  id: number;

  @ApiProperty({ example: 'john@example.com' })
  email: string;

  @ApiProperty({ example: 'John Doe' })
  name: string;

  @ApiProperty({ example: 'user' })
  role: string;

  @ApiProperty({ example: '2026-02-17T12:00:00.000Z' })
  createdAt: Date;

  @Exclude()  // Exclude from response
  password: string;
}
```

---

## 🏛️ Schema Generation

```typescript
// users/users.module.ts
import { ApiProperty } from '@nestjs/swagger';

export class PaginatedResponseDto<T> {
  @ApiProperty({ type: [Object] })
  data: T[];

  @ApiProperty()
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  };
}

// Controller
@Get()
@ApiResponse({
  status: 200,
  type: PaginatedResponseDto(UserResponseDto),
})
findAll() {
  return this.usersService.findAll();
}
```

---

## 🔐 Security Decorators

```typescript
// Add Bearer auth
@ApiBearerAuth()
@Controller('users')
export class UsersController {}

// Add API key auth
@ApiSecurity('api_key')
@Controller('admin')
export class AdminController {}

// Add OAuth2
@ApiSecurity('oauth2', ['read', 'write'])
@Controller('oauth')
export class OAuthController {}

// Multiple security
@ApiBearerAuth()
@ApiSecurity('api_key')
@Controller('secure')
export class SecureController {}
```

---

## 🌍 Global Configuration

```typescript
// app.module.ts
import { SwaggerModule } from '@nestjs/swagger';

@Module({
  imports: [
    // ...
  ],
})
export class AppModule implements OnModuleInit {
  constructor(private app: NestApplication) {}

  onModuleInit() {
    const config = new DocumentBuilder()
      .setTitle('My API')
      .setDescription('API documentation')
      .setVersion('1.0')
      .addBearerAuth()
      .build();

    const document = SwaggerModule.createDocument(this.app, config);
    SwaggerModule.setup('api', this.app, document);
  }
}
```

---

## 📋 Environment

```bash
# .env
SWAGGER_ENABLED=true
SWAGGER_PATH=api
API_TITLE=My API
API_VERSION=1.0
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-DTO-Validation]] — DTO & Validation
- [[NestJS-Controllers]] — Controllers
- [[NestJS-Microservices]] — Microservices
