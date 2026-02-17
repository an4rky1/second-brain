---
created: 2026-02-16
tags:
  - performance
  - caching
  - architecture
aliases:
  - Caching Patterns
  - Caching Strategies
related:
  - Redis-Cheatsheet
  - Performance-Optimization
  - System-Design-Basics
---

# Caching Patterns

> [!SUMMARY] Обзор
> Паттерны кэширования для улучшения производительности: Cache-Aside, Write-Through, Write-Behind, стратегия инвалидации.

---

## 📊 Cache Levels

```
┌─────────────────────────────────────────────────────────────────┐
│                  Caching Layers                                  │
│                                                                  │
│  ┌─────────────────┐  L1: Browser Cache (localStorage, CDN)     │
│  ├─────────────────┤  L2: API Gateway Cache (Redis, Memcached)  │
│  ├─────────────────┤  L3: Application Cache (in-memory)         │
│  ├─────────────────┤  L4: Database Cache (query cache)          │
│  └─────────────────┘                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Cache Patterns

### 1. Cache-Aside (Lazy Loading)

```
┌─────────────────────────────────────────────────────┐
│  1. Request → Check Cache                           │
│  2. Cache Miss → Load from DB                       │
│  3. Populate Cache → Return                         │
└─────────────────────────────────────────────────────┘
```

```typescript
// Implementation
class UserService {
  constructor(
    private cache: RedisService,
    private db: UserRepository,
  ) {}

  async getUser(id: string): Promise<User> {
    const cacheKey = `user:${id}`;
    
    // 1. Check cache
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // 2. Cache miss - load from DB
    const user = await this.db.findById(id);
    
    // 3. Populate cache (TTL: 1 hour)
    await this.cache.setex(cacheKey, 3600, JSON.stringify(user));
    
    return user;
  }
}
```

**✅ Плюсы:**
- Простая реализация
- Кэшируются только читаемые данные

**❌ Минусы:**
- Cache miss penalty
- Stale data проблема

---

### 2. Write-Through

```
┌─────────────────────────────────────────────────────┐
│  1. Write → Update Cache                            │
│  2. Cache → Update DB (sync)                        │
│  3. Return                                          │
└─────────────────────────────────────────────────────┘
```

```typescript
class UserService {
  async updateUser(id: string, data: Partial<User>): Promise<User> {
    // 1. Update DB
    const user = await this.db.update(id, data);
    
    // 2. Update cache
    await this.cache.setex(`user:${id}`, 3600, JSON.stringify(user));
    
    return user;
  }
}
```

**✅ Плюсы:**
- Data consistency
- No stale reads

**❌ Минусы:**
- Write penalty
- Write latency

---

### 3. Write-Behind (Write-Back)

```
┌─────────────────────────────────────────────────────┐
│  1. Write → Update Cache                            │
│  2. Return immediately                              │
│  3. Async → Update DB (batched)                     │
└─────────────────────────────────────────────────────┘
```

```typescript
class UserService {
  private writeQueue = new Queue('user-writes');

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    // 1. Update cache immediately
    const user = { ...(await this.getUser(id)), ...data };
    await this.cache.setex(`user:${id}`, 3600, JSON.stringify(user));
    
    // 2. Queue DB update (async)
    await this.writeQueue.add('update-user', { id, data }, {
      delay: 1000,  // Batch updates
    });
    
    return user;
  }
}
```

**✅ Плюсы:**
- Fast writes
- Write batching

**❌ Минусы:**
- Data loss risk
- Complex recovery

---

### 4. Refresh-Ahead

```
┌─────────────────────────────────────────────────────┐
│  Proactively refresh cache before expiration        │
│  Based on access patterns                           │
└─────────────────────────────────────────────────────┘
```

```typescript
class CacheService {
  async getWithRefresh(key: string, ttl: number, refreshFn: () => Promise<any>) {
    const cached = await this.cache.get(key);
    
    if (!cached) {
      const data = await refreshFn();
      await this.cache.setex(key, ttl, JSON.stringify(data));
      return data;
    }
    
    const parsed = JSON.parse(cached);
    const ttlLeft = await this.cache.ttl(key);
    
    // Refresh if less than 20% TTL left
    if (ttlLeft < ttl * 0.2) {
      // Refresh in background
      this.refresh(key, ttl, refreshFn);
    }
    
    return parsed;
  }
  
  private async refresh(key: string, ttl: number, refreshFn: () => Promise<any>) {
    try {
      const data = await refreshFn();
      await this.cache.setex(key, ttl, JSON.stringify(data));
    } catch (error) {
      console.error('Cache refresh failed:', error);
    }
  }
}
```

**✅ Плюсы:**
- No cache miss penalty
- Fresh data

**❌ Минусы:**
- Complex implementation
- May refresh unused data

---

## 🗑️ Cache Invalidation

### Time-Based (TTL)

```typescript
// Simple TTL
await redis.setex('user:123', 3600, JSON.stringify(user));  // 1 hour

// Different TTLs by data type
const TTL = {
  USER: 3600,           // 1 hour
  PRODUCT: 1800,        // 30 min
  SESSION: 86400,       // 24 hours
  ANALYTICS: 300,       // 5 min
};
```

### Explicit Invalidation

```typescript
// Invalidate on update
async updateUser(id: string, data: Partial<User>) {
  const user = await db.update(id, data);
  await cache.del(`user:${id}`);  // Invalidate
  return user;
}

// Invalidate related keys
async deleteUser(id: string) {
  await db.delete(id);
  
  // Delete related cache
  await cache.del(`user:${id}`);
  await cache.del(`user:${id}:posts`);
  await cache.del(`user:${id}:comments`);
  
  // Or use pattern
  const keys = await cache.keys(`user:${id}:*`);
  if (keys.length > 0) {
    await cache.del(keys);
  }
}
```

### Cache-Aside with Version

```typescript
class CachedService {
  async getData(id: string) {
    const version = await this.getVersion(id);
    const cacheKey = `data:${id}:v${version}`;
    
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    const data = await this.db.findById(id);
    await this.cache.setex(cacheKey, 3600, JSON.stringify(data));
    
    return data;
  }
  
  async updateData(id: string, data: Partial<Data>) {
    await this.db.update(id, data);
    // Increment version (invalidates old cache)
    await this.cache.incr(`version:${id}`);
  }
}
```

---

## 📦 Multi-Level Caching

```typescript
class MultiLevelCache {
  constructor(
    private l1: LocalCache,      // In-memory (LRU)
    private l2: RedisCache,      // Distributed
  ) {}

  async get<T>(key: string): Promise<T | null> {
    // L1 cache
    const l1Value = await this.l1.get(key);
    if (l1Value) {
      return l1Value as T;
    }
    
    // L2 cache
    const l2Value = await this.l2.get(key);
    if (l2Value) {
      // Populate L1
      await this.l1.set(key, l2Value, 60);  // 1 min L1 TTL
      return JSON.parse(l2Value) as T;
    }
    
    return null;
  }

  async set(key: string, value: any, l1Ttl = 60, l2Ttl = 3600) {
    await this.l1.set(key, value, l1Ttl);
    await this.l2.setex(key, l2Ttl, JSON.stringify(value));
  }

  async invalidate(key: string) {
    await this.l1.del(key);
    await this.l2.del(key);
  }
}
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Use meaningful cache keys
`user:${userId}`
`user:${userId}:posts`
`posts:page:${page}:limit:${limit}`

// 2. Set appropriate TTLs
const TTL = {
  STATIC: 86400 * 7,    // 7 days
  DYNAMIC: 3600,        // 1 hour
  REALTIME: 60,         // 1 min
};

// 3. Handle cache failures gracefully
try {
  const cached = await cache.get(key);
  if (cached) return cached;
} catch (error) {
  logger.error('Cache error:', error);
  // Fallback to DB
}

// 4. Use cache warming for critical data
async function warmCache() {
  const topUsers = await db.users.findTop(100);
  for (const user of topUsers) {
    await cache.setex(`user:${user.id}`, 3600, JSON.stringify(user));
  }
}

// 5. Monitor cache hit rate
const hitRate = hits / (hits + misses);
if (hitRate < 0.8) {
  // Consider increasing TTL or cache size
}
```

### ❌ Не делать

```typescript
// 1. Don't cache everything
// Cache only expensive operations

// 2. Don't use infinite TTL
// Always set expiration

// 3. Don't cache sensitive data without encryption
await cache.set('session', encrypt(sessionData));

// 4. Don't ignore cache stampede
// Use locks or singleflight

// 5. Don't forget about memory limits
// Monitor cache size and evict old entries
```

---

## 🔗 Связанные заметки

- [[Redis-Cheatsheet]] — Redis для кэширования
- [[Performance-Optimization]] — Оптимизация производительности
- [[System-Design-Basics]] — System design

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Cache-Aside** — start simple
> 2. **TTL всегда** — избегайте stale data
> 3. **Invalidation strategy** — plan ahead
> 4. **Monitor hit rate** — >80% is good
> 5. **Multi-level** — L1 (fast) + L2 (large)

> [!INFO] Cache Invalidation — 2 hard problems
> 
> - Name cache keys consistently
> - Document invalidation rules
> - Test cache behavior
