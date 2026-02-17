---
created: 2026-02-17
tags:
  - nestjs
  - modules
  - providers
  - dependency-injection
aliases:
  - NestJS Modules
  - NestJS DI
related:
  - NestJS-MOC
  - NestJS-Controllers
  - NestJS-Services
---

# NestJS — Modules & Dependency Injection

> [!SUMMARY] Обзор
> Модули — базовая единица организации кода в NestJS. Dependency injection, провайдеры, exports.

---

## 📦 Module Decorator

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { PrismaModule } from '../prisma/prisma.module';

@Module({
  imports: [PrismaModule],        // Другие модули
  controllers: [UsersController], // Контроллеры этого модуля
  providers: [UsersService],      // Сервисы (DI)
  exports: [UsersService],        // Что доступно другим модулям
})
export class UsersModule {}
```

---

## 🔧 Provider Types

### Class Provider

```typescript
// users/users.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}
  
  async findAll() {
    return this.prisma.user.findMany();
  }
}

// users/users.module.ts
@Module({
  providers: [UsersService],
})
export class UsersModule {}
```

### Value Provider

```typescript
// config/config.module.ts
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({
  providers: [
    {
      provide: 'CONFIG_OPTIONS',
      useValue: {
        apiKey: process.env.API_KEY,
        apiUrl: process.env.API_URL,
      },
    },
    ConfigService,
  ],
  exports: [ConfigService],
})
export class ConfigModule {}

// config/config.service.ts
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class ConfigService {
  constructor(
    @Inject('CONFIG_OPTIONS') private configOptions: any,
  ) {}

  get(key: string): any {
    return this.configOptions[key];
  }
}
```

### Factory Provider

```typescript
// database/database.module.ts
import { Module } from '@nestjs/common';
import { DatabaseService } from './database.service';
import { ConfigService } from '../config/config.service';

@Module({
  providers: [
    {
      provide: 'DATABASE_CONNECTION',
      useFactory: async (config: ConfigService) => {
        const host = config.get('DB_HOST');
        const port = config.get('DB_PORT');
        const user = config.get('DB_USER');
        const password = config.get('DB_PASSWORD');
        
        // Create connection
        const connection = await createConnection({
          host, port, user, password,
        });
        
        return connection;
      },
      inject: [ConfigService],
    },
    DatabaseService,
  ],
  exports: ['DATABASE_CONNECTION'],
})
export class DatabaseModule {}
```

### Existing Provider (Alias)

```typescript
// users/users.module.ts
@Module({
  providers: [
    UsersService,
    {
      provide: 'UserService',  // Alias
      useExisting: UsersService,
    },
  ],
})
export class UsersModule {}
```

---

## 🌍 Module Scopes

### Default (Singleton)

```typescript
@Injectable()  // Singleton по умолчанию
export class UsersService {
  private counter = 0;
  
  increment() {
    return ++this.counter;
  }
}
```

### Request Scoped

```typescript
@Injectable({ scope: Scope.REQUEST })
export class RequestLoggerService {
  private logs: string[] = [];
  
  log(message: string) {
    this.logs.push(message);
  }
  
  getLogs() {
    return this.logs;
  }
}

// Controller
@Controller('users')
export class UsersController {
  constructor(
    @Inject(REQUEST) private request: Request,
    private logger: RequestLoggerService,
  ) {}
  
  @Get()
  findAll() {
    this.logger.log('Fetching users');
    return this.usersService.findAll();
  }
}

// Module
@Module({
  providers: [
    {
      provide: RequestLoggerService,
      useClass: RequestLoggerService,
      scope: Scope.REQUEST,
    },
  ],
})
export class UsersModule {}
```

### Transient

```typescript
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {
  constructor() {
    console.log('New instance created');
  }
}
```

---

## 🔄 Dynamic Modules

```typescript
// config/config.module.ts
import { Module, DynamicModule, Global } from '@nestjs/common';
import { ConfigService } from './config.service';

export interface ConfigModuleOptions {
  folder: string;
  isGlobal?: boolean;
  cache?: boolean;
}

@Global()
@Module({})
export class ConfigModule {
  static forRoot(options: ConfigModuleOptions): DynamicModule {
    return {
      module: ConfigModule,
      global: options.isGlobal,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }

  static forRootAsync(options: {
    useFactory: (...args: any[]) => Promise<ConfigModuleOptions> | ConfigModuleOptions;
    inject?: any[];
    isGlobal?: boolean;
  }): DynamicModule {
    return {
      module: ConfigModule,
      global: options.isGlobal,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useFactory: options.useFactory,
          inject: options.inject,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}

// Usage in AppModule
@Module({
  imports: [
    ConfigModule.forRoot({
      folder: './config',
      isGlobal: true,
      cache: true,
    }),
  ],
})
export class AppModule {}
```

---

## 📁 Module Structure

```
src/
├── app.module.ts           # Root module
├── main.ts                 # Entry point
├── common/                 # Shared modules
│   ├── common.module.ts
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/                 # Config module
│   ├── config.module.ts
│   └── config.service.ts
├── database/               # Database module
│   ├── database.module.ts
│   └── database.service.ts
└── users/                  # Feature module
    ├── users.module.ts
    ├── users.controller.ts
    ├── users.service.ts
    └── dto/
        ├── create-user.dto.ts
        └── update-user.dto.ts
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Controllers]] — Контроллеры
- [[NestJS-Services]] — Сервисы
