---
created: 2026-02-16
tags:
  - cheat-sheet
  - nosql
  - mongodb
  - redis
aliases:
  - NoSQL Cheatsheet
  - MongoDB & Redis Reference
related:
  - Database-Design
  - Caching-Patterns
  - Data-Modeling
---

# NoSQL — Полная шпаргалка

> [!SUMMARY] Обзор
> NoSQL базы данных: документные (MongoDB), key-value (Redis), column-family, graph. Для специфических use cases: кэширование, сессии, аналитика, графовые данные.

---

## 📚 MongoDB

### Теория

```
┌─────────────────────────────────────────────────────┐
│              MongoDB Architecture                    │
│                                                      │
│  Document (BSON) → Collection → Database            │
│                                                      │
│  {                                                    │
│    "_id": ObjectId,                                  │
│    "name": "John",                                   │
│    "tags": ["admin", "user"],                        │
│    "address": {                                      │
│      "city": "NYC",                                  │
│      "zip": "10001"                                  │
│    }                                                 │
│  }                                                   │
└─────────────────────────────────────────────────────┘
```

### CRUD операции

```javascript
// Создание
db.users.insertOne({
  name: 'John',
  email: 'john@example.com',
  age: 30,
  createdAt: new Date()
});

db.users.insertMany([
  { name: 'Jane', email: 'jane@example.com' },
  { name: 'Bob', email: 'bob@example.com' }
]);

// Чтение
db.users.find();
db.users.find({ age: { $gt: 25 } });
db.users.find({ name: { $regex: /john/i } });
db.users.findOne({ email: 'john@example.com' });

// Операторы
{ field: { $eq: value } }      // Равно
{ field: { $ne: value } }      // Не равно
{ field: { $gt: value } }      // Больше
{ field: { $gte: value } }     // Больше или равно
{ field: { $lt: value } }      // Меньше
{ field: { $lte: value } }     // Меньше или равно
{ field: { $in: [v1, v2] } }   // В списке
{ field: { $nin: [v1, v2] } }  // Не в списке
{ $and: [{...}, {...}] }       // И
{ $or: [{...}, {...}] }        // ИЛИ
{ $not: {...} }                // НЕ
{ $nor: [{...}, {...}] }       // НИ

// Обновление
db.users.updateOne(
  { _id: ObjectId('...') },
  { $set: { age: 31, updatedAt: new Date() } }
);

db.users.updateMany(
  { status: 'inactive' },
  { $set: { status: 'archived' } }
);

// Операторы обновления
{ $set: { field: value } }           // Установить
{ $unset: { field: 1 } }             // Удалить поле
{ $inc: { count: 1 } }               // Инкремент
{ $push: { tags: 'new' } }           // Добавить в массив
{ $pull: { tags: 'old' } }           // Удалить из массива
{ $addToSet: { tags: 'unique' } }    // Добавить уникальное
{ $rename: { old: 'new' } }          // Переименовать

// Удаление
db.users.deleteOne({ _id: ObjectId('...') });
db.users.deleteMany({ status: 'deleted' });

// Projection
db.users.find({}, { name: 1, email: 1, _id: 0 });
db.users.find({}, { 'address.city': 1 });
```

### Агрегация

```javascript
// Pipeline
db.orders.aggregate([
  { $match: { status: 'completed' } },
  { $group: {
    _id: '$customerId',
    totalOrders: { $sum: 1 },
    totalAmount: { $sum: '$amount' },
    avgAmount: { $avg: '$amount' }
  }},
  { $having: { totalAmount: { $gt: 1000 } } },
  { $sort: { totalAmount: -1 } },
  { $limit: 10 },
  { $lookup: {
    from: 'customers',
    localField: '_id',
    foreignField: '_id',
    as: 'customer'
  }},
  { $unwind: '$customer' },
  { $project: {
    _id: 0,
    customerId: '$_id',
    customerName: '$customer.name',
    totalOrders: 1,
    totalAmount: 1
  }}
]);

// Stages
$match      // Фильтрация
$group      // Группировка
$sort       // Сортировка
$limit      // Лимит
$skip       // Пропуск
$project    // Проекция
$unwind     // Развернуть массив
$lookup     // JOIN
$facet      // Multiple pipelines
$bucket     // Гистограмма
```

### Индексы

```javascript
// Создание
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ name: 1, age: -1 });
db.users.createIndex({ location: '2dsphere' });  // Geo
db.users.createIndex({ tags: 1 }, { sparse: true });
db.users.createIndex({ name: 'text' });  // Full-text

// Compound индекс (порядок важен!)
db.orders.createIndex({ customerId: 1, createdAt: -1 });

// TTL индекс
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });

// Анализ
db.users.getIndexes();
db.users.dropIndex('email_1');

// Explain
db.users.find({ email: 'test@example.com' }).explain('executionStats');
```

---

## 📚 Redis

### Теория

```
┌─────────────────────────────────────────────────────┐
│              Redis Data Types                        │
│                                                      │
│  String    →  Кэши, сессии, счетчики               │
│  Hash      →  Объекты, профили                      │
│  List      →  Очереди, ленты                        │
│  Set       →  Уникальные значения, теги             │
│  Sorted Set →  Рейтинги, leaderboards               │
│  Stream    →  Event sourcing, logs                  │
│  Pub/Sub   →  Messaging, real-time                  │
└─────────────────────────────────────────────────────┘
```

### Строки

```bash
# Базовые операции
SET key value
GET key
MSET key1 value1 key2 value2
MGET key1 key2
DEL key

# Счетчики
INCR counter
INCRBY counter 10
DECR counter
DECRBY counter 5

# Истечение
SET key value EX 3600  # Истекает через 1 час
SET key value PX 60000 # Истекает через 60 секунд
EXPIRE key 3600
TTL key

# Бинарные операции
APPEND key value
STRLEN key
GETRANGE key 0 10
SETRANGE key 0 value
```

### Hashes

```bash
# Объекты
HSET user:1 name "John" email "john@example.com" age 30
HGET user:1 name
HMGET user:1 name email
HGETALL user:1
HDEL user:1 age
HLEN user:1
HEXISTS user:1 name

# Инкремент
HINCRBY user:1 points 10
HINCRBYFLOAT user:1 balance 0.5

# Scan
HSCAN user:1 0 MATCH "name*"
```

### Lists

```bash
# Queue (FIFO)
LPUSH queue item1 item2
RPUSH queue item3
LPOP queue
RPOP queue
LINDEX queue 0
LRANGE queue 0 -1
LLEN queue
LREM queue 0 "value"  # Удалить
LTRIM queue 0 100     # Обрезать

# Blocking (для очередей)
BLPOP queue1 queue2 0  # Блокировать пока нет элемента
BRPOP queue 0
BRPOPLPUSH source dest 0

# Stack (LIFO)
LPUSH stack item
LPOP stack
```

### Sets

```bash
# Уникальные значения
SADD tags:post:1 "tech" "programming" "redis"
SMEMBERS tags:post:1
SISMEMBER tags:post:1 "tech"
SCARD tags:post:1
SREM tags:post:1 "tech"

# Set operations
SUNION set1 set2        # Объединение
SINTER set1 set2        # Пересечение
SDIFF set1 set2         # Разность

# Random
SRANDMEMBER set 5       # 5 случайных
SPOP set 2              # 2 случайных и удалить
```

### Sorted Sets

```bash
# Leaderboard
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2"
ZADD leaderboard 150 "player3"

ZREVRANGE leaderboard 0 9 WITHSCORES  # Топ 10
ZREVRANK leaderboard "player1"        # Ранг
ZSCORE leaderboard "player1"          # Очки
ZCARD leaderboard                     # Количество
ZREM leaderboard "player1"            # Удалить

# Range by score
ZRANGEBYSCORE leaderboard 100 200
ZCOUNT leaderboard 100 200

# Increment
ZINCRBY leaderboard 10 "player1"
```

### Pub/Sub

```bash
# Подписка
SUBSCRIBE channel1 channel2
PSUBSCRIBE news.*  # Pattern

# Публикация
PUBLISH channel1 "message"

# В приложении (Node.js)
const subscriber = redis.createClient();
const publisher = redis.createClient();

subscriber.subscribe('notifications', (message) => {
  console.log('Received:', message);
});

publisher.publish('notifications', 'Hello!');
```

### Streams

```bash
# Добавление
XADD mystream * field1 value1 field2 value2
XADD mystream 1234567890-0 field1 value1

# Чтение
XREAD COUNT 2 STREAMS mystream 0
XREADGROUP GROUP mygroup consumer1 STREAMS mystream >

# Consumer Groups
XGROUP CREATE mystream mygroup $ MKSTREAM
XGROUP CREATECONSUMER mystream mygroup consumer1

# Acknowledge
XACK mystream mygroup 1234567890-0

# Pending
XPENDING mystream mygroup
XCLAIM mystream mygroup consumer1 3600000 1234567890-0
```

---

## 🎯 Patterns

### Caching

```typescript
// Cache-Aside (Lazy Loading)
async function getUser(id: string) {
  // Check cache
  const cached = await redis.get(`user:${id}`);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Cache miss - load from DB
  const user = await db.users.findById(id);
  
  // Populate cache
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  
  return user;
}

// Write-Through
async function updateUser(id: string, data: Partial<User>) {
  // Update DB
  const user = await db.users.findByIdAndUpdate(id, data, { new: true });
  
  // Update cache
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  
  return user;
}

// Cache-Invalidation
async function deleteUser(id: string) {
  await db.users.findByIdAndDelete(id);
  await redis.del(`user:${id}`);
}
```

### Rate Limiting

```typescript
// Fixed Window
async function isRateLimited(userId: string, limit: number, window: number) {
  const key = `ratelimit:${userId}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, window);
  }
  
  return current > limit;
}

// Sliding Window Log
async function isRateLimitedSliding(userId: string, limit: number, window: number) {
  const key = `ratelimit:${userId}`;
  const now = Date.now();
  
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, now - window);
  pipeline.zadd(key, now, now);
  pipeline.zcard(key);
  pipeline.expire(key, window);
  
  const [, , , count] = await pipeline.exec();
  return count > limit;
}

// Token Bucket
async function consumeToken(userId: string, capacity: number, refillRate: number) {
  const key = `bucket:${userId}`;
  const now = Date.now();
  
  const [tokens, lastRefill] = await redis.hmget(key, 'tokens', 'lastRefill');
  
  let currentTokens = tokens ? parseFloat(tokens) : capacity;
  const lastTime = lastRefill ? parseInt(lastRefill) : now;
  
  // Refill tokens
  const elapsed = (now - lastTime) / 1000;
  currentTokens = Math.min(capacity, currentTokens + elapsed * refillRate);
  
  if (currentTokens >= 1) {
    currentTokens -= 1;
    await redis.hmset(key, 'tokens', currentTokens, 'lastRefill', now);
    return true;  // Allowed
  }
  
  return false;  // Rate limited
}
```

### Distributed Lock

```typescript
// Redlock Pattern
async function acquireLock(resource: string, ttl: number): Promise<string | null> {
  const lockId = crypto.randomBytes(16).toString('hex');
  const key = `lock:${resource}`;
  
  const acquired = await redis.set(key, lockId, 'EX', ttl, 'NX');
  
  if (acquired) {
    return lockId;
  }
  
  return null;
}

async function releaseLock(resource: string, lockId: string): Promise<boolean> {
  const key = `lock:${resource}`;
  
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;
  
  const result = await redis.eval(script, 1, key, lockId);
  return result === 1;
}

// Usage
async function processWithLock(resource: string, fn: () => Promise<void>) {
  const lockId = await acquireLock(resource, 10000);
  
  if (!lockId) {
    throw new Error('Could not acquire lock');
  }
  
  try {
    await fn();
  } finally {
    await releaseLock(resource, lockId);
  }
}
```

### Session Storage

```typescript
// Express Session with Redis
import session from 'express-session';
import connectRedis from 'connect-redis';

const RedisStore = connectRedis(session);

app.use(session({
  store: new RedisStore({ 
    client: redis,
    prefix: 'sess:',
    ttl: 86400  // 24 hours
  }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 86400000
  }
}));
```

---

## 🔗 Связанные заметки

- [[Database-Design]] — Проектирование БД
- [[Caching-Patterns]] — Паттерны кэширования
- [[Data-Modeling]] — Моделирование данных

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **MongoDB** — для документов с гибкой схемой
> 2. **Redis** — для кэша, сессий, очередей
> 3. **Индексы** — критичны для производительности
> 4. **TTL** — используйте для временных данных
> 5. **Pipeline** — для атомарных операций в Redis

> [!INFO] Инструменты
> ```bash
> # MongoDB
> mongosh                          # Shell
> mongoimport --db test --file data.json
> mongodump --db test --out backup/
> mongorestore --db test backup/
> 
> # Redis
> redis-cli                        # CLI
> redis-cli MONITOR                # Monitor commands
> redis-cli --stat                 # Statistics
> redis-benchmark                  # Benchmark
> ```
