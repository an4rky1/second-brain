---
created: 2026-02-16
tags:
  - cheat-sheet
  - redis
  - nosql
  - cache
  - database
aliases:
  - Redis Cheatsheet
  - Redis Reference
related:
  - NoSQL-Cheatsheet
  - Caching-Patterns
  - Docker-Templates
---

# Redis — Полная шпаргалка

> [!SUMMARY] Обзор
> Redis — in-memory key-value хранилище. Кэширование, сессии, очереди, pub/sub, стримы. Поддержка различных структур данных.

---

## 📚 Теория

### Типы данных

```
┌─────────────────────────────────────────────────────┐
│              Redis Data Types                        │
│                                                      │
│  String    →  Кэши, сессии, счетчики               │
│  Hash      →  Объекты, профили пользователей        │
│  List      →  Очереди, ленты событий                │
│  Set       →  Уникальные значения, теги             │
│  Sorted Set →  Рейтинги, leaderboards               │
│  Stream    →  Event sourcing, логи событий          │
│  Pub/Sub   →  Real-time уведомления, messaging      │
│  Geospatial →  Геоданные, location-based сервисы    │
└─────────────────────────────────────────────────────┘
```

### Use Cases

| Тип | Use Case | Пример |
|-----|----------|--------|
| String | Кэш, сессии, счетчики | User session, page views |
| Hash | Объекты | User profile, product data |
| List | Очереди | Task queue, activity feed |
| Set | Уникальные значения | Tags, blocked users |
| Sorted Set | Рейтинги | Leaderboard, top users |
| Stream | Event sourcing | Order events, audit log |

---

## ⚡ Быстрый старт

```bash
# Установка
sudo apt install redis-server

# Запуск
sudo systemctl start redis
sudo systemctl enable redis

# CLI
redis-cli
redis-cli -h localhost -p 6379
redis-cli -a password

# Проверка
redis-cli ping  # PONG

# Docker
docker run -d -p 6379:6379 --name redis redis:7-alpine
docker run -d -p 6379:6379 redis:7-alpine redis-server --requirepass mypass
```

---

## 🔧 Практические примеры

### Strings

```bash
# Базовые операции
SET key value
GET key
MSET key1 value1 key2 value2
MGET key1 key2
DEL key
EXISTS key

# Истечение (TTL)
SET key value EX 3600      # Истекает через 1 час
SET key value PX 60000     # Истекает через 60 секунд
EXPIRE key 3600            # Установить TTL
PEXPIRE key 60000          # TTL в миллисекундах
TTL key                    # Оставшееся время (сек)
PTTL key                   # Оставшееся время (мс)
PERSIST key                # Убрать TTL

# Счетчики
INCR counter               # Инкремент на 1
INCRBY counter 10          # Инкремент на N
INCRBYFLOAT counter 0.5    # Инкремент на float
DECR counter               # Декремент
DECRBY counter 5           # Декремент на N
GETSET counter 0           # Получить и сбросить

# Бинарные операции
APPEND key value           # Добавить к строке
STRLEN key                 # Длина строки
GETRANGE key 0 10          # Подстрока
SETRANGE key 0 value       # Заменить часть

# Примеры использования
# Счётчик просмотров
INCR page:views:123

# Сессия пользователя
SETEX session:abc123 3600 '{"userId": 1, "role": "admin"}'

# Rate limiting
INCR ratelimit:user:123
EXPIRE ratelimit:user:123 60
```

### Hashes

```bash
# Создание и чтение
HSET user:1 name "John" email "john@example.com" age 30
HGET user:1 name
HMGET user:1 name email
HGETALL user:1

# Модификация
HSET user:1 age 31         # Обновить поле
HDEL user:1 age            # Удалить поле
HINCRBY user:1 points 10   # Инкремент числа
HINCRBYFLOAT user:1 balance 0.5

# Информация
HLEN user:1                # Количество полей
HEXISTS user:1 name        # Существует ли поле
HKEYS user:1               # Все ключи
HVALS user:1               # Все значения
HSCAN user:1 0 MATCH "name*"  # Scan по полям

# Примеры использования
# Профиль пользователя
HSET user:123 name "John" email "john@example.com" created_at "2024-01-01"
HGETALL user:123

# Кэш объекта
HSET cache:product:456 title "Laptop" price 999 stock 50
HMGET cache:product:456 title price

# Корзина покупок
HSET cart:user:123 product:1 2  # 2 штуки товара 1
HSET cart:user:123 product:2 1
HGETALL cart:user:123
```

### Lists

```bash
# Queue (FIFO)
LPUSH queue item1 item2 item3   # Push left (в начало)
RPUSH queue item4               # Push right (в конец)
LPOP queue                      # Pop left (из начала)
RPOP queue                      # Pop right (из конца)

# Blocking operations (для очередей)
BLPOP queue1 queue2 0           # Блокировать пока нет элемента
BRPOP queue 0                   # Blocking RPOP
BRPOPLPUSH source dest 0        # Атомарно: pop из source, push в dest

# Информация
LLEN queue                      # Длина списка
LRANGE queue 0 -1               # Все элементы
LRANGE queue 0 10               # Первые 10
LINDEX queue 0                  # Элемент по индексу
LPOS queue "value"              # Позиция элемента

# Модификация
LREM queue 0 "value"            # Удалить все вхождения
LTRIM queue 0 100               # Обрезать до 100 элементов
LSET queue 0 "newvalue"         # Установить значение по индексу
LINSERT queue BEFORE "val" "new" # Вставить перед значением

# Примеры использования
# Очередь задач
LPUSH tasks:pending '{"id": 1, "type": "email"}'
BRPOP tasks:pending 0  # Блокировать пока нет задачи

# Лента событий
LPUSH feed:user:123 "post:1" "post:2" "post:3"
LRANGE feed:user:123 0 9  # Последние 10

# Стек (LIFO)
LPUSH stack item1 item2
LPOP stack
```

### Sets

```bash
# Базовые операции
SADD tags:post:1 "tech" "programming" "redis"  # Добавить
SMEMBERS tags:post:1                            # Все элементы
SISMEMBER tags:post:1 "tech"                    # Проверка
SCARD tags:post:1                               # Количество
SREM tags:post:1 "tech"                         # Удалить

# Set operations
SUNION set1 set2           # Объединение
SUNIONSTORE dest set1 set2 # Объединение в новый set
SINTER set1 set2           # Пересечение
SINTERSTORE dest set1 set2 # Пересечение в новый set
SDIFF set1 set2            # Разность (set1 - set2)
SDIFFSTORE dest set1 set2  # Разность в новый set

# Random
SRANDMEMBER set 5          # 5 случайных элементов
SPOP set 2                 # 2 случайных и удалить

# Примеры использования
# Теги поста
SADD post:1:tags "redis" "database" "nosql"
SMEMBERS post:1:tags

# Общие друзья
SINTER friends:user:1 friends:user:2

# Blocked users
SADD blocked:users "spam_bot_123"
SISMEMBER blocked:users "spam_bot_123"

# Онлайн пользователи
SADD online:users "user:1" "user:2"
SCARD online:users  # Количество онлайн
```

### Sorted Sets

```bash
# Добавление
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2"
ZADD leaderboard 150 "player3"

# Чтение
ZREVRANGE leaderboard 0 9 WITHSCORES  # Топ 10 (по убыванию)
ZRANGE leaderboard 0 9 WITHSCORES     # Топ 10 (по возрастанию)
ZREVRANK leaderboard "player1"        # Ранг (0-based)
ZRANK leaderboard "player1"
ZSCORE leaderboard "player1"          # Очки
ZCARD leaderboard                     # Количество

# Range by score
ZRANGEBYSCORE leaderboard 100 200           # Очки 100-200
ZRANGEBYSCORE leaderboard -inf +inf LIMIT 0 10  # С лимитом
ZREVRANGEBYSCORE leaderboard +inf -inf      # По убыванию

# Count
ZCOUNT leaderboard 100 200  # Количество в диапазоне

# Модификация
ZINCRBY leaderboard 10 "player1"  # Инкремент очков
ZREM leaderboard "player1"        # Удалить игрока
ZREMRANGEBYRANK leaderboard 0 9   # Удалить топ 10
ZREMRANGEBYSCORE leaderboard 0 100 # Удалить с очками 0-100

# Примеры использования
# Leaderboard игры
ZADD game:leaderboard 1500 "player:1"
ZADD game:leaderboard 2000 "player:2"
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# Топ комментариев
ZADD post:1:comments:top 100 "comment:1"
ZINCRBY post:1:comments:top 1 "comment:1"  # +1 голос

# Приоритетная очередь
ZADD tasks:priority 10 "task:high"
ZADD tasks:priority 5 "task:medium"
ZADD tasks:priority 1 "task:low"
ZRANGE tasks:priority 0 0  # Получить высший приоритет
```

### Streams

```bash
# Добавление
XADD mystream * field1 value1 field2 value2
XADD mystream 1234567890-0 field1 value1  # С явным ID

# Чтение
XREAD COUNT 2 STREAMS mystream 0          # Читать 2 сообщения
XREAD STREAMS mystream $                  # Только новые
XREAD BLOCK 5000 STREAMS mystream $       # Блокировать 5 сек

# Consumer Groups
XGROUP CREATE mystream mygroup $ MKSTREAM  # Создать группу
XGROUP CREATECONSUMER mystream mygroup consumer1

# Чтение из группы
XREADGROUP GROUP mygroup consumer1 STREAMS mystream >  # Новые сообщения
XREADGROUP GROUP mygroup consumer1 STREAMS mystream 0  # Все не ack

# Acknowledge
XACK mystream mygroup 1234567890-0

# Pending
XPENDING mystream mygroup              # Статистика
XPENDING mystream mygroup - + 10       # Детали (10 сообщений)

# Claim (перезапуск зависших)
XCLAIM mystream mygroup consumer1 3600000 1234567890-0

# Примеры использования
# Order events
XADD orders:events * order_id 123 event created timestamp 1704067200
XREADGROUP GROUP orders:processing worker1 STREAMS orders:events >

# Audit log
XADD audit:user:123 * action login ip 1.2.3.4 timestamp 1704067200
```

### Pub/Sub

```bash
# Подписка
SUBSCRIBE channel1 channel2
PSUBSCRIBE news.*  # Pattern subscription

# Публикация
PUBLISH channel1 "message"

# Информация
PUBSUB CHANNELS        # Активные каналы
PUBSUB NUMSUB channel1 # Подписчики канала
PUBSUB NUMPAT          # Pattern подписки

# Примеры использования (Node.js)
# Publisher
const publisher = redis.createClient();
await publisher.publish('notifications', JSON.stringify({
  userId: 123,
  type: 'order_completed',
  data: { orderId: 456 }
}));

# Subscriber
const subscriber = redis.createClient();
await subscriber.subscribe('notifications', (message) => {
  const event = JSON.parse(message);
  console.log('Received:', event);
});

await subscriber.pSubscribe('orders.*', (message, channel) => {
  console.log(`Channel ${channel}:`, message);
});
```

### Geospatial

```bash
# Добавление
GEOADD locations 13.361389 38.115556 "Palermo"
GEOADD locations 15.087269 37.502669 "Catania"

# Поиск рядом
GEORADIUS locations 15 37 200 km WITHDIST
GEORADIUSBYMEMBER locations Palermo 200 km WITHDIST

# Расстояние
GEODIST locations Palermo Catania km

# Hash
GEOHASH locations Palermo Catania

# Примеры использования
# Поиск ресторанов рядом
GEOADD restaurants:ny -73.935242 40.730610 "Restaurant A"
GEORADIUS restaurants:ny -73.935242 40.730610 5 km WITHDIST

# Delivery tracking
GEOADD drivers:active -73.935242 40.730610 "driver:123"
GEORADIUS drivers:active -73.935242 40.730610 10 km
```

---

## 🎯 Patterns

### Caching

```typescript
// Cache-Aside (Lazy Loading)
async function getUser(id: string): Promise<User> {
  const cached = await redis.get(`user:${id}`);
  if (cached) {
    return JSON.parse(cached);
  }
  
  const user = await db.users.findById(id);
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  
  return user;
}

// Write-Through
async function updateUser(id: string, data: Partial<User>) {
  const user = await db.users.findByIdAndUpdate(id, data, { new: true });
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}

// Cache Invalidation
async function deleteUser(id: string) {
  await db.users.findByIdAndDelete(id);
  await redis.del(`user:${id}`);
}

// Bulk caching
async function getUsers(ids: string[]): Promise<User[]> {
  const keys = ids.map(id => `user:${id}`);
  const cached = await redis.mGet(keys);
  
  const missing = ids.filter((_, i) => !cached[i]);
  const fromDb = await db.users.find({ _id: { $in: missing } });
  
  // Populate cache
  const pipeline = redis.pipeline();
  fromDb.forEach(user => {
    pipeline.setex(`user:${user.id}`, 3600, JSON.stringify(user));
  });
  await pipeline.exec();
  
  return [...cached.filter(Boolean).map(JSON.parse), ...fromDb];
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
  const key = `ratelimit:sliding:${userId}`;
  const now = Date.now();
  
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, now - window);
  pipeline.zadd(key, now, now.toString());
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
  
  const elapsed = (now - lastTime) / 1000;
  currentTokens = Math.min(capacity, currentTokens + elapsed * refillRate);
  
  if (currentTokens >= 1) {
    currentTokens -= 1;
    await redis.hmset(key, 'tokens', currentTokens, 'lastRefill', now);
    return true;
  }
  
  return false;
}
```

### Distributed Lock

```typescript
// Redlock Pattern
async function acquireLock(resource: string, ttl: number): Promise<string | null> {
  const lockId = crypto.randomBytes(16).toString('hex');
  const key = `lock:${resource}`;
  
  const acquired = await redis.set(key, lockId, 'EX', ttl, 'NX');
  return acquired ? lockId : null;
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
  if (!lockId) throw new Error('Could not acquire lock');
  
  try {
    await fn();
  } finally {
    await releaseLock(resource, lockId);
  }
}
```

### Session Storage

```typescript
// Express Session
import session from 'express-session';

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

## 🐛 Troubleshooting

| Проблема | Решение |
|----------|---------|
| `OOM command not allowed` | Увеличьте maxmemory или очистите ключи |
| `MISCONF Redis is configured` | Проверьте save конфигурацию |
| `Connection refused` | Проверьте что Redis запущен и порт открыт |
| `ERR AUTH` | Проверьте пароль в конфигурации |
| Медленная работа | Проверьте maxmemory-policy, используйте pipeline |

---

## 🔗 Связанные заметки

- [[NoSQL-Cheatsheet]] — NoSQL основы
- [[Caching-Patterns]] — Паттерны кэширования
- [[Docker-Templates]] — Docker для Redis

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Используйте префиксы** — `user:123`, `session:abc`
> 2. **TTL для всего** — избегайте утечек памяти
> 3. **Pipeline для батчей** — уменьшает RTT
> 4. **Lua скрипты** — атомарные операции
> 5. **Monitor для отладки** — `redis-cli monitor`

> [!INFO] Конфигурация
> ```bash
> # redis.conf
> maxmemory 256mb
> maxmemory-policy allkeys-lru
> save 900 1
> save 300 10
> save 60 10000
> appendonly yes
> ```
