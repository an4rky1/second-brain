---
created: 2026-02-17
tags:
  - nestjs
  - events
  - event-emitter
  - event-bus
aliases:
  - NestJS Events
  - NestJS Event Emitter
related:
  - NestJS-MOC
  - NestJS-CQRS
  - NestJS-Microservices
---

# NestJS — Events

> [!SUMMARY] Обзор
> Event-driven архитектура в NestJS: Event Emitter, EventBus, event handlers, sagas.

---

## 📦 Event Emitter Module

### Установка

```bash
npm install @nestjs/event-emitter
```

### Setup

```typescript
// app.module.ts
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot({
      wildcard: true,           // Enable wildcard listeners
      delimiter: '.',           // Event delimiter
      verboseMemoryLeak: true,  // Warn on memory leaks
      ignoreErrors: false,      // Don't ignore errors
    }),
  ],
})
export class AppModule {}
```

---

## 📬 Emitting Events

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class UsersService {
  constructor(private eventEmitter: EventEmitter2) {}

  async create(createUserDto: CreateUserDto) {
    const user = await this.prisma.user.create({
      data: createUserDto,
    });

    // Emit event
    this.eventEmitter.emit('user.created', {
      userId: user.id,
      email: user.email,
      name: user.name,
    });

    // Emit with wildcard
    this.eventEmitter.emit('users.*.created', {
      userId: user.id,
    });

    return user;
  }

  async update(id: number, updateUserDto: UpdateUserDto) {
    const user = await this.prisma.user.update({
      where: { id },
      data: updateUserDto,
    });

    // Emit event
    this.eventEmitter.emit('user.updated', {
      userId: user.id,
      changes: updateUserDto,
    });

    return user;
  }

  async delete(id: number) {
    await this.prisma.user.delete({ where: { id } });

    // Emit event
    this.eventEmitter.emit('user.deleted', {
      userId: id,
    });
  }
}
```

---

## 🎯 Event Listeners

### Basic Listener

```typescript
// users/listeners/user-created.listener.ts
import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { MailService } from '../../mail/mail.service';

@Injectable()
export class UserCreatedListener {
  constructor(private mailService: MailService) {}

  @OnEvent('user.created')
  async handleUserCreated(payload: any) {
    console.log('User created:', payload);
    
    // Send welcome email
    await this.mailService.sendWelcomeEmail(payload.email, payload.name);
  }
}
```

### Wildcard Listener

```typescript
// users/listeners/user-events.listener.ts
import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';

@Injectable()
export class UserEventsListener {
  @OnEvent('user.*')
  async handleUserEvents(payload: any, event: string) {
    console.log(`User event ${event}:`, payload);
    
    // Log all user events
    await this.logEvent(event, payload);
  }

  @OnEvent('users.*.created')
  async handleUserCreated(payload: any) {
    console.log('User created in any context:', payload);
  }

  private async logEvent(event: string, payload: any) {
    // Save to audit log
    console.log('Audit log:', { event, payload, timestamp: new Date() });
  }
}
```

### Multiple Listeners for Same Event

```typescript
// mail/listeners/user-events.listener.ts
@Injectable()
export class MailUserListener {
  constructor(private mailService: MailService) {}

  @OnEvent('user.created')
  async sendWelcomeEmail(payload: any) {
    await this.mailService.sendWelcomeEmail(payload.email, payload.name);
  }

  @OnEvent('user.updated')
  async sendProfileUpdateEmail(payload: any) {
    if (payload.changes.email) {
      await this.mailService.sendEmailChangeNotification(payload.userId, payload.changes.email);
    }
  }
}

// notifications/listeners/user-events.listener.ts
@Injectable()
export class NotificationsUserListener {
  constructor(private notificationsService: NotificationsService) {}

  @OnEvent('user.created')
  async sendWelcomeNotification(payload: any) {
    await this.notificationsService.sendPushNotification(payload.userId, 'Welcome!');
  }

  @OnEvent('user.deleted')
  async cleanupUserNotifications(payload: any) {
    await this.notificationsService.deleteUserNotifications(payload.userId);
  }
}
```

---

## 🔄 CQRS EventBus

### Setup

```bash
npm install @nestjs/cqrs
```

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  imports: [CqrsModule],
  // ...
})
export class UsersModule {}
```

### Events

```typescript
// users/events/user-created.event.ts
export class UserCreatedEvent {
  constructor(
    public readonly userId: number,
    public readonly email: string,
    public readonly name: string,
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
```

### Publishing Events

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import { EventBus } from '@nestjs/cqrs';
import { UserCreatedEvent } from './events/user-created.event';
import { UserUpdatedEvent } from './events/user-updated.event';

@Injectable()
export class UsersService {
  constructor(private eventBus: EventBus) {}

  async create(createUserDto: CreateUserDto) {
    const user = await this.prisma.user.create({
      data: createUserDto,
    });

    // Publish event
    this.eventBus.publish(new UserCreatedEvent(user.id, user.email, user.name));

    return user;
  }

  async update(id: number, updateUserDto: UpdateUserDto) {
    const user = await this.prisma.user.update({
      where: { id },
      data: updateUserDto,
    });

    // Publish event
    this.eventBus.publish(new UserUpdatedEvent(user.id, updateUserDto));

    return user;
  }
}
```

### Event Handlers

```typescript
// users/handlers/user-created.handler.ts
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { UserCreatedEvent } from '../events/user-created.event';
import { MailService } from '../../mail/mail.service';

@EventsHandler(UserCreatedEvent)
export class UserCreatedHandler implements IEventHandler<UserCreatedEvent> {
  constructor(private mailService: MailService) {}

  async handle(event: UserCreatedEvent) {
    console.log('UserCreatedEvent:', event);
    
    await this.mailService.sendWelcomeEmail(event.email, event.name);
  }
}

// users/handlers/user-updated.handler.ts
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { UserUpdatedEvent } from '../events/user-updated.event';

@EventsHandler(UserUpdatedEvent)
export class UserUpdatedHandler implements IEventHandler<UserUpdatedEvent> {
  async handle(event: UserUpdatedEvent) {
    console.log('UserUpdatedEvent:', event);
    
    // Log changes for audit
    await this.auditLog(event.userId, event.changes);
  }

  private async auditLog(userId: number, changes: any) {
    // Save to audit log
  }
}

// users/handlers/index.ts
export * from './user-created.handler';
export * from './user-updated.handler';
```

### Register Handlers

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';
import { UserCreatedHandler, UserUpdatedHandler } from './handlers';

@Module({
  imports: [CqrsModule],
  providers: [
    UsersService,
    UserCreatedHandler,
    UserUpdatedHandler,
  ],
})
export class UsersModule {}
```

---

## 🎯 Event Sourcing Pattern

```typescript
// users/entities/user.aggregate.ts
import { AggregateRoot } from '@nestjs/cqrs';

export class User extends AggregateRoot {
  id: number;
  email: string;
  name: string;
  private changes: any[] = [];

  apply(event: any) {
    this.changes.push(event);
    
    switch (event.constructor.name) {
      case 'UserCreatedEvent':
        this.id = event.userId;
        this.email = event.email;
        this.name = event.name;
        break;
      case 'UserUpdatedEvent':
        Object.assign(this, event.changes);
        break;
    }
  }

  getUncommittedChanges() {
    return this.changes;
  }

  commit() {
    this.changes.forEach((event) => this.$emit(event));
    this.changes = [];
  }

  loadFromHistory(history: any[]) {
    this.changes = [];
    history.forEach((event) => this.apply(event));
  }
}
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-CQRS]] — CQRS pattern
- [[NestJS-Microservices]] — Микросервисы
