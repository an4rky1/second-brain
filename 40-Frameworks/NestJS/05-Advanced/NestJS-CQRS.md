---
created: 2026-02-17
tags:
  - nestjs
  - cqrs
  - architecture
  - patterns
aliases:
  - NestJS CQRS
  - Command Query Responsibility Segregation
related:
  - NestJS-MOC
  - NestJS-Services
  - NestJS-Events
---

# NestJS — CQRS Pattern

> [!SUMMARY] Обзор
> CQRS (Command Query Responsibility Segregation) — разделение операций чтения и записи. Команды, запросы, handlers, events.

---

## 📦 Установка

```bash
npm install @nestjs/cqrs
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Command Side                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Command  │  │  Handler │  │  Event   │          │
│  │          │→ │          │→ │ Publisher│          │
│  └──────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                   Write Database                     │
│                   (PostgreSQL)                       │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                     Query Side                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  Query   │  │  Handler │  │  Read    │          │
│  │          │→ │          │→ │  Model   │          │
│  └──────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                    Read Database                     │
│                  (MongoDB / Redis)                   │
└─────────────────────────────────────────────────────┘
```

---

## 📝 Commands

```typescript
// users/commands/create-user.command.ts
export class CreateUserCommand {
  constructor(
    public readonly email: string,
    public readonly password: string,
    public readonly name: string,
  ) {}
}

// users/commands/update-user.command.ts
export class UpdateUserCommand {
  constructor(
    public readonly id: number,
    public readonly email?: string,
    public readonly name?: string,
  ) {}
}

// users/commands/delete-user.command.ts
export class DeleteUserCommand {
  constructor(public readonly id: number) {}
}

// users/commands/index.ts
export * from './create-user.command';
export * from './update-user.command';
export * from './delete-user.command';
```

---

## 🔍 Queries

```typescript
// users/queries/get-users.query.ts
export class GetUsersQuery {
  constructor(
    public readonly page: number = 1,
    public readonly limit: number = 10,
    public readonly search?: string,
  ) {}
}

// users/queries/get-user-by-id.query.ts
export class GetUserByIdQuery {
  constructor(public readonly id: number) {}
}

// users/queries/index.ts
export * from './get-users.query';
export * from './get-user-by-id.query';
```

---

## 🛠️ Command Handlers

```typescript
// users/handlers/create-user.handler.ts
import { CommandHandler, EventPublisher, ICommandHandler } from '@nestjs/cqrs';
import { CreateUserCommand } from '../commands';
import { UserRepository } from '../users.repository';
import { UserCreatedEvent } from '../events';
import { ConflictException } from '@nestjs/common';

@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(
    private repository: UserRepository,
    private publisher: EventPublisher,
  ) {}

  async execute(command: CreateUserCommand): Promise<any> {
    const { email, password, name } = command;

    // Check for existing user
    const existing = await this.repository.findByEmail(email);
    if (existing) {
      throw new ConflictException('Email already exists');
    }

    // Create user
    const user = await this.repository.create({
      email,
      password,
      name,
    });

    // Publish event
    this.publisher.mergeObjectContext(user);
    user.apply(new UserCreatedEvent(user.id, user.email));
    user.commit();

    return user;
  }
}

// users/handlers/update-user.handler.ts
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { UpdateUserCommand } from '../commands';
import { UserRepository } from '../users.repository';
import { NotFoundException } from '@nestjs/common';

@CommandHandler(UpdateUserCommand)
export class UpdateUserHandler implements ICommandHandler<UpdateUserCommand> {
  constructor(private repository: UserRepository) {}

  async execute(command: UpdateUserCommand): Promise<any> {
    const { id, email, name } = command;

    const user = await this.repository.findById(id);
    if (!user) {
      throw new NotFoundException('User not found');
    }

    // Check email uniqueness
    if (email) {
      const existing = await this.repository.findByEmail(email);
      if (existing && existing.id !== id) {
        throw new ConflictException('Email already exists');
      }
    }

    return this.repository.update(id, { email, name });
  }
}

// users/handlers/delete-user.handler.ts
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { DeleteUserCommand } from '../commands';
import { UserRepository } from '../users.repository';

@CommandHandler(DeleteUserCommand)
export class DeleteUserHandler implements ICommandHandler<DeleteUserCommand> {
  constructor(private repository: UserRepository) {}

  async execute(command: DeleteUserCommand): Promise<void> {
    await this.repository.delete(command.id);
  }
}

// users/handlers/index.ts
export * from './create-user.handler';
export * from './update-user.handler';
export * from './delete-user.handler';
```

---

## 🔎 Query Handlers

```typescript
// users/handlers/get-users.handler.ts
import { IQueryHandler, QueryHandler } from '@nestjs/cqrs';
import { GetUsersQuery } from '../queries';
import { UserRepository } from '../users.repository';

@QueryHandler(GetUsersQuery)
export class GetUsersHandler implements IQueryHandler<GetUsersQuery> {
  constructor(private repository: UserRepository) {}

  async execute(query: GetUsersQuery): Promise<any> {
    const { page, limit, search } = query;
    const skip = (page - 1) * limit;

    const [data, total] = await Promise.all([
      this.repository.findAll({ skip, take: limit, search }),
      this.repository.count({ search }),
    ]);

    return {
      data,
      meta: {
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
      },
    };
  }
}

// users/handlers/get-user-by-id.handler.ts
import { IQueryHandler, QueryHandler } from '@nestjs/cqrs';
import { GetUserByIdQuery } from '../queries';
import { UserRepository } from '../users.repository';
import { NotFoundException } from '@nestjs/common';

@QueryHandler(GetUserByIdQuery)
export class GetUserByIdHandler implements IQueryHandler<GetUserByIdQuery> {
  constructor(private repository: UserRepository) {}

  async execute(query: GetUserByIdQuery): Promise<any> {
    const user = await this.repository.findById(query.id);

    if (!user) {
      throw new NotFoundException('User not found');
    }

    return user;
  }
}

// users/handlers/index.ts
export * from './get-users.handler';
export * from './get-user-by-id.handler';
```

---

## 📬 Events

```typescript
// users/events/user-created.event.ts
export class UserCreatedEvent {
  constructor(
    public readonly userId: number,
    public readonly email: string,
  ) {}
}

// users/events/user-updated.event.ts
export class UserUpdatedEvent {
  constructor(
    public readonly userId: number,
    public readonly changes: Partial<any>,
  ) {}
}

// users/events/user-deleted.event.ts
export class UserDeletedEvent {
  constructor(public readonly userId: number) {}
}

// users/events/index.ts
export * from './user-created.event';
export * from './user-updated.event';
export * from './user-deleted.event';
```

---

## 📥 Event Handlers

```typescript
// users/sagas/user-created.saga.ts
import { Injectable } from '@nestjs/common';
import { ICommand, ofType, Saga } from '@nestjs/cqrs';
import { Observable, of } from 'rxjs';
import { UserCreatedEvent } from '../events';
import { SendWelcomeEmailCommand } from '../../mail/commands';

@Injectable()
export class UserSagas {
  @Saga()
  userCreated = (events$: Observable<UserCreatedEvent>): Observable<ICommand> => {
    return events$.pipe(
      ofType(UserCreatedEvent),
      map((event) => {
        return new SendWelcomeEmailCommand(event.email);
      }),
    );
  };
}

// mail/handlers/send-welcome-email.handler.ts
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { SendWelcomeEmailCommand } from '../commands';
import { MailService } from '../mail.service';

@CommandHandler(SendWelcomeEmailCommand)
export class SendWelcomeEmailHandler implements ICommandHandler {
  constructor(private mailService: MailService) {}

  async execute(command: SendWelcomeEmailCommand): Promise<void> {
    await this.mailService.sendWelcomeEmail(command.email);
  }
}
```

---

## 🏛️ Module Setup

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserRepository } from './users.repository';

// Import handlers
import {
  CreateUserHandler,
  UpdateUserHandler,
  DeleteUserHandler,
  GetUsersHandler,
  GetUserByIdHandler,
} from './handlers';

import { UserSagas } from './sagas/user-created.saga';

const commandHandlers = [
  CreateUserHandler,
  UpdateUserHandler,
  DeleteUserHandler,
];

const queryHandlers = [
  GetUsersHandler,
  GetUserByIdHandler,
];

@Module({
  imports: [CqrsModule],
  controllers: [UsersController],
  providers: [
    UsersService,
    UserRepository,
    UserSagas,
    ...commandHandlers,
    ...queryHandlers,
  ],
})
export class UsersModule {}
```

---

## 🎮 Controller with CQRS

```typescript
// users/users.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, Query } from '@nestjs/common';
import { CommandBus, QueryBus } from '@nestjs/cqrs';
import { CreateUserCommand } from './commands/create-user.command';
import { UpdateUserCommand } from './commands/update-user.command';
import { DeleteUserCommand } from './commands/delete-user.command';
import { GetUsersQuery } from './queries/get-users.query';
import { GetUserByIdQuery } from './queries/get-user-by-id.query';

@Controller('users')
export class UsersController {
  constructor(
    private commandBus: CommandBus,
    private queryBus: QueryBus,
  ) {}

  @Post()
  async create(@Body() createUserDto: CreateUserDto) {
    return this.commandBus.execute(
      new CreateUserCommand(
        createUserDto.email,
        createUserDto.password,
        createUserDto.name,
      ),
    );
  }

  @Get()
  async findAll(
    @Query('page') page = 1,
    @Query('limit') limit = 10,
    @Query('search') search?: string,
  ) {
    return this.queryBus.execute(new GetUsersQuery(page, limit, search));
  }

  @Get(':id')
  async findOne(@Param('id') id: number) {
    return this.queryBus.execute(new GetUserByIdQuery(id));
  }

  @Put(':id')
  async update(@Param('id') id: number, @Body() updateUserDto: UpdateUserDto) {
    return this.commandBus.execute(
      new UpdateUserCommand(id, updateUserDto.email, updateUserDto.name),
    );
  }

  @Delete(':id')
  async remove(@Param('id') id: number) {
    return this.commandBus.execute(new DeleteUserCommand(id));
  }
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Events]] — Event emitter
- [[NestJS-Services]] — Сервисы
- [[NestJS-Microservices]] — Микросервисы
