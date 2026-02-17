---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - patterns
  - best-practices
aliases:
  - TS Patterns
  - TypeScript Best Practices
related:
  - TypeScript-MOC
  - TypeScript-Basics
  - TypeScript-DataFetching
---

# TypeScript — Паттерны и лучшие практики

> [!SUMMARY] Обзор
> Паттерны проектирования, best practices, domain типы и архитектурные решения для TypeScript.

---

## Domain Types

### Деньги и валюта

```typescript
type Currency = 'USD' | 'EUR' | 'GBP' | 'RUB' | 'JPY' | 'CNY';

type Money = {
  amount: number;      // в минимальных единицах (центы/копейки)
  currency: Currency;
};

function createMoney(amount: number, currency: Currency): Money {
  return { amount, currency };
}

function addMoney(a: Money, b: Money): Money {
  if (a.currency !== b.currency) {
    throw new Error('Cannot add money with different currencies');
  }
  return { amount: a.amount + b.amount, currency: a.currency };
}

function formatMoney(money: Money, locale: string = 'en-US'): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency: money.currency,
    minimumFractionDigits: 2,
  }).format(money.amount / 100);
}

// Использование
const price: Money = createMoney(1999, 'USD');
const total = addMoney(price, createMoney(500, 'USD'));
formatMoney(total);  // "$24.99"
```

### Даты и диапазоны

```typescript
type DateRange = {
  start: Date;
  end: Date;
};

function createDateRange(start: Date, end: Date): DateRange | null {
  if (start > end) return null;
  return { start, end };
}

function isDateInRange(date: Date, range: DateRange): boolean {
  return date >= range.start && date <= range.end;
}

function dateRangesOverlap(a: DateRange, b: DateRange): boolean {
  return a.start <= b.end && b.start <= a.end;
}

function getDaysInRange(range: DateRange): number {
  const msPerDay = 24 * 60 * 60 * 1000;
  return Math.round((range.end.getTime() - range.start.getTime()) / msPerDay) + 1;
}

// Использование
const booking = createDateRange(new Date('2024-01-01'), new Date('2024-01-07'));
if (booking) {
  getDaysInRange(booking);  // 7
}
```

### Пагинация

```typescript
type PaginationParams = {
  page: number;
  limit: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
};

type PaginatedResult<T> = {
  items: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
};

function calculatePagination(
  total: number,
  page: number,
  limit: number
): { offset: number; totalPages: number; hasMore: boolean } {
  const totalPages = Math.ceil(total / limit);
  const offset = (page - 1) * limit;
  const hasMore = page < totalPages;
  return { offset, totalPages, hasMore };
}

function createPaginatedResult<T>(
  items: T[],
  total: number,
  page: number,
  limit: number
): PaginatedResult<T> {
  const totalPages = Math.ceil(total / limit);
  return { items, total, page, limit, totalPages };
}

// Использование
const { offset, totalPages, hasMore } = calculatePagination(100, 3, 10);
// offset: 20, totalPages: 10, hasMore: true
```

### Email и UUID

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UUID = Brand<string, 'UUID'>;
type Email = Brand<string, 'Email'>;
type JwtToken = Brand<string, 'JwtToken'>;

function createUUID(): UUID {
  return crypto.randomUUID() as UUID;
}

function createEmail(email: string): Email | null {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email) ? (email as Email) : null;
}

function createJwtToken(token: string): JwtToken | null {
  const parts = token.split('.');
  return parts.length === 3 ? (token as JwtToken) : null;
}

// Использование
const userId = createUUID();
const email = createEmail('user@example.com');
if (email) {
  // email имеет тип Email, не просто string
}
```

---

## API Response Types

### Базовый ответ API

```typescript
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: { code: string; message: string; details?: unknown } };

async function fetchUser(id: number): Promise<ApiResponse<User>> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

const result = await fetchUser(1);
if (result.success) {
  console.log(result.data);  // User
} else {
  console.error(result.error.code, result.error.message);
}
```

### Paginated response

```typescript
type PaginatedResponse<T> = {
  data: T[];
  meta: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
};

async function fetchUsers(params: PaginationParams): Promise<PaginatedResponse<User>> {
  const response = await fetch(`/api/users?page=${params.page}&limit=${params.limit}`);
  return response.json();
}
```

### Result type (как в Rust)

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function safeJsonParse<T>(json: string): Result<T, SyntaxError> {
  try {
    return { ok: true, value: JSON.parse(json) as T };
  } catch (e) {
    return { ok: false, error: e as SyntaxError };
  }
}

function safeExecute<T, E = Error>(
  fn: () => T,
  errorFactory?: (error: unknown) => E
): Result<T, E> {
  try {
    return { ok: true, value: fn() };
  } catch (e) {
    return { ok: false, error: errorFactory ? errorFactory(e) : (e as E) };
  }
}

async function safeExecuteAsync<T, E = Error>(
  fn: () => Promise<T>,
  errorFactory?: (error: unknown) => E
): Promise<Result<T, E>> {
  try {
    return { ok: true, value: await fn() };
  } catch (e) {
    return { ok: false, error: errorFactory ? errorFactory(e) : (e as E) };
  }
}

// Использование
const result = safeExecute(() => JSON.parse(invalidJson));
if (result.ok) {
  console.log(result.value);
} else {
  console.error(result.error);
}
```

### Option type (как в FP)

```typescript
type Option<T> = Some<T> | None;
type Some<T> = { tag: 'some'; value: T };
type None = { tag: 'none' };

function some<T>(value: T): Some<T> {
  return { tag: 'some', value };
}

const none: None = { tag: 'none' };

function getOption<T>(option: Option<T>, fallback: T): T {
  return option.tag === 'some' ? option.value : fallback;
}

function mapOption<T, U>(option: Option<T>, fn: (value: T) => U): Option<U> {
  return option.tag === 'some' ? some(fn(option.value)) : none;
}

function flatMapOption<T, U>(option: Option<T>, fn: (value: T) => Option<U>): Option<U> {
  return option.tag === 'some' ? fn(option.value) : none;
}

// Использование
const maybeUser = findUser(id);  // Option<User>
const userName = getOption(mapOption(maybeUser, u => u.name), 'Anonymous');
```

---

## Event Types

### Typed events

```typescript
type EventMap = {
  click: MouseEvent;
  focus: FocusEvent;
  input: InputEvent;
  custom: { value: string; timestamp: number };
};

type EventListener<K extends keyof EventMap> = (event: EventMap[K]) => void;

function addEvent<K extends keyof EventMap>(
  element: HTMLElement,
  type: K,
  listener: EventListener<K>
) {
  element.addEventListener(type, listener as EventListener);
}

function removeEvent<K extends keyof EventMap>(
  element: HTMLElement,
  type: K,
  listener: EventListener<K>
) {
  element.removeEventListener(type, listener as EventListener);
}

// Использование
addEvent(button, 'click', (e) => {
  console.log(e.clientX, e.clientY);  // MouseEvent
});
```

### State machine

```typescript
type MachineState<T extends string> = {
  state: T;
  transitions: Record<T, T[]>;
};

type AuthState = MachineState<'idle' | 'loading' | 'success' | 'error'>;

const authMachine: AuthState = {
  state: 'idle',
  transitions: {
    idle: ['loading'],
    loading: ['success', 'error'],
    success: ['idle'],
    error: ['loading'],
  },
};

function canTransition(state: AuthState, to: string): boolean {
  return state.transitions[state.state].includes(to as never);
}

function transition(state: AuthState, to: string): AuthState {
  if (!canTransition(state, to)) {
    throw new Error(`Invalid transition from ${state.state} to ${to}`);
  }
  return { ...state, state: to as never };
}
```

---

## Repository Pattern

```typescript
interface Repository<T, ID = number> {
  findById(id: ID): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(entity: Omit<T, 'id'>): Promise<T>;
  update(id: ID, entity: Partial<T>): Promise<T | null>;
  delete(id: ID): Promise<boolean>;
  exists(id: ID): Promise<boolean>;
}

// Implementation
class UserRepository implements Repository<User, number> {
  constructor(private db: Database) {}

  async findById(id: number): Promise<User | null> {
    return this.db.query('SELECT * FROM users WHERE id = ?', [id]);
  }

  async findAll(): Promise<User[]> {
    return this.db.query('SELECT * FROM users');
  }

  async create(entity: Omit<User, 'id'>): Promise<User> {
    const result = await this.db.query(
      'INSERT INTO users (name, email) VALUES (?, ?)',
      [entity.name, entity.email]
    );
    return { ...entity, id: result.insertId };
  }

  async update(id: number, entity: Partial<User>): Promise<User | null> {
    await this.db.query('UPDATE users SET ? WHERE id = ?', [entity, id]);
    return this.findById(id);
  }

  async delete(id: number): Promise<boolean> {
    const result = await this.db.query('DELETE FROM users WHERE id = ?', [id]);
    return result.affectedRows > 0;
  }

  async exists(id: number): Promise<boolean> {
    const result = await this.db.query(
      'SELECT 1 FROM users WHERE id = ? LIMIT 1',
      [id]
    );
    return result.length > 0;
  }
}
```

---

## Builder Pattern

```typescript
class QueryBuilder {
  private selectFields: string[] = [];
  private fromTable: string = '';
  private whereClauses: string[] = [];
  private orderByFields: string[] = [];
  private limitValue?: number;
  private offsetValue?: number;

  select(...fields: string[]): this {
    this.selectFields = fields;
    return this;
  }

  from(table: string): this {
    this.fromTable = table;
    return this;
  }

  where(condition: string): this {
    this.whereClauses.push(condition);
    return this;
  }

  orderBy(field: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.orderByFields.push(`${field} ${direction}`);
    return this;
  }

  limit(value: number): this {
    this.limitValue = value;
    return this;
  }

  offset(value: number): this {
    this.offsetValue = value;
    return this;
  }

  build(): string {
    let query = `SELECT ${this.selectFields.join(', ') || '*'} FROM ${this.fromTable}`;
    
    if (this.whereClauses.length > 0) {
      query += ` WHERE ${this.whereClauses.join(' AND ')}`;
    }
    
    if (this.orderByFields.length > 0) {
      query += ` ORDER BY ${this.orderByFields.join(', ')}`;
    }
    
    if (this.limitValue !== undefined) {
      query += ` LIMIT ${this.limitValue}`;
    }
    
    if (this.offsetValue !== undefined) {
      query += ` OFFSET ${this.offsetValue}`;
    }
    
    return query;
  }
}

// Использование
const query = new QueryBuilder()
  .select('id', 'name', 'email')
  .from('users')
  .where('active = true')
  .where('created_at > "2024-01-01"')
  .orderBy('created_at', 'DESC')
  .limit(10)
  .offset(0)
  .build();
```

---

## Best Practices

### ✅ Делать

```typescript
// 1. Экспортируйте типы из отдельных файлов
// types/user.ts
export interface User { id: number; name: string; }
// Использование: import type { User } from './types/user';

// 2. Используйте const assertions для литералов
const roles = ['admin', 'user'] as const;
type Role = typeof roles[number];  // "admin" | "user"

// 3. Типизируйте границы приложения
interface CreateUserDTO {
  name: string;
  email: string;
}

interface UserResponse {
  success: true;
  data: User;
}

// 4. Используйте discriminated unions
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

// 5. Generic для переиспользования
interface Repository<T, ID = number> {
  findById(id: ID): Promise<T | null>;
  findAll(): Promise<T[]>;
}
```

### ❌ Не делать

```typescript
// 1. Избегайте any
let data: any;  // ❌
let data: unknown;  // ✅

// 2. Не используйте non-null assertion без необходимости
let value: string | null = null;
value!.length;  // ❌ (может упасть)
if (value) value.length;  // ✅

// 3. Не дублируйте типы
interface User { id: number; name: string; }
type UserDTO = { id: number; name: string; };  // ❌
type UserDTO = User;  // ✅

// 4. Избегайте сложных условных типов
// Делайте их простыми и читаемыми
```

---

## Архитектурные паттерны

### Layered Architecture

```typescript
// Controllers (HTTP layer)
interface UserController {
  getUser(req: Request, res: Response): Promise<void>;
  createUser(req: Request, res: Response): Promise<void>;
}

// Services (Business logic)
interface UserService {
  getUserById(id: number): Promise<User | null>;
  createUser(data: CreateUserDTO): Promise<User>;
}

// Repositories (Data access)
interface UserRepository {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserData): Promise<User>;
}

// Зависимости: Controller → Service → Repository
```

### Dependency Injection

```typescript
interface Database {
  query(sql: string, params?: unknown[]): Promise<unknown[]>;
}

class UserRepository {
  constructor(private db: Database) {}
  
  async findById(id: number): Promise<User | null> {
    const results = await this.db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    return results[0] as User | null;
  }
}

class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async getUserById(id: number): Promise<User | null> {
    return this.userRepository.findById(id);
  }
}

// Composition root
const db: Database = new PostgresDatabase(config);
const userRepository = new UserRepository(db);
const userService = new UserService(userRepository);
```

---

## 🔗 Связанные заметки

- [[TypeScript-MOC]] — индекс раздела
- [[TypeScript-Basics]] — основы
- [[TypeScript-Utility-Types]] — типы и дженерики
- [[TypeScript-Functions]] — полезные функции
- [[TypeScript-DataFetching]] — fetch, API

---

> [!TIP] Совет
> 4. **Builder для сложных объектов** — читаемое создание объектов
> 5. **DI для тестируемости** — лёгкая замена зависимостей
