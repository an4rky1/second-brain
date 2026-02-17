---
created: 2026-02-16
tags:
  - pagination
  - api
  - database
aliases:
  - Pagination Patterns
  - Offset vs Cursor Pagination
related:
  - REST-API
  - PostgreSQL-Cheatsheet
  - React-Cheatsheet
---

# Pagination Patterns

> [!SUMMARY] Обзор
> Паттерны пагинации данных: offset-based, cursor-based, keyset. Выбор метода зависит от use case: админка, лента, real-time данные.

---

## 📚 Типы пагинации

### 1. Offset-Based (По смещению)

```
┌─────────────────────────────────────────────────────┐
│  ?page=1&limit=20  →  OFFSET 0 LIMIT 20            │
│  ?page=2&limit=20  →  OFFSET 20 LIMIT 20           │
│  ?page=3&limit=20  →  OFFSET 40 LIMIT 20           │
└─────────────────────────────────────────────────────┘
```

**✅ Когда использовать:**
- Простые CRUD приложения
- Пользователю нужно перейти на конкретную страницу
- Данные не часто меняются

**❌ Проблемы:**
- Performance деградирует на больших OFFSET
- Дубликаты/пропуски при изменении данных
- Не работает для real-time данных

---

### 2. Cursor-Based (По курсору)

```
┌─────────────────────────────────────────────────────┐
│  ?cursor=eyJpZCI6MTAwfQ&limit=20                   │
│  WHERE id > 100 ORDER BY id ASC LIMIT 21           │
│  nextCursor = last_item.id                         │
└─────────────────────────────────────────────────────┘
```

**✅ Когда использовать:**
- Infinite scroll / ленты
- Real-time данные
- Большие объёмы данных
- Консистентность важна

**❌ Проблемы:**
- Нельзя перейти на страницу N
- Сложнее в реализации

---

### 3. Keyset / Seek Method

```
┌─────────────────────────────────────────────────────┐
│  WHERE (created_at, id) < ('2024-01-01', 100)      │
│  ORDER BY created_at DESC, id DESC                 │
│  LIMIT 20                                          │
└─────────────────────────────────────────────────────┘
```

**✅ Когда использовать:**
- Сортировка по дате + ID
- Высокая производительность
- Ленты новостей, сообщения

---

## ⚡ Backend реализация

### Offset Pagination (NestJS)

```typescript
// DTO
export class PaginationDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;
}

// Response
export class PaginatedResponse<T> {
  data: T[];
  meta: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

// Service
async findAll(pagination: PaginationDto) {
  const { page = 1, limit = 20 } = pagination;
  const skip = (page - 1) * limit;

  const [data, total] = await this.repo.findAndCount({
    skip,
    take: limit,
    order: { createdAt: 'DESC' },
  });

  return {
    data,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
      hasNext: page * limit < total,
      hasPrev: page > 1,
    },
  };
}
```

### Cursor Pagination (NestJS)

```typescript
// DTO
export class CursorPaginationDto {
  @IsOptional()
  @IsString()
  cursor?: string;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;
}

// Service
async findAllCursor(pagination: CursorPaginationDto) {
  const { cursor, limit = 20 } = pagination;

  const decodedCursor = cursor 
    ? Buffer.from(cursor, 'base64').toString('utf-8') 
    : null;
  const cursorId = decodedCursor ? parseInt(decodedCursor) : null;

  const query = this.repo.createQueryBuilder('item')
    .orderBy('item.id', 'DESC')
    .take(limit + 1); // +1 чтобы проверить есть ли ещё

  if (cursorId) {
    query.andWhere('item.id < :cursorId', { cursorId });
  }

  const data = await query.getMany();

  const hasNext = data.length > limit;
  if (hasNext) data.pop(); // Удаляем лишний элемент

  const lastItem = data[data.length - 1];

  return {
    data,
    meta: {
      limit,
      hasNext,
      nextCursor: lastItem 
        ? Buffer.from(String(lastItem.id)).toString('base64') 
        : null,
    },
  };
}
```

### PostgreSQL (raw SQL)

```sql
-- Offset pagination
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;  -- page 3, limit 20

-- Считаем total (для meta)
SELECT COUNT(*) FROM users;

-- Cursor pagination (по ID)
SELECT * FROM users
WHERE id < 100  -- cursor
ORDER BY id DESC
LIMIT 21;  -- +1 для проверки hasNext

-- Keyset pagination (composite)
SELECT * FROM posts
WHERE (created_at, id) < ('2024-01-01', 100)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

---

## 🔧 Frontend реализация

### React Hook (Offset)

```typescript
function usePagination(options: { total: number; limit?: number }) {
  const { total, limit = 20 } = options;
  const [page, setPage] = useState(1);
  
  const totalPages = Math.ceil(total / limit);
  const hasNext = page < totalPages;
  const hasPrev = page > 1;

  return {
    page,
    limit,
    total,
    totalPages,
    hasNext,
    hasPrev,
    nextPage: () => setPage(p => Math.min(p + 1, totalPages)),
    prevPage: () => setPage(p => Math.max(p - 1, 1)),
    setPage,
  };
}

// Usage
function UsersList() {
  const { data } = useQuery({
    queryKey: ['users', page],
    queryFn: () => api.getUsers({ page }),
  });

  const pagination = usePagination({ total: data?.meta.total });

  return (
    <div>
      <UsersGrid users={data?.data} />
      <Pagination {...pagination} />
    </div>
  );
}
```

### React Hook (Cursor / Infinite)

```typescript
function useInfiniteQuery(queryKey: string, fetchFn: (cursor?: string) => Promise<any>) {
  const [data, setData] = useState([]);
  const [nextCursor, setNextCursor] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [hasNext, setHasNext] = useState(true);

  const fetchNext = useCallback(async () => {
    if (!hasNext || isLoading) return;
    
    setIsLoading(true);
    try {
      const result = await fetchFn(nextCursor ?? undefined);
      setData(prev => [...prev, ...result.data]);
      setNextCursor(result.meta.nextCursor);
      setHasNext(result.meta.hasNext);
    } finally {
      setIsLoading(false);
    }
  }, [nextCursor, hasNext, isLoading]);

  return { data, isLoading, hasNext, fetchNext };
}

// Usage
function InfiniteFeed() {
  const { data, isLoading, hasNext, fetchNext } = useInfiniteQuery(
    'posts',
    (cursor) => api.getPosts({ cursor, limit: 20 })
  );

  return (
    <InfiniteScroll
      dataLength={data.length}
      next={fetchNext}
      hasMore={hasNext}
      loader={<Loading />}
    >
      <PostsList posts={data} />
    </InfiniteScroll>
  );
}
```

---

## 📊 Сравнение методов

| Метод | Performance | Гибкость | Консистентность | Сложность |
|-------|-------------|----------|-----------------|-----------|
| Offset | Низкая (большие OFFSET) | Высокая | Низкая | Низкая |
| Cursor | Высокая | Средняя | Высокая | Средняя |
| Keyset | Очень высокая | Средняя | Высокая | Средняя |

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Всегда валидируйте limit
const limit = Math.min(Math.max(1, userLimit), 100);

// 2. Используйте дефолтные значения
page = 1, limit = 20

// 3. Индексы для сортировки
CREATE INDEX idx_users_created_at ON users(created_at DESC);

// 4. Для cursor используйте opaque курсоры
const cursor = Buffer.from(JSON.stringify({ id, sortBy })).toString('base64');

// 5. Кэшируйте total count для больших таблиц
```

### ❌ Не делать

```typescript
// 1. Огромные лимиты
limit = 1000;  // ❌
limit = Math.min(userLimit, 100);  // ✅

// 2. OFFSET без индекса
SELECT * FROM users OFFSET 100000;  // ❌ медленно

// 3. COUNT(*) на больших таблицах без кэша
// Используйте приблизительный count

// 4. Доверять пользовательскому sortBy
sortBy = 'createdAt';  // ✅ whitelist
sortBy = userInput;  // ❌ SQL injection риск
```

---

## 🔗 Связанные заметки

- [[REST-API]] — REST API design
- [[PostgreSQL-Cheatsheet]] — SQL запросы
- [[React-Cheatsheet]] — React hooks

---

## 📝 Заметки

> [!TIP] Выбор метода
> 
> - **Offset** — для админок, CRUD с малыми данными
> - **Cursor** — для лент, infinite scroll, real-time
> - **Keyset** — для высоких нагрузок, сортировка по дате
