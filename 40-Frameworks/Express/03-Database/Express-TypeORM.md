---
created: 2026-02-17
tags:
  - nestjs
  - typeorm
  - database
  - orm
  - postgresql
aliases:
  - NestJS TypeORM
  - TypeORM Integration
related:
  - NestJS-MOC
  - Database-Prisma
  - PostgreSQL-Cheatsheet
---

# NestJS — TypeORM Integration

> [!SUMMARY] Обзор
> TypeORM — классический ORM для TypeScript/JavaScript. Поддерживает PostgreSQL, MySQL, SQLite, MS SQL Server.

---

## Установка

```bash
npm install @nestjs/typeorm typeorm pg
npm install -D @types/pg
```

---

## Конфигурация

### TypeORM Module

```typescript
// database/database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DB_HOST', 'localhost'),
        port: config.get('DB_PORT', 5432),
        username: config.get('DB_USERNAME'),
        password: config.get('DB_PASSWORD'),
        database: config.get('DB_DATABASE'),
        entities: [__dirname + '/../**/*.entity{.ts,.js}'],
        synchronize: config.get('NODE_ENV') === 'development',
        logging: config.get('NODE_ENV') === 'development',
        extra: {
          max: 20,
          min: 5,
          idleTimeoutMillis: 30000,
        },
      }),
      inject: [ConfigService],
    }),
  ],
})
export class DatabaseModule {}
```

---

## Entity

```typescript
// users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
  OneToMany,
  Index,
} from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  @Index()
  email: string;

  @Column({ nullable: true })
  name: string;

  @Column()
  password: string;

  @Column({ default: 'user' })
  @Index()
  role: string;

  @OneToMany(() => Post, post => post.author)
  posts: Post[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn()
  deletedAt: Date; // Soft delete
}
```

---

## Repository Pattern

```typescript
// users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { DataSource, Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersRepository extends Repository<User> {
  constructor(private dataSource: DataSource) {
    super(User, dataSource.createEntityManager());
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.createQueryBuilder('user')
      .where('user.email = :email', { email })
      .getOne();
  }

  async findActiveUsers(): Promise<User[]> {
    return this.createQueryBuilder('user')
      .where('user.deletedAt IS NULL')
      .orderBy('user.createdAt', 'DESC')
      .getMany();
  }

  async paginate(page: number, limit: number, where?: any) {
    const query = this.createQueryBuilder('user')
      .skip((page - 1) * limit)
      .take(limit)
      .orderBy('user.createdAt', 'DESC');

    if (where) {
      Object.keys(where).forEach(key => {
        query.andWhere(`user.${key} = :${key}`, where);
      });
    }

    const [data, total] = await query.getManyAndCount();

    return {
      data,
      meta: { total, page, limit, totalPages: Math.ceil(total / limit) },
    };
  }
}
```

---

## Service

```typescript
// users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { UsersRepository } from './users.repository';

@Injectable()
export class UsersService {
  constructor(private usersRepository: UsersRepository) {}

  async findAll(page = 1, limit = 10) {
    return this.usersRepository.paginate(page, limit);
  }

  async findOne(id: number) {
    const user = await this.usersRepository.findOne({ where: { id } });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  async create(data: CreateUserDto) {
    const user = this.usersRepository.create(data);
    return this.usersRepository.save(user);
  }

  async update(id: number, data: UpdateUserDto) {
    await this.findOne(id);
    await this.usersRepository.update(id, data);
    return this.findOne(id);
  }

  async remove(id: number) {
    await this.findOne(id);
    await this.usersRepository.softDelete(id); // Soft delete
  }
}
```

---

## Environment

```bash
# .env
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=postgres
DB_DATABASE=mydb
DB_POOL_MAX=20
DB_POOL_MIN=5
```

---

## Scripts

```json
{
  "scripts": {
    "typeorm": "typeorm-ts-node-commonjs",
    
    # Generate migration
    "migration:generate": "npm run typeorm migration:generate -- -d src/database/data-source.ts src/database/migrations/InitialMigration",
    
    # Run migrations
    "migration:run": "npm run typeorm migration:run -- -d src/database/data-source.ts",
    
    # Revert migration
    "migration:revert": "npm run typeorm migration:revert -- -d src/database/data-source.ts",
    
    # Show migrations
    "migration:show": "npm run typeorm migration:show -- -d src/database/data-source.ts",
    
    # Create entity
    "entity:create": "npm run typeorm entity:create -- src/users/entities/User",
    
    # Sync schema (dev only!)
    "schema:sync": "npm run typeorm schema:sync -- -d src/database/data-source.ts",
    
    # Drop schema
    "schema:drop": "npm run typeorm schema:drop -- -d src/database/data-source.ts"
  }
}
```

### Data Source

```typescript
// src/database/data-source.ts
import { DataSource } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import { User } from '../users/entities/user.entity';

const configService = new ConfigService();

export const AppDataSource = new DataSource({
  type: 'postgres',
  host: configService.get('DB_HOST', 'localhost'),
  port: configService.get('DB_PORT', 5432),
  username: configService.get('DB_USERNAME'),
  password: configService.get('DB_PASSWORD'),
  database: configService.get('DB_DATABASE'),
  entities: [User],
  migrations: ['src/database/migrations/*.ts'],
  synchronize: false,  // Never in production!
  logging: configService.get('NODE_ENV') === 'development',
});
```

### Migration Commands

```bash
# Generate migration from entity changes
npm run migration:generate -- InitialMigration

# Run all pending migrations
npm run migration:run

# Revert last migration
npm run migration:revert

# Show migration status
npm run migration:show
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[Database-Prisma]] — Prisma ORM
- [[PostgreSQL-Cheatsheet]] — PostgreSQL
