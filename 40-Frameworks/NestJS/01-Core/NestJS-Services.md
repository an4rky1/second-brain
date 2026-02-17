---
created: 2026-02-17
tags:
  - nestjs
  - services
  - business-logic
  - providers
aliases:
  - NestJS Services
  - NestJS Business Logic
related:
  - NestJS-MOC
  - NestJS-Modules
  - NestJS-Controllers
---

# NestJS — Services

> [!SUMMARY] Обзор
> Сервисы содержат бизнес-логику приложения. Dependency injection, transaction management, error handling.

---

## 🔧 Basic Service

```typescript
// users/users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
  Logger,
} from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  constructor(private prisma: PrismaService) {}

  // ─────────────────────────────────────────────────────
  // CREATE
  // ─────────────────────────────────────────────────────
  async create(createUserDto: CreateUserDto) {
    // Проверка на дубликат
    const existing = await this.prisma.user.findUnique({
      where: { email: createUserDto.email },
    });

    if (existing) {
      throw new ConflictException('Email already exists');
    }

    // Хэширование пароля
    const hashedPassword = await this.hashPassword(createUserDto.password);

    // Создание
    const user = await this.prisma.user.create({
      data: {
        ...createUserDto,
        password: hashedPassword,
      },
    });

    this.logger.log(`User created: ${user.id}`);
    
    // Исключаем пароль из ответа
    const { password, ...result } = user;
    return result;
  }

  // ─────────────────────────────────────────────────────
  // FIND ALL с пагинацией
  // ─────────────────────────────────────────────────────
  async findAll(pagination: { page: number; limit: number; search?: string }) {
    const { page, limit, search } = pagination;
    const skip = (page - 1) * limit;

    const [data, total] = await Promise.all([
      this.prisma.user.findMany({
        skip,
        take: limit,
        where: search ? {
          OR: [
            { name: { contains: search } },
            { email: { contains: search } },
          ],
        } : {},
        orderBy: { createdAt: 'DESC' },
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          createdAt: true,
        },
      }),
      this.prisma.user.count(),
    ]);

    return {
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
  }

  // ─────────────────────────────────────────────────────
  // FIND ONE
  // ─────────────────────────────────────────────────────
  async findOne(id: number) {
    const user = await this.prisma.user.findUnique({
      where: { id },
      select: {
        id: true,
        email: true,
        name: true,
        role: true,
        createdAt: true,
        updatedAt: true,
      },
    });

    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }

    return user;
  }

  // ─────────────────────────────────────────────────────
  // UPDATE
  // ─────────────────────────────────────────────────────
  async update(id: number, updateUserDto: UpdateUserDto) {
    // Проверка существования
    await this.findOne(id);

    // Проверка email на уникальность
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

  // ─────────────────────────────────────────────────────
  // DELETE
  // ─────────────────────────────────────────────────────
  async remove(id: number) {
    await this.findOne(id);
    await this.prisma.user.delete({
      where: { id },
    });
  }

  // ─────────────────────────────────────────────────────
  // HELPER
  // ─────────────────────────────────────────────────────
  private async hashPassword(password: string): Promise<string> {
    const bcrypt = require('bcrypt');
    return bcrypt.hash(password, 10);
  }
}
```

---

## 🔄 Transaction Management

```typescript
// users/users.service.ts
@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  // Transaction with Prisma
  async transferCredits(fromId: number, toId: number, amount: number) {
    return this.prisma.$transaction(async (tx) => {
      // Lock users for update
      const [fromUser, toUser] = await Promise.all([
        tx.user.findUnique({ 
          where: { id: fromId },
          lock: { mode: 'pessimistic_write' },
        }),
        tx.user.findUnique({ 
          where: { id: toId },
          lock: { mode: 'pessimistic_write' },
        }),
      ]);

      if (!fromUser || !toUser) {
        throw new NotFoundException('User not found');
      }

      if (fromUser.credits < amount) {
        throw new ConflictException('Insufficient credits');
      }

      // Update with optimistic locking
      await Promise.all([
        tx.user.update({
          where: { id: fromId },
          data: { credits: { decrement: amount } },
        }),
        tx.user.update({
          where: { id: toId },
          data: { credits: { increment: amount } },
        }),
      ]);

      return { success: true };
    });
  }

  // Bulk operations with transaction
  async bulkCreate(users: CreateUserDto[]) {
    return this.prisma.$transaction(async (tx) => {
      const created = [];
      
      for (const userData of users) {
        const existing = await tx.user.findUnique({
          where: { email: userData.email },
        });

        if (!existing) {
          const user = await tx.user.create({
            data: {
              ...userData,
              password: await this.hashPassword(userData.password),
            },
          });
          created.push(user);
        }
      }

      return created;
    });
  }
}
```

---

## 📦 Service with Repository Pattern

```typescript
// users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { User } from '@prisma/client';

@Injectable()
export class UsersRepository {
  constructor(private prisma: PrismaService) {}

  async findAll(skip: number, take: number): Promise<User[]> {
    return this.prisma.user.findMany({ skip, take });
  }

  async findById(id: number): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(data: any): Promise<User> {
    return this.prisma.user.create({ data });
  }

  async update(id: number, data: any): Promise<User> {
    return this.prisma.user.update({ where: { id }, data });
  }

  async delete(id: number): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }
}

// users/users.service.ts
@Injectable()
export class UsersService {
  constructor(
    private usersRepository: UsersRepository,
    private logger: LoggerService,
  ) {}

  async findAll(page: number, limit: number) {
    const skip = (page - 1) * limit;
    const [data, total] = await Promise.all([
      this.usersRepository.findAll(skip, limit),
      this.usersRepository.findById(1).then(() => 
        this.usersRepository.findAll(0, 0).then(u => u.length)
      ),
    ]);

    return { data, meta: { total, page, limit } };
  }

  async findOne(id: number) {
    const user = await this.usersRepository.findById(id);
    
    if (!user) {
      throw new NotFoundException('User not found');
    }
    
    return user;
  }
}
```

---

## 🎯 Service Best Practices

### ✅ Делать

```typescript
@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  constructor(
    private prisma: PrismaService,
    private mailService: MailService,  // DI других сервисов
  ) {}

  async create(dto: CreateUserDto) {
    try {
      // Валидация бизнес-правил
      await this.validateBusinessRules(dto);

      // Создание
      const user = await this.prisma.user.create({ data: dto });

      // Асинхронные побочные эффекты (не блокируем ответ)
      this.mailService.sendWelcomeEmail(user.email).catch(err => {
        this.logger.error('Failed to send welcome email', err);
      });

      return user;
    } catch (error) {
      this.logger.error('Failed to create user', error);
      throw error;  // Пробрасываем ошибку дальше
    }
  }

  private async validateBusinessRules(dto: CreateUserDto) {
    // Бизнес-валидация
    if (dto.age && dto.age < 18) {
      throw new BadRequestException('Must be at least 18');
    }
  }
}
```

### ❌ Не делать

```typescript
// ❌ Бизнес-логика в контроллере
@Controller('users')
export class UsersController {
  @Post()
  async create(@Body() dto: CreateUserDto) {
    // ❌ Логика в контроллере
    const existing = await this.prisma.user.findUnique({
      where: { email: dto.email },
    });
    
    if (existing) {
      throw new ConflictException('Email exists');
    }
    
    return this.prisma.user.create({ data: dto });
  }
}

// ✅ Правильно: логика в сервисе
@Injectable()
export class UsersService {
  async create(dto: CreateUserDto) {
    const existing = await this.prisma.user.findUnique({
      where: { email: dto.email },
    });
    
    if (existing) {
      throw new ConflictException('Email exists');
    }
    
    return this.prisma.user.create({ data: dto });
  }
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Modules]] — Модули
- [[NestJS-Controllers]] — Контроллеры
- [[NestJS-Prisma]] — Prisma integration
