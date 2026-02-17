---
created: 2026-02-17
tags:
  - nestjs
  - redis
  - cache
  - session
aliases:
  - NestJS Redis
  - Redis Cache Integration
related:
  - NestJS-MOC
  - Redis-Cheatsheet
---

# NestJS — Redis Integration

> [!SUMMARY] Обзор
> Redis для кэширования, сессий, pub/sub. ioredis, cache-manager, RedisModule.

---

## Установка

```bash
npm install redis ioredis @nestjs/cache-manager cache-manager cache-manager-ioredis
npm install -D @types/redis @types/ioredis
```

---

## Redis Module

```typescript
// redis/redis.module.ts
import { Module, Global, DynamicModule } from '@nestjs/common';
import { createClient, RedisClientType } from 'redis';

@Global()
@Module({})
export class RedisModule {
  static forRoot(options: { url?: string; host?: string; port?: number }): DynamicModule {
    const redisProvider = {
      provide: 'REDIS_CLIENT',
      useFactory: async (): Promise<RedisClientType> => {
        const client = createClient(options);
        client.on('error', err => console.error('Redis Error', err));
        await client.connect();
        return client;
      },
    };

    return {
      module: RedisModule,
      providers: [redisProvider],
      exports: [redisProvider],
    };
  }
}
```

---

## Redis Service

```typescript
// redis/redis.service.ts
import { Inject, Injectable, OnModuleDestroy } from '@nestjs/common';
import { RedisClientType } from 'redis';

@Injectable()
export class RedisService implements OnModuleDestroy {
  private readonly DEFAULT_TTL = 3600;

  constructor(@Inject('REDIS_CLIENT') private client: RedisClientType) {}

  // Basic
  async get<T>(key: string): Promise<T | null> {
    const value = await this.client.get(key);
    return value ? JSON.parse(value) : null;
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    await this.client.setEx(key, ttl ?? this.DEFAULT_TTL, JSON.stringify(value));
  }

  async del(key: string): Promise<void> {
    await this.client.del(key);
  }

  // Cache-Aside Pattern
  async cacheAside<T>(
    key: string,
    fetchFn: () => Promise<T>,
    ttl?: number,
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached) return cached;

    const fresh = await fetchFn();
    await this.set(key, fresh, ttl);
    return fresh;
  }

  // Invalidate
  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern);
    if (keys.length) await this.client.del(keys);
  }

  async onModuleDestroy() {
    await this.client.quit();
  }
}
```

---

## Cache Module

```typescript
// cache/cache.module.ts
import { Module, CacheModule } from '@nestjs/common';
import * as redisStore from 'cache-manager-ioredis';
import { ConfigService } from '@nestjs/config';

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: (config: ConfigService) => ({
        store: redisStore,
        host: config.get('REDIS_HOST', 'localhost'),
        port: config.get('REDIS_PORT', 6379),
        ttl: config.get('REDIS_TTL', 3600),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class CacheModule {}
```

---

## Usage

```typescript
// users/users.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { Cache } from 'cache-manager';
import { CACHE_MANAGER } from '@nestjs/cache-manager';

@Injectable()
export class UsersService {
  constructor(@Inject(CACHE_MANAGER) private cache: Cache) {}

  @CacheKey('user:1')
  @CacheTTL(3600)
  async findOne(id: number) {
    return this.cache.wrap(
      `user:${id}`,
      () => this.repo.findOne({ where: { id } }),
    );
  }

  async update(id: number, data: any) {
    await this.repo.update(id, data);
    await this.cache.del(`user:${id}`); // Invalidate
  }
}
```

---

## Environment

```bash
# .env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_TTL=3600
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[Redis-Cheatsheet]] — Redis шпаргалка
