---
created: 2026-02-17
tags:
  - architecture
  - clean-architecture
  - hexagonal
  - patterns
aliases:
  - Clean Architecture
  - Hexagonal Architecture
related:
  - MOC-Patterns
  - Backend-Architecture-MOC
  - Microservices-Patterns
---

# Clean & Hexagonal Architecture

> [!SUMMARY] Обзор
> Clean Architecture (Robert Martin) и Hexagonal (Alistair Cockburn) — паттерны для создания поддерживаемых, тестируемых приложений.

---

## 🏗️ Clean Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Frameworks & Drivers                  │
│  (Express, NestJS, React, Database, External APIs)      │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                 Interface Adapters                       │
│  (Controllers, Presenters, Gateways, Repositories)      │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                 Use Cases (Application)                  │
│  (Business rules, Application logic)                    │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                 Entities (Domain)                        │
│  (Enterprise business rules, Domain models)             │
└─────────────────────────────────────────────────────────┘
```

### Принципы

1. **Зависимости направлены внутрь** — внешние слои зависят от внутренних
2. **Изоляция бизнес-логики** — use cases не зависят от фреймворков
3. **Testability** — бизнес-логика тестируется без UI/DB
4. **Framework Independence** — можно заменить фреймворк

---

## 📁 Структура проекта

```
src/
├── domain/                 # Domain Layer (Entities)
│   ├── entities/
│   │   ├── user.entity.ts
│   │   └── order.entity.ts
│   ├── value-objects/
│   │   ├── email.vo.ts
│   │   └── money.vo.ts
│   ├── repositories/
│   │   └── user.repository.ts  # Interface
│   └── services/
│       └── domain.service.ts
│
├── application/            # Application Layer (Use Cases)
│   ├── use-cases/
│   │   ├── create-user.use-case.ts
│   │   ├── get-user.use-case.ts
│   │   └── update-user.use-case.ts
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── user-response.dto.ts
│   └── interfaces/
│       └── user.service.interface.ts
│
├── infrastructure/         # Infrastructure Layer
│   ├── database/
│   │   ├── prisma/
│   │   │   ├── prisma.service.ts
│   │   │   └── user.repository.impl.ts
│   │   └── typeorm/
│   ├── http/
│   │   ├── axios/
│   │   └── fetch/
│   └── messaging/
│       ├── rabbitmq/
│       └── kafka/
│
├── interface-adapters/     # Interface Adapters Layer
│   ├── controllers/
│   │   ├── users.controller.ts
│   │   └── orders.controller.ts
│   ├── presenters/
│   │   └── user.presenter.ts
│   └── middleware/
│       ├── auth.middleware.ts
│       └── logging.middleware.ts
│
└── frameworks/             # Frameworks & Drivers Layer
    ├── nestjs/
    │   ├── app.module.ts
    │   └── main.ts
    ├── express/
    └── cli/
```

---

## 🎯 Пример реализации

### Domain Layer

```typescript
// domain/entities/user.entity.ts
export class User {
  constructor(
    public readonly id: string,
    public email: EmailValueObject,
    public name: string,
    public role: UserRole,
    public createdAt: Date,
    public updatedAt: Date,
  ) {}

  updateEmail(email: string): void {
    this.email = new EmailValueObject(email);
    this.updatedAt = new Date();
  }

  updateName(name: string): void {
    this.name = name;
    this.updatedAt = new Date();
  }

  hasRole(role: UserRole): boolean {
    return this.role === role;
  }
}

// domain/value-objects/email.vo.ts
export class EmailValueObject {
  constructor(public readonly value: string) {
    if (!this.isValid(value)) {
      throw new InvalidEmailError(value);
    }
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

// domain/repositories/user.repository.ts
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(user: User): Promise<User>;
  update(user: User): Promise<User>;
  delete(id: string): Promise<void>;
}

// domain/services/domain.service.ts
export interface IEmailService {
  sendWelcomeEmail(email: string): Promise<void>;
  sendPasswordReset(email: string, token: string): Promise<void>;
}
```

### Application Layer

```typescript
// application/use-cases/create-user.use-case.ts
import { IUserRepository } from '../../domain/repositories/user.repository';
import { IEmailService } from '../../domain/services/domain.service';
import { User } from '../../domain/entities/user.entity';
import { EmailValueObject } from '../../domain/value-objects/email.vo';
import { CreateUserDto } from '../dto/create-user.dto';

export class CreateUserUseCase {
  constructor(
    private userRepository: IUserRepository,
    private emailService: IEmailService,
  ) {}

  async execute(dto: CreateUserDto): Promise<User> {
    // Check for existing user
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) {
      throw new UserAlreadyExistsError(dto.email);
    }

    // Create user entity
    const user = new User(
      crypto.randomUUID(),
      new EmailValueObject(dto.email),
      dto.name,
      UserRole.USER,
      new Date(),
      new Date(),
    );

    // Save user
    const saved = await this.userRepository.create(user);

    // Send welcome email (side effect)
    await this.emailService.sendWelcomeEmail(saved.email.value);

    return saved;
  }
}

// application/use-cases/get-user.use-case.ts
export class GetUserUseCase {
  constructor(private userRepository: IUserRepository) {}

  async execute(id: string): Promise<User> {
    const user = await this.userRepository.findById(id);
    
    if (!user) {
      throw new UserNotFoundError(id);
    }
    
    return user;
  }
}
```

### Infrastructure Layer

```typescript
// infrastructure/database/prisma/user.repository.impl.ts
import { Injectable } from '@nestjs/common';
import { IUserRepository } from '../../../domain/repositories/user.repository';
import { User } from '../../../domain/entities/user.entity';
import { PrismaService } from './prisma.service';

@Injectable()
export class PrismaUserRepository implements IUserRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { id } });
    return data ? this.toDomain(data) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { email } });
    return data ? this.toDomain(data) : null;
  }

  async create(user: User): Promise<User> {
    const data = await this.prisma.user.create({
      data: this.toPersistence(user),
    });
    return this.toDomain(data);
  }

  async update(user: User): Promise<User> {
    const data = await this.prisma.user.update({
      where: { id: user.id },
      data: this.toPersistence(user),
    });
    return this.toDomain(data);
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }

  private toDomain(data: any): User {
    return new User(
      data.id,
      new EmailValueObject(data.email),
      data.name,
      data.role,
      data.createdAt,
      data.updatedAt,
    );
  }

  private toPersistence(user: User): any {
    return {
      id: user.id,
      email: user.email.value,
      name: user.name,
      role: user.role,
    };
  }
}
```

### Interface Adapters Layer

```typescript
// interface-adapters/controllers/users.controller.ts
import { Controller, Post, Get, Body, Param, HttpStatus } from '@nestjs/common';
import { CreateUserUseCase } from '../../application/use-cases/create-user.use-case';
import { GetUserUseCase } from '../../application/use-cases/get-user.use-case';
import { CreateUserDto } from '../../application/dto/create-user.dto';
import { UserPresenter } from '../presenters/user.presenter';

@Controller('users')
export class UsersController {
  constructor(
    private createUserUseCase: CreateUserUseCase,
    private getUserUseCase: GetUserUseCase,
    private presenter: UserPresenter,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateUserDto) {
    const user = await this.createUserUseCase.execute(dto);
    return this.presenter.present(user);
  }

  @Get(':id')
  async getOne(@Param('id') id: string) {
    const user = await this.getUserUseCase.execute(id);
    return this.presenter.present(user);
  }
}

// interface-adapters/presenters/user.presenter.ts
export class UserPresenter {
  present(user: User): UserResponseDto {
    return {
      id: user.id,
      email: user.email.value,
      name: user.name,
      role: user.role,
      createdAt: user.createdAt,
    };
  }
}
```

---

## 🔄 Dependency Injection

```typescript
// frameworks/nestjs/app.module.ts
@Module({
  imports: [
    PrismaModule,  // Infrastructure
    HttpModule,    // Infrastructure
  ],
  providers: [
    // Domain
    { provide: 'IUserRepository', useClass: PrismaUserRepository },
    { provide: 'IEmailService', useClass: SmtpEmailService },
    
    // Application
    CreateUserUseCase,
    GetUserUseCase,
    UpdateUserUseCase,
    DeleteUserUseCase,
    
    // Interface Adapters
    UserPresenter,
    UsersController,
  ],
  controllers: [UsersController],
})
export class UsersModule {}
```

---

## ✅ Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Testability** | Бизнес-логика тестируется без DB/UI |
| **Maintainability** | Чёткое разделение ответственности |
| **Flexibility** | Легко заменить фреймворк/БД |
| **Scalability** | Легко добавлять новые use cases |
| **Domain-Driven** | Фокус на бизнес-логике |

---

## 🔗 Связанные заметки

- [[MOC-Patterns]] — Patterns index
- [[Backend-Architecture-MOC]] — Backend architecture
- [[Microservices-Patterns]] — Microservices
- [[Hexagonal-Architecture]] — Hexagonal architecture
