---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - backend
  - nodejs
  - api
aliases:
  - TypeScript Backend Cheatsheet
  - Backend Development TS
related:
  - NestJS-Cheatsheet
  - Docker-Cheatsheet
  - PostgreSQL-Cheatsheet
  - TypeScript-Patterns
  - Auth-Patterns
  - Backend-Architecture-MOC
---

# TypeScript Backend — Полная шпаргалка

> [!SUMMARY] Обзор
> Шпаргалка по разработке backend на TypeScript: сервисы, контроллеры, резолверы, middleware, DTO, валидация. Паттерны и лучшие практики для Go, PHP, Rust и TypeScript.

---

## 📚 Теория

### Архитектурные слои

```
┌─────────────────────────────────────────────────────────┐
│                    HTTP / GraphQL                        │
│                     (Transport Layer)                    │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   Controllers / Resolvers                │
│                   (API Layer / Router)                   │
│  • Обработка HTTP запросов                               │
│  • Маршрутизация                                         │
│  • Authentication / Authorization                        │
│  • Request/Response трансформация                        │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                      Services                            │
│                   (Business Logic Layer)                 │
│  • Бизнес-логика                                         │
│  • Валидация бизнес-правил                               │
│  • Оркестрация данных                                    │
│  • Транзакции                                            │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                 Repositories / DAO                       │
│                   (Data Access Layer)                    │
│  • Работа с БД                                           │
│  • Кэширование                                           │
│  • Маппинг данных                                        │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    Database                              │
│              (PostgreSQL, MySQL, MongoDB)                │
└─────────────────────────────────────────────────────────┘
```

### Что нужно vs Что не нужно

| Компонент | Нужно | Не нужно | Когда использовать |
|-----------|-------|----------|-------------------|
| **Controller** | ✅ HTTP специфика, маршрутизация, статусы | ❌ Бизнес-логика, SQL запросы | Всегда для HTTP endpoints |
| **Service** | ✅ Бизнес-логика, валидация, транзакции | ❌ HTTP детали, прямые SQL | Всегда для бизнес-логики |
| **Repository** | ✅ DB запросы, маппинг, кэш | ❌ Бизнес-правила | Для сложной работы с БД |
| **DTO** | ✅ Валидация, типизация, документация | ❌ Бизнес-логика в методах | Всегда для API границ |
| **Resolver** | ✅ GraphQL специфика, DataLoader | ❌ Бизнес-логика | Только для GraphQL |
| **Middleware** | ✅ Cross-cutting: auth, log, cors | ❌ Бизнес-логика | Для сквозной функциональности |

---

## ⚡ Быстрый старт

### Минимальная структура проекта

```
src/
├── main.ts                    # Точка входа
├── config/                    # Конфигурация
│   ├── database.ts
│   └── app.ts
├── common/                    # Общие модули
│   ├── decorators/
│   ├── guards/
│   ├── interceptors/
│   ├── filters/
│   └── pipes/
├── modules/                   # Функциональные модули
│   ├── users/
│   │   ├── users.module.ts
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── dto/
│   │   │   ├── create-user.dto.ts
│   │   │   └── update-user.dto.ts
│   │   └── entities/
│   │       └── user.entity.ts
│   └── auth/
└── database/                  # Миграции, seeders
```

---

## 🔧 Практические примеры

### 1. Контроллеры (Controllers)

**Что это:** Обработчики HTTP запросов. Маршрутизация, request/response.

**Что нужно:**
- ✅ Декораторы маршрутов (`@Get`, `@Post`)
- ✅ DTO для body/query/params
- ✅ Guards для авторизации
- ✅ Interceptors для трансформации
- ✅ HTTP статус коды

**Что не нужно:**
- ❌ Бизнес-логика
- ❌ Прямые SQL запросы
- ❌ Сложные вычисления

```typescript
// users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  Request,
  Response,
  Header,
  HttpCode,
  HttpStatus,
  UseGuards,
  UseInterceptors,
  ParseIntPipe,
  ValidationPipe,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { AuthGuard } from '../auth/auth.guard';
import { Roles } from '../common/decorators/roles.decorator';
import { Public } from '../common/decorators/public.decorator';

@Controller('api/v1/users')  // ✅ Base path
export class UsersController {
  // ✅ Внедряем только сервис (не repository!)
  constructor(private readonly usersService: UsersService) {}

  // ───────────────────────────────────────────────────────
  // GET /api/v1/users
  // ───────────────────────────────────────────────────────
  @Get()
  @UseGuards(AuthGuard)
  async findAll(
    @Query('page', new ParseIntPipe({ optional: true })) page = 1,
    @Query('limit', new ParseIntPipe({ optional: true })) limit = 10,
    @Query('search') search?: string,
  ) {
    // ✅ Делегируем бизнес-логику сервису
    return this.usersService.findAll({ page, limit, search });
  }

  // ───────────────────────────────────────────────────────
  // GET /api/v1/users/:id
  // ───────────────────────────────────────────────────────
  @Get(':id')
  @UseGuards(AuthGuard)
  async findOne(@Param('id', ParseIntPipe) id: number) {
    // ✅ Обработка ошибок на уровне контроллера
    const user = await this.usersService.findOne(id);
    
    if (!user) {
      // ✅ Возвращаем правильный HTTP статус
      throw new NotFoundException(`User ${id} not found`);
    }
    
    return user;
  }

  // ───────────────────────────────────────────────────────
  // POST /api/v1/users
  // ───────────────────────────────────────────────────────
  @Post()
  @Public()  // ✅ Публичный endpoint
  @HttpCode(HttpStatus.CREATED)  // ✅ Правильный статус
  @UseInterceptors(ClassSerializerInterceptor)  // ✅ Скрытие полей
  async create(
    @Body(new ValidationPipe({ transform: true })) 
    createUserDto: CreateUserDto,
  ) {
    // ✅ Возвращаем созданный ресурс
    const user = await this.usersService.create(createUserDto);
    
    // ✅ Можно добавить заголовок Location
    // response.setHeader('Location', `/users/${user.id}`);
    
    return user;
  }

  // ───────────────────────────────────────────────────────
  // PUT /api/v1/users/:id
  // ───────────────────────────────────────────────────────
  @Put(':id')
  @UseGuards(AuthGuard)
  @Roles('admin')  // ✅ Только админы
  async update(
    @Param('id', ParseIntPipe) id: number,
    @Body(new ValidationPipe({ transform: true })) 
    updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  // ───────────────────────────────────────────────────────
  // DELETE /api/v1/users/:id
  // ───────────────────────────────────────────────────────
  @Delete(':id')
  @UseGuards(AuthGuard)
  @Roles('admin')
  @HttpCode(HttpStatus.NO_CONTENT)  // ✅ 204 No Content
  async remove(@Param('id', ParseIntPipe) id: number) {
    await this.usersService.remove(id);
    // ✅ Ничего не возвращаем для DELETE
  }

  // ───────────────────────────────────────────────────────
  // PATCH /api/v1/users/:id/activate
  // ───────────────────────────────────────────────────────
  @Patch(':id/activate')
  @UseGuards(AuthGuard)
  @Roles('admin')
  async activate(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.activate(id);
  }
}
```

---

### 2. Сервисы (Services)

**Что это:** Бизнес-логика приложения. Оркестрация, валидация, транзакции.

**Что нужно:**
- ✅ Вся бизнес-логика здесь
- ✅ Валидация бизнес-правил
- ✅ Транзакции
- ✅ Вызов других сервисов
- ✅ Логирование

**Что не нужно:**
- ❌ HTTP специфика (req, res, status)
- ❌ Прямые SQL (это задача repository)
- ❌ DTO (используйте интерфейсы)

```typescript
// users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
  BadRequestException,
  Logger,
} from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { EmailService } from '../email/email.service';
import { CacheService } from '../cache/cache.service';

@Injectable()  // ✅ Decorator для DI
export class UsersService {
  // ✅ Логгер для сервиса
  private readonly logger = new Logger(UsersService.name);

  constructor(
    // ✅ Repository для работы с БД
    @InjectRepository(User)
    private readonly usersRepository: Repository<User>,
    
    // ✅ Другие сервисы для оркестрации
    private readonly emailService: EmailService,
    private readonly cacheService: CacheService,
  ) {}

  // ───────────────────────────────────────────────────────
  // CREATE
  // ───────────────────────────────────────────────────────
  async create(createUserDto: CreateUserDto): Promise<User> {
    // ✅ Проверка на дубликат
    const existing = await this.usersRepository.findOne({
      where: { email: createUserDto.email },
    });

    if (existing) {
      // ✅ Бизнес-валидация
      throw new ConflictException('Email already exists');
    }

    // ✅ Проверка сложности пароля
    this.validatePasswordStrength(createUserDto.password);

    // ✅ Хэширование пароля
    const hashedPassword = await this.hashPassword(createUserDto.password);

    // ✅ Создание entity
    const user = this.usersRepository.create({
      ...createUserDto,
      password: hashedPassword,
    });

    // ✅ Сохранение
    const savedUser = await this.usersRepository.save(user);

    // ✅ Логирование
    this.logger.log(`User created: ${savedUser.id}`);

    // ✅ Асинхронные побочные эффекты (не блокируем ответ)
    this.emailService.sendWelcomeEmail(savedUser.email).catch(err => {
      this.logger.error('Failed to send welcome email', err);
    });

    // ✅ Инвалидация кэша
    await this.cacheService.invalidate('users:list');

    return savedUser;
  }

  // ───────────────────────────────────────────────────────
  // FIND ALL с пагинацией
  // ───────────────────────────────────────────────────────
  async findAll(pagination: { 
    page: number; 
    limit: number; 
    search?: string 
  }) {
    const { page, limit, search } = pagination;

    // ✅ Проверка кэша
    const cacheKey = `users:list:${page}:${limit}:${search || 'all'}`;
    const cached = await this.cacheService.get(cacheKey);
    if (cached) {
      return cached;
    }

    // ✅ Построение query
    const queryBuilder = this.usersRepository.createQueryBuilder('user');
    
    if (search) {
      queryBuilder.where('user.name ILIKE :search', { 
        search: `%${search}%` 
      });
    }

    // ✅ Пагинация
    const [data, total] = await queryBuilder
      .skip((page - 1) * limit)
      .take(limit)
      .orderBy('user.createdAt', 'DESC')
      .getManyAndCount();

    const result = {
      data,
      meta: {
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
        hasNext: page * limit < total,
        hasPrev: page > 1,
      },
    };

    // ✅ Кэширование
    await this.cacheService.set(cacheKey, result, 300); // 5 минут

    return result;
  }

  // ───────────────────────────────────────────────────────
  // FIND ONE
  // ───────────────────────────────────────────────────────
  async findOne(id: number): Promise<User | null> {
    // ✅ Проверка кэша
    const cached = await this.cacheService.get(`user:${id}`);
    if (cached) {
      return cached;
    }

    const user = await this.usersRepository.findOne({
      where: { id },
      // ✅ Исключаем чувствительные данные
      select: ['id', 'name', 'email', 'role', 'createdAt'],
    });

    if (!user) {
      return null;
    }

    // ✅ Кэширование
    await this.cacheService.set(`user:${id}`, user, 600);

    return user;
  }

  // ───────────────────────────────────────────────────────
  // UPDATE
  // ───────────────────────────────────────────────────────
  async update(id: number, updateUserDto: UpdateUserDto): Promise<User> {
    // ✅ Проверяем существование
    const user = await this.findOne(id);
    
    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }

    // ✅ Бизнес-валидация
    if (updateUserDto.email && updateUserDto.email !== user.email) {
      const existing = await this.usersRepository.findOne({
        where: { email: updateUserDto.email },
      });
      
      if (existing) {
        throw new ConflictException('Email already exists');
      }
    }

    // ✅ Обновление
    Object.assign(user, updateUserDto);
    const updated = await this.usersRepository.save(user);

    // ✅ Логирование
    this.logger.log(`User updated: ${id}`);

    // ✅ Инвалидация кэша
    await this.cacheService.invalidateMulti([
      `user:${id}`,
      'users:list:*',
    ]);

    return updated;
  }

  // ───────────────────────────────────────────────────────
  // DELETE
  // ───────────────────────────────────────────────────────
  async remove(id: number): Promise<void> {
    const user = await this.findOne(id);
    
    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }

    // ✅ Бизнес-правило: нельзя удалить последнего админа
    if (user.role === 'admin') {
      const adminCount = await this.usersRepository.count({
        where: { role: 'admin' },
      });
      
      if (adminCount <= 1) {
        throw new BadRequestException('Cannot delete last admin');
      }
    }

    // ✅ Soft delete (если нужно)
    await this.usersRepository.softDelete(id);
    
    // ✅ Или hard delete
    // await this.usersRepository.delete(id);

    this.logger.log(`User deleted: ${id}`);
    
    await this.cacheService.invalidateMulti([
      `user:${id}`,
      'users:list:*',
    ]);
  }

  // ───────────────────────────────────────────────────────
  // ACTIVATE / DEACTIVATE
  // ───────────────────────────────────────────────────────
  async activate(id: number): Promise<User> {
    return this.updateStatus(id, 'active');
  }

  async deactivate(id: number): Promise<User> {
    return this.updateStatus(id, 'inactive');
  }

  // ───────────────────────────────────────────────────────
  // ТРАНЗАКЦИИ
  // ───────────────────────────────────────────────────────
  async transferCredits(
    fromUserId: number,
    toUserId: number,
    amount: number,
  ): Promise<void> {
    // ✅ Используем транзакцию
    const queryRunner = this.usersRepository.manager.connection.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      const fromUser = await queryRunner.manager.findOne(User, { 
        where: { id: fromUserId },
        lock: { mode: 'pessimistic_write' },  // ✅ Блокировка
      });
      
      const toUser = await queryRunner.manager.findOne(User, { 
        where: { id: toUserId },
        lock: { mode: 'pessimistic_write' },
      });

      if (!fromUser || !toUser) {
        throw new NotFoundException('User not found');
      }

      if (fromUser.credits < amount) {
        throw new BadRequestException('Insufficient credits');
      }

      // ✅ Обновление в транзакции
      fromUser.credits -= amount;
      toUser.credits += amount;

      await queryRunner.manager.save(fromUser);
      await queryRunner.manager.save(toUser);

      await queryRunner.commitTransaction();
      
      this.logger.log(`Credits transferred: ${fromUserId} -> ${toUserId}, ${amount}`);
    } catch (error) {
      await queryRunner.rollbackTransaction();
      this.logger.error('Transfer failed', error);
      throw error;
    } finally {
      await queryRunner.release();
    }
  }

  // ───────────────────────────────────────────────────────
  // ПРИВАТНЫЕ МЕТОДЫ
  // ───────────────────────────────────────────────────────
  private validatePasswordStrength(password: string): void {
    const minLength = 8;
    const hasUppercase = /[A-Z]/.test(password);
    const hasLowercase = /[a-z]/.test(password);
    const hasNumbers = /\d/.test(password);

    if (
      password.length < minLength ||
      !hasUppercase ||
      !hasLowercase ||
      !hasNumbers
    ) {
      throw new BadRequestException(
        'Password must be at least 8 chars with uppercase, lowercase and number',
      );
    }
  }

  private async hashPassword(password: string): Promise<string> {
    const bcrypt = require('bcrypt');
    return bcrypt.hash(password, 10);
  }

  private async updateStatus(id: number, status: string): Promise<User> {
    const user = await this.findOne(id);
    
    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }

    user.status = status;
    return this.usersRepository.save(user);
  }
}
```

---

### 3. Репозитории (Repositories / DAO)

**Что это:** Абстракция над работой с БД. SQL запросы, маппинг.

**Что нужно:**
- ✅ Все SQL/ORM запросы
- ✅ Маппинг entity ↔ DTO
- ✅ Кэширование на уровне БД
- ✅ Сложные query builder

**Что не нужно:**
- ❌ Бизнес-логика
- ❌ Валидация (кроме DB constraints)

```typescript
// users.repository.ts
import { Injectable } from '@nestjs/common';
import { DataSource, Repository, QueryBuilder } from 'typeorm';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersRepository extends Repository<User> {
  constructor(private dataSource: DataSource) {
    super(User, dataSource.createEntityManager());
  }

  // ───────────────────────────────────────────────────────
  // CUSTOM QUERIES
  // ───────────────────────────────────────────────────────
  
  async findByEmail(email: string): Promise<User | null> {
    return this.createQueryBuilder('user')
      .where('user.email = :email', { email })
      .getOne();
  }

  async findActiveUsers(): Promise<User[]> {
    return this.createQueryBuilder('user')
      .where('user.status = :status', { status: 'active' })
      .orderBy('user.createdAt', 'DESC')
      .getMany();
  }

  async findWithRoles(): Promise<User[]> {
    return this.createQueryBuilder('user')
      .leftJoinAndSelect('user.roles', 'role')
      .getMany();
  }

  // ───────────────────────────────────────────────────────
  // COMPLEX QUERIES
  // ───────────────────────────────────────────────────────
  
  async searchUsers(query: string, limit = 10): Promise<User[]> {
    return this.createQueryBuilder('user')
      .where('user.name ILIKE :query', { query: `%${query}%` })
      .orWhere('user.email ILIKE :query', { query: `%${query}%` })
      .limit(limit)
      .getMany();
  }

  async findWithStats(userId: number): Promise<any> {
    return this.createQueryBuilder('user')
      .leftJoin('user.orders', 'orders')
      .select([
        'user.id',
        'user.name',
        'user.email',
        'COUNT(orders.id) as orderCount',
        'SUM(orders.total) as totalSpent',
      ])
      .where('user.id = :userId', { userId })
      .groupBy('user.id')
      .getRawOne();
  }

  // ───────────────────────────────────────────────────────
  // BULK OPERATIONS
  // ───────────────────────────────────────────────────────
  
  async bulkCreate(users: CreateUserDto[]): Promise<User[]> {
    const entities = users.map(u => this.create(u));
    return this.save(entities);
  }

  async bulkUpdateStatus(ids: number[], status: string): Promise<void> {
    await this.createQueryBuilder()
      .update(User)
      .set({ status })
      .where('id IN (:...ids)', { ids })
      .execute();
  }

  // ───────────────────────────────────────────────────────
  // SOFT DELETE
  // ───────────────────────────────────────────────────────
  
  async softDeleteById(id: number): Promise<void> {
    await this.softDelete(id);
  }

  async restore(id: number): Promise<void> {
    await this.restore(id);
  }
}
```

---

### 4. DTO (Data Transfer Objects)

**Что это:** Объекты для передачи данных между слоями. Валидация, типизация.

**Что нужно:**
- ✅ class-validator декораторы
- ✅ Типизация всех полей
- ✅ Partial/Omit/Pick для разных случаев
- ✅ Transform для конвертации

**Что не нужно:**
- ❌ Бизнес-логика в методах
- ❌ Зависимости от других модулей

```typescript
// ───────────────────────────────────────────────────────
// CREATE DTO
// ───────────────────────────────────────────────────────
import {
  IsString,
  IsEmail,
  IsNotEmpty,
  IsOptional,
  IsInt,
  IsBoolean,
  IsArray,
  IsEnum,
  IsUrl,
  IsPhoneNumber,
  MinLength,
  MaxLength,
  Min,
  Max,
  Matches,
  ValidateNested,
  IsObject,
} from 'class-validator';
import { Type } from 'class-transformer';

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(128)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/, {
    message: 'Password must contain uppercase, lowercase, number and special char',
  })
  password: string;

  @IsOptional()
  @IsInt()
  @Min(18)
  @Max(120)
  age?: number;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;

  @IsOptional()
  @IsBoolean()
  isActive?: boolean;

  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  tags?: string[];

  @IsOptional()
  @IsObject()
  @ValidateNested()
  @Type(() => AddressDto)
  address?: AddressDto;
}

// ───────────────────────────────────────────────────────
// NESTED DTO
// ───────────────────────────────────────────────────────
export class AddressDto {
  @IsString()
  @IsNotEmpty()
  street: string;

  @IsString()
  @IsNotEmpty()
  city: string;

  @IsString()
  @IsNotEmpty()
  zipCode: string;

  @IsString()
  @IsOptional()
  state?: string;

  @IsString()
  @IsNotEmpty()
  country: string;
}

// ───────────────────────────────────────────────────────
// UPDATE DTO (Partial)
// ───────────────────────────────────────────────────────
// Все поля опциональны
export class UpdateUserDto {
  @IsOptional()
  @IsString()
  @MinLength(2)
  name?: string;

  @IsOptional()
  @IsEmail()
  email?: string;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;

  @IsOptional()
  @IsBoolean()
  isActive?: boolean;
}

// Или используем PartialType из NestJS
// import { PartialType } from '@nestjs/swagger';
// export class UpdateUserDto extends PartialType(CreateUserDto) {}

// ───────────────────────────────────────────────────────
// RESPONSE DTO (исключаем чувствительные данные)
// ───────────────────────────────────────────────────────
import { OmitType, PickType } from '@nestjs/swagger';

// Исключаем password
export class UserResponseDto extends OmitType(CreateUserDto, ['password'] as const) {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

// Или только нужные поля
export class UserPublicDto extends PickType(UserResponseDto, ['id', 'name', 'email'] as const) {}

// ───────────────────────────────────────────────────────
// QUERY PARAMS DTO
// ───────────────────────────────────────────────────────
export class FindUsersQueryDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsEnum(['name', 'email', 'createdAt'])
  sortBy?: string = 'createdAt';

  @IsOptional()
  @IsEnum(['ASC', 'DESC'])
  order?: 'ASC' | 'DESC' = 'DESC';
}

// ───────────────────────────────────────────────────────
// CUSTOM VALIDATORS
// ───────────────────────────────────────────────────────
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
} from 'class-validator';

// Кастомный валидатор: поле должно быть больше другого
@ValidatorConstraint({ async: false })
export class IsGreaterThanConstraint implements ValidatorConstraintInterface {
  validate(value: any, args: any) {
    const [relatedPropertyName] = args.constraints;
    const relatedValue = args.object[relatedPropertyName];
    return value > relatedValue;
  }

  defaultMessage(args: any) {
    const [relatedPropertyName] = args.constraints;
    return `$property must be greater than ${relatedPropertyName}`;
  }
}

export function IsGreaterThan(property: string, validationOptions?: ValidationOptions) {
  return function (object: any, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [property],
      validator: IsGreaterThanConstraint,
    });
  };
}

// Использование
export class EventDto {
  startDate: Date;
  
  @IsGreaterThan('startDate', { message: 'End date must be after start date' })
  endDate: Date;
}
```

---

### 5. GraphQL Resolvers

**Что это:** Обработчики GraphQL запросов. Аналог контроллеров для GraphQL.

**Что нужно:**
- ✅ Декораторы `@Query`, `@Mutation`, `@ResolveField`
- ✅ DataLoader для N+1 проблемы
- ✅ Типизация через GraphQL типы
- ✅ Делегирование бизнес-логики сервисам

**Что не нужно:**
- ❌ Бизнес-логика
- ❌ Прямые SQL запросы

```typescript
// users.resolver.ts
import {
  Resolver,
  Query,
  Mutation,
  ResolveField,
  Parent,
  Args,
  Int,
  Context,
} from '@nestjs/graphql';
import { UseGuards } from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';
import { AuthGuard } from '../auth/auth.guard';
import { GqlExecutionContext } from '@nestjs/graphql';
import { DataLoader } from '../common/loaders/user.loader';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  // ───────────────────────────────────────────────────────
  // QUERIES
  // ───────────────────────────────────────────────────────
  
  @Query(() => [User])
  @UseGuards(AuthGuard)
  async users(
    @Args('page', { type: () => Int, defaultValue: 1 }) page: number,
    @Args('limit', { type: () => Int, defaultValue: 10 }) limit: number,
  ): Promise<User[]> {
    const result = await this.usersService.findAll({ page, limit });
    return result.data;
  }

  @Query(() => User, { nullable: true })
  async user(@Args('id', { type: () => Int }) id: number): Promise<User | null> {
    return this.usersService.findOne(id);
  }

  // ───────────────────────────────────────────────────────
  // MUTATIONS
  // ───────────────────────────────────────────────────────
  
  @Mutation(() => User)
  async createUser(
    @Args('input') input: CreateUserInput,
  ): Promise<User> {
    return this.usersService.create(input);
  }

  @Mutation(() => User)
  @UseGuards(AuthGuard)
  async updateUser(
    @Args('id', { type: () => Int }) id: number,
    @Args('input') input: UpdateUserInput,
  ): Promise<User> {
    return this.usersService.update(id, input);
  }

  @Mutation(() => Boolean)
  @UseGuards(AuthGuard)
  async deleteUser(@Args('id', { type: () => Int }) id: number): Promise<boolean> {
    await this.usersService.remove(id);
    return true;
  }

  // ───────────────────────────────────────────────────────
  // RESOLVE FIELD (для связанных данных)
  // ───────────────────────────────────────────────────────
  
  // ✅ Решаем N+1 проблему с DataLoader
  @ResolveField(() => [Post])
  async posts(
    @Parent() user: User,
    @DataLoader('posts') postsLoader: DataLoader<User, Post[]>,
  ): Promise<Post[]> {
    return postsLoader.load(user);
  }

  @ResolveField(() => Int)
  async totalPosts(@Parent() user: User): Promise<number> {
    // ✅ Делегируем сервису
    return this.usersService.getUserPostCount(user.id);
  }

  // ✅ Вычисляемое поле
  @ResolveField(() => String)
  displayName(@Parent() user: User): string {
    return `${user.firstName} ${user.lastName}`.trim();
  }

  // ✅ Скрытие чувствительных данных
  @ResolveField(() => String, { nullable: true })
  email(@Parent() user: User, @Context() context: any): string | null {
    // ✅ Показываем email только владельцу или админу
    const currentUser = context.req.user;
    if (currentUser?.id === user.id || currentUser?.role === 'admin') {
      return user.email;
    }
    return null;
  }
}

// ───────────────────────────────────────────────────────
// INPUT TYPES (аналог DTO для GraphQL)
// ───────────────────────────────────────────────────────
import { InputType, Field } from '@nestjs/graphql';

@InputType()
export class CreateUserInput {
  @Field()
  firstName: string;

  @Field()
  lastName: string;

  @Field()
  email: string;

  @Field()
  password: string;

  @Field({ nullable: true })
  age?: number;
}

@InputType()
export class UpdateUserInput {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field({ nullable: true })
  email?: string;
}
```

---

### 6. Middleware / Guards / Interceptors

```typescript
// ───────────────────────────────────────────────────────
// AUTH GUARD
// ───────────────────────────────────────────────────────
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // ✅ Поддержка HTTP и GraphQL
    let token: string | undefined;
    
    if (context.getType() === 'http') {
      const request = context.switchToHttp().getRequest();
      token = this.extractTokenFromHeader(request);
    } else {
      const ctx = GqlExecutionContext.create(context);
      token = this.extractTokenFromHeader(ctx.getContext().req);
    }

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      const payload = await this.jwtService.verifyAsync(token);
      // ✅ Добавляем пользователя в request
      if (context.getType() === 'http') {
        context.switchToHttp().getRequest()['user'] = payload;
      } else {
        GqlExecutionContext.create(context).getContext().req.user = payload;
      }
    } catch {
      throw new UnauthorizedException('Invalid token');
    }

    return true;
  }

  private extractTokenFromHeader(request: any): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}

// ───────────────────────────────────────────────────────
// ROLES GUARD
// ───────────────────────────────────────────────────────
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// ───────────────────────────────────────────────────────
// LOGGING INTERCEPTOR
// ───────────────────────────────────────────────────────
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

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const startTime = Date.now();

    return next
      .handle()
      .pipe(
        tap(() => {
          const response = context.switchToHttp().getResponse();
          const { statusCode } = response;
          const duration = Date.now() - startTime;
          
          this.logger.log(`${method} ${url} ${statusCode} ${duration}ms`);
        }),
      );
  }
}

// ───────────────────────────────────────────────────────
// TRANSFORM INTERCEPTOR
// ───────────────────────────────────────────────────────
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
    return next.handle().pipe(
      map(data => ({
        data,
        meta: {
          timestamp: new Date().toISOString(),
          path: context.switchToHttp().getRequest().url,
        },
      })),
    );
  }
}

// ───────────────────────────────────────────────────────
// EXCEPTION FILTER
// ───────────────────────────────────────────────────────
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger('Exceptions');

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const message =
      exception instanceof HttpException
        ? exception.getMessage()
        : 'Internal server error';

    // ✅ Логирование
    this.logger.error(
      `${request.method} ${request.url}`,
      exception instanceof Error ? exception.stack : '',
    );

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Контроллер → Сервис → Repository
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}  // ✅
}

// 2. DTO для всех входящих данных
@Post()
create(@Body() dto: CreateUserDto) {}  // ✅

// 3. Guards для авторизации
@Get('admin')
@UseGuards(AuthGuard, RolesGuard)
@Roles('admin')
getAdminData() {}  // ✅

// 4. Interceptors для трансформации
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findAll() {}  // ✅

// 5. Exception filters для обработки ошибок
app.useGlobalFilters(new AllExceptionsFilter());  // ✅

// 6. Логирование в сервисах
private readonly logger = new Logger(MyService.name);
this.logger.log('Message');  // ✅

// 7. Транзакции для критичных операций
await this.dataSource.transaction(async (manager) => {
  // ✅
});

// 8. Кэширование
await this.cache.get(key);
await this.cache.set(key, value, ttl);  // ✅
```

### ❌ Не делать

```typescript
// 1. Бизнес-логика в контроллере
@Get()
async findAll() {
  const users = await this.repo.find();  // ❌
  return users.filter(u => u.active);    // ❌
}

// 2. Прямой repository в контроллере
@Controller('users')
export class UsersController {
  constructor(
    @InjectRepository(User)  // ❌
    private readonly repo: Repository<User>,
  ) {}
}

// 3. any вместо DTO
@Post()
create(@Body() data: any) {}  // ❌

// 4. Игнорирование ошибок
try {
  await this.service.doSomething();
} catch (e) {
  console.error(e);  // ❌
}

// 5. Возврат чувствительных данных
return user;  // ❌ (password внутри)

// 6. N+1 запросы
@ResolveField()
async posts(@Parent() user: User) {
  return this.postsRepo.find({ where: { userId: user.id } });  // ❌
}
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| **N+1 запросы** | Запрос в цикле | Используйте DataLoader |
| **Утечка паролей** | Возврат entity напрямую | Используйте DTO/OmitType |
| **Нет валидации** | @Body() без DTO | Добавьте ValidationPipe + DTO |
| **Бизнес-логика в контроллере** | Нарушение слоев | Переместите в Service |
| **Прямой SQL в контроллере** | Нарушение слоев | Используйте Repository |
| **Нет обработки ошибок** | Падение приложения | Exception Filters |
| **Нет авторизации** | Доступ всем | Guards + @UseGuards |
| **Кэш не инвалидируется** | Устаревшие данные | Invalidate при изменении |

---

## 🔗 Связанные заметки

- [[NestJS-Cheatsheet]] — NestJS фреймворк
- [[PostgreSQL-Cheatsheet]] — PostgreSQL
- [[TypeScript-Patterns]] — TypeScript паттерны
- [[Auth-Patterns]] — Аутентификация
- [[MOC-Backend]] — Backend архитектура
- [[Backend-Architecture-MOC]] — Backend паттерны

---

## 📝 Заметки

> [!TIP] Архитектурные принципы
>
> 1. **Single Responsibility** — один слой = одна ответственность
> 2. **Dependency Injection** — внедряем зависимости, не создаем
> 3. **SOLID** — применяем принципы к каждому слою
> 4. **CQRS (опционально)** — разделение команд и запросов
> 5. **Repository Pattern** — абстракция над БД

> [!INFO] Сравнение языков
>
> | Язык | Контроллеры | Сервисы | ORM |
> |------|-------------|---------|-----|
> | **TypeScript** | NestJS Controllers | NestJS Services | TypeORM, Prisma |
> | **Go** | Gin/Echo Handlers | Service structs | GORM, sqlx |
> | **PHP** | Laravel Controllers | Laravel Services | Eloquent |
> | **Rust** | Actix/Axum Handlers | Service structs | Diesel, sqlx |

> [!CHECK] Чеклист перед коммитом
>
> - [ ] Контроллер не содержит бизнес-логики
> - [ ] Сервис не знает про HTTP
> - [ ] DTO используются для всех входящих данных
> - [ ] Guards для авторизации
> - [ ] Обработка ошибок (Exception Filters)
> - [ ] Логирование в сервисах
> - [ ] Чувствительные данные скрыты
> - [ ] N+1 проблема решена (DataLoader)
> - [ ] Кэш инвалидируется при изменениях
