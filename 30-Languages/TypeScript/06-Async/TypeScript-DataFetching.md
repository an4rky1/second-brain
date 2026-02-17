---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - fetch
  - async
  - api
aliases:
  - TS Data Fetching
  - TypeScript Fetch Guide
related:
  - TypeScript-MOC
  - TypeScript-Functions
  - TypeScript-Patterns
---

# TypeScript — Работа с данными (Fetch, API)

> [!SUMMARY] Обзор
> Полное руководство по работе с данными: fetch, обработка ошибок, abort, retry, кэширование, identity keys.

---

## Базовый Fetch

### Простой запрос

```typescript
type FetchOptions = {
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
  headers?: Record<string, string>;
  body?: unknown;
  signal?: AbortSignal;
};

async function fetchApi<T>(
  url: string,
  options: FetchOptions = {}
): Promise<T> {
  const { method = 'GET', headers = {}, body, signal } = options;

  const config: RequestInit = {
    method,
    headers: {
      'Content-Type': 'application/json',
      ...headers,
    },
    signal,
  };

  if (body) {
    config.body = JSON.stringify(body);
  }

  const response = await fetch(url, config);

  if (!response.ok) {
    throw new ApiError(
      `HTTP ${response.status}: ${response.statusText}`,
      response.status,
      await response.text()
    );
  }

  return response.json() as Promise<T>;
}

// Использование
const user = await fetchApi<User>('/api/users/1');
```

### ApiError класс

```typescript
class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public body?: unknown,
    public url?: string
  ) {
    super(message);
    this.name = 'ApiError';
  }

  isNotFound(): boolean {
    return this.status === 404;
  }

  isUnauthorized(): boolean {
    return this.status === 401;
  }

  isForbidden(): boolean {
    return this.status === 403;
  }

  isServerError(): boolean {
    return this.status >= 500;
  }
}

// Использование
try {
  const user = await fetchApi<User>('/api/users/1');
} catch (error) {
  if (error instanceof ApiError) {
    if (error.isNotFound()) {
      console.log('User not found');
    } else if (error.isUnauthorized()) {
      // Redirect to login
    } else {
      console.error('API Error:', error.message);
    }
  }
}
```

---

## Обработка ошибок

### Try-Catch паттерны

```typescript
// Базовый try-catch
async function getUser(id: number): Promise<User | null> {
  try {
    return await fetchApi<User>(`/api/users/${id}`);
  } catch (error) {
    console.error('Failed to fetch user:', error);
    return null;
  }
}

// С результатом
async function getUserSafe(id: number): Promise<Result<User, Error>> {
  try {
    const user = await fetchApi<User>(`/api/users/${id}`);
    return { ok: true, value: user };
  } catch (error) {
    return { ok: false, error: error as Error };
  }
}

// С retry
async function getUserWithRetry(id: number): Promise<User> {
  return retry(
    () => fetchApi<User>(`/api/users/${id}`),
    { retries: 3, delay: 1000 }
  );
}
```

### Error Boundary для API

```typescript
type ApiState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error; retry: () => void };

function createApiState<T>() {
  let state: ApiState<T> = { status: 'idle' };
  const listeners = new Set<(state: ApiState<T>) => void>();

  const setState = (newState: ApiState<T>) => {
    state = newState;
    listeners.forEach(listener => listener(state));
  };

  const fetch = async (fn: () => Promise<T>) => {
    setState({ status: 'loading' });
    try {
      const data = await fn();
      setState({ status: 'success', data });
    } catch (error) {
      setState({ 
        status: 'error', 
        error: error as Error,
        retry: () => fetch(fn)
      });
    }
  };

  return {
    getState: () => state,
    subscribe: (listener: (state: ApiState<T>) => void) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
    fetch,
  };
}

// Использование
const userState = createApiState<User>();
userState.fetch(() => fetchApi<User>('/api/users/1'));
```

---

## Abort Controller

### Базовое использование

```typescript
// Отмена запроса
function createFetchWithAbort() {
  const controller = new AbortController();

  const fetch = async <T>(url: string, options: RequestInit = {}): Promise<T> => {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });
    return response.json();
  };

  const abort = () => controller.abort();

  return { fetch, abort };
}

// Использование
const { fetch, abort } = createFetchWithAbort();

// Отмена при размонтировании
useEffect(() => {
  const { fetch, abort } = createFetchWithAbort();
  
  fetch<User>('/api/users/1')
    .then(setUser)
    .catch(err => {
      if (err.name === 'AbortError') {
        console.log('Request aborted');
      }
    });

  return () => abort();
}, []);
```

### Abort для всех запросов

```typescript
class ApiClient {
  private controller = new AbortController();

  async get<T>(url: string): Promise<T> {
    return this.request<T>(url, { method: 'GET' });
  }

  async post<T>(url: string, body: unknown): Promise<T> {
    return this.request<T>(url, { method: 'POST', body });
  }

  private async request<T>(url: string, options: RequestInit): Promise<T> {
    const response = await fetch(url, {
      ...options,
      signal: this.controller.signal,
    });
    return response.json();
  }

  abortAll(): void {
    this.controller.abort();
    this.controller = new AbortController();
  }
}

// Использование
const api = new ApiClient();

// Отменить все запросы
api.abortAll();
```

### Debounce + Abort

```typescript
function createSearchApi() {
  let controller: AbortController | null = null;
  let debounceTimer: NodeJS.Timeout | null = null;

  const search = async (query: string): Promise<SearchResult[]> => {
    // Отменяем предыдущий запрос
    if (controller) {
      controller.abort();
    }

    // Очищаем предыдущий таймер
    if (debounceTimer) {
      clearTimeout(debounceTimer);
    }

    // Ждём 300ms перед новым запросом
    await new Promise<void>(resolve => {
      debounceTimer = setTimeout(resolve, 300);
    });

    controller = new AbortController();

    try {
      const response = await fetch(`/api/search?q=${query}`, {
        signal: controller.signal,
      });
      return response.json();
    } catch (error) {
      if (error instanceof Error && error.name === 'AbortError') {
        return [];
      }
      throw error;
    }
  };

  const cancel = () => {
    if (controller) controller.abort();
    if (debounceTimer) clearTimeout(debounceTimer);
  };

  return { search, cancel };
}
```

---

## Retry логика

### Базовый retry

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  options: {
    retries?: number;
    delay?: number;
    maxDelay?: number;
    factor?: number;
    shouldRetry?: (error: Error) => boolean;
  } = {}
): Promise<T> {
  const {
    retries = 3,
    delay = 1000,
    maxDelay = 10000,
    factor = 2,
    shouldRetry = () => true,
  } = options;

  try {
    return await fn();
  } catch (error) {
    if (retries <= 0 || !shouldRetry(error as Error)) {
      throw error;
    }
    await sleep(Math.min(delay * factor ** (3 - retries), maxDelay));
    return retry(fn, { retries: retries - 1, delay, maxDelay, factor, shouldRetry });
  }
}

// Использование с умной логикой
const user = await retry(
  () => fetchApi<User>('/api/users/1'),
  {
    retries: 3,
    shouldRetry: (error) => {
      // Не retry'им 4xx ошибки
      if (error instanceof ApiError && error.status < 500) {
        return false;
      }
      return true;
    },
  }
);
```

### Retry с бэк-оффом

```typescript
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelay?: number;
    maxDelay?: number;
  } = {}
): Promise<T> {
  const { maxRetries = 5, baseDelay = 100, maxDelay = 10000 } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }

      // Экспоненциальный бэк-офф с джиттером
      const delay = Math.min(
        baseDelay * 2 ** attempt + Math.random() * 1000,
        maxDelay
      );
      
      await sleep(delay);
    }
  }

  throw new Error('Unreachable');
}
```

---

## Кэширование

### Простое кэширование

```typescript
class Cache<T> {
  private cache = new Map<string, { value: T; timestamp: number }>();
  private ttl: number;

  constructor(ttl: number = 5 * 60 * 1000) {
    this.ttl = ttl;
  }

  get(key: string): T | null {
    const item = this.cache.get(key);
    if (!item) return null;

    if (Date.now() - item.timestamp > this.ttl) {
      this.cache.delete(key);
      return null;
    }

    return item.value;
  }

  set(key: string, value: T): void {
    this.cache.set(key, { value, timestamp: Date.now() });
  }

  delete(key: string): void {
    this.cache.delete(key);
  }

  clear(): void {
    this.cache.clear();
  }
}

// Использование с fetch
const cache = new Cache<User>(5 * 60 * 1000);

async function fetchUserWithCache(id: number): Promise<User> {
  const cached = cache.get(`user:${id}`);
  if (cached) return cached;

  const user = await fetchApi<User>(`/api/users/${id}`);
  cache.set(`user:${id}`, user);
  return user;
}
```

### Кэширование с инвалидацией

```typescript
type QueryKey = string[];

interface QueryCache {
  get<T>(key: QueryKey): T | null;
  set<T>(key: QueryKey, value: T): void;
  invalidate(key: QueryKey): void;
  invalidatePrefix(prefix: QueryKey): void;
}

function createQueryCache(): QueryCache {
  const cache = new Map<string, unknown>();

  const serializeKey = (key: QueryKey): string => key.join(':');

  return {
    get<T>(key: QueryKey): T | null {
      return (cache.get(serializeKey(key)) as T) ?? null;
    },
    set<T>(key: QueryKey, value: T): void {
      cache.set(serializeKey(key), value);
    },
    invalidate(key: QueryKey): void {
      cache.delete(serializeKey(key));
    },
    invalidatePrefix(prefix: QueryKey): void {
      const prefixStr = prefix.join(':');
      for (const key of cache.keys()) {
        if (key.startsWith(prefixStr)) {
          cache.delete(key);
        }
      }
    },
  };
}

// Использование
const queryCache = createQueryCache();

// Fetch с кэшем
async function fetchUser(id: number): Promise<User> {
  const cached = queryCache.get<User>(['users', id.toString()]);
  if (cached) return cached;

  const user = await fetchApi<User>(`/api/users/${id}`);
  queryCache.set(['users', id.toString()], user);
  return user;
}

// Инвалидация после мутации
async function updateUser(id: number, data: Partial<User>): Promise<User> {
  const user = await fetchApi<User>(`/api/users/${id}`, {
    method: 'PUT',
    body: data,
  });
  
  // Инвалидируем кэш для этого пользователя
  queryCache.invalidate(['users', id.toString()]);
  // Или все пользователи
  queryCache.invalidatePrefix(['users']);
  
  return user;
}
```

---

## Identity Keys

### Генерация ключей

```typescript
// UUID для уникальных идентификаторов
function createIdentityKey(): string {
  return crypto.randomUUID();
}

// Композитный ключ из нескольких полей
function createCompositeKey(...parts: (string | number | undefined | null)[]): string {
  return parts.filter(p => p != null).join(':');
}

// Использование
const userKey = createCompositeKey('user', userId);  // "user:123"
const postKey = createCompositeKey('user', userId, 'posts', postId);  // "user:123:posts:456"
```

### Identity Map паттерн

```typescript
class IdentityMap<T extends { id: string | number }> {
  private map = new Map<string, T>();

  get(id: string | number): T | null {
    return this.map.get(String(id)) ?? null;
  }

  set(entity: T): void {
    this.map.set(String(entity.id), entity);
  }

  has(id: string | number): boolean {
    return this.map.has(String(id));
  }

  delete(id: string | number): void {
    this.map.delete(String(id));
  }

  getAll(): T[] {
    return Array.from(this.map.values());
  }

  clear(): void {
    this.map.clear();
  }
}

// Использование
const userIdentityMap = new IdentityMap<User>();

async function getUser(id: number): Promise<User> {
  // Проверяем в Identity Map
  const cached = userIdentityMap.get(id);
  if (cached) return cached;

  // Фетчим из API
  const user = await fetchApi<User>(`/api/users/${id}`);
  
  // Сохраняем в Identity Map
  userIdentityMap.set(user);
  
  return user;
}
```

### Нормализация данных

```typescript
interface NormalizedData<T> {
  entities: Record<string, T>;
  order: string[];
}

function normalize<T extends { id: string | number }>(
  items: T[]
): NormalizedData<T> {
  const entities: Record<string, T> = {};
  const order: string[] = [];

  for (const item of items) {
    const id = String(item.id);
    entities[id] = item;
    order.push(id);
  }

  return { entities, order };
}

function denormalize<T>(normalized: NormalizedData<T>): T[] {
  return normalized.order.map(id => normalized.entities[id]);
}

// Использование с вложенными данными
interface Post {
  id: number;
  title: string;
  authorId: number;
}

interface NormalizedState {
  users: NormalizedData<User>;
  posts: NormalizedData<Post>;
}

function normalizePosts(posts: Post[], users: User[]): NormalizedState {
  return {
    users: normalize(users),
    posts: normalize(posts),
  };
}
```

---

## React Query паттерн

### Базовая реализация

```typescript
type QueryStatus = 'idle' | 'loading' | 'success' | 'error';

interface QueryState<T> {
  status: QueryStatus;
  data: T | null;
  error: Error | null;
  isFetching: boolean;
  updatedAt: number;
}

interface QueryConfig {
  staleTime?: number;
  cacheTime?: number;
  retry?: number;
  retryDelay?: number;
}

class QueryClient {
  private queries = new Map<string, QueryState<unknown>>();
  private listeners = new Set<() => void>();
  private config: QueryConfig;

  constructor(config: QueryConfig = {}) {
    this.config = {
      staleTime: 5 * 60 * 1000,
      cacheTime: 10 * 60 * 1000,
      retry: 3,
      retryDelay: 1000,
      ...config,
    };
  }

  private serializeKey(key: string[]): string {
    return key.join(':');
  }

  async fetch<T>(
    queryKey: string[],
    fetchFn: () => Promise<T>
  ): Promise<QueryState<T>> {
    const key = this.serializeKey(queryKey);
    const existing = this.queries.get(key) as QueryState<T> | undefined;

    // Проверяем есть ли свежие данные
    if (existing?.status === 'success') {
      const isStale = Date.now() - existing.updatedAt > this.config.staleTime!;
      if (!isStale) {
        return existing;
      }
    }

    // Обновляем состояние
    this.setState(key, {
      ...existing,
      status: 'loading',
      isFetching: true,
    } as QueryState<T>);

    try {
      const data = await retry(
        () => fetchFn(),
        { retries: this.config.retry!, delay: this.config.retryDelay! }
      );

      this.setState(key, {
        status: 'success',
        data,
        error: null,
        isFetching: false,
        updatedAt: Date.now(),
      } as QueryState<T>);

      return this.getState<T>(key);
    } catch (error) {
      this.setState(key, {
        ...existing,
        status: 'error',
        error: error as Error,
        isFetching: false,
      } as QueryState<T>);

      throw error;
    }
  }

  private getState<T>(key: string): QueryState<T> {
    return (this.queries.get(key) ?? {
      status: 'idle',
      data: null,
      error: null,
      isFetching: false,
      updatedAt: 0,
    }) as QueryState<T>;
  }

  private setState<T>(key: string, state: QueryState<T>): void {
    this.queries.set(key, state as QueryState<unknown>);
    this.listeners.forEach(listener => listener());
  }

  subscribe(listener: () => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  invalidateQueries(queryKey: string[]): void {
    const prefix = this.serializeKey(queryKey);
    for (const key of this.queries.keys()) {
      if (key.startsWith(prefix)) {
        const state = this.queries.get(key);
        if (state) {
          this.queries.set(key, { ...state, updatedAt: 0 });
        }
      }
    }
    this.listeners.forEach(listener => listener());
  }
}

// Использование
const queryClient = new QueryClient({
  staleTime: 5 * 60 * 1000,
  retry: 3,
});

// Fetch данных
const state = await queryClient.fetch(['users', '1'], () =>
  fetchApi<User>('/api/users/1')
);

// Инвалидация
queryClient.invalidateQueries(['users']);
```

---

## Полные примеры

### API Client класс

```typescript
interface ApiClientConfig {
  baseUrl: string;
  token?: string;
  timeout?: number;
}

class ApiClient {
  private baseUrl: string;
  private token?: string;
  private timeout: number;
  private abortController = new AbortController();

  constructor(config: ApiClientConfig) {
    this.baseUrl = config.baseUrl;
    this.token = config.token;
    this.timeout = config.timeout ?? 30000;
  }

  private async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseUrl}${endpoint}`;

    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...(this.token && { Authorization: `Bearer ${this.token}` }),
      ...options.headers,
    };

    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);

    try {
      const response = await fetch(url, {
        ...options,
        headers,
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new ApiError(
          response.statusText,
          response.status,
          await response.text().catch(() => undefined),
          url
        );
      }

      return response.json() as Promise<T>;
    } catch (error) {
      clearTimeout(timeoutId);
      
      if (error instanceof Error && error.name === 'AbortError') {
        throw new Error('Request timeout');
      }
      
      throw error;
    }
  }

  async get<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, { method: 'GET' });
  }

  async post<T>(endpoint: string, body: unknown): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'POST',
      body: JSON.stringify(body),
    });
  }

  async put<T>(endpoint: string, body: unknown): Promise<T> {
    return this.request<T>(endpoint, {
      method: 'PUT',
      body: JSON.stringify(body),
    });
  }

  async delete<T>(endpoint: string): Promise<T> {
    return this.request<T>(endpoint, { method: 'DELETE' });
  }

  abortAll(): void {
    this.abortController.abort();
    this.abortController = new AbortController();
  }

  setToken(token: string): void {
    this.token = token;
  }
}

// Использование
const api = new ApiClient({
  baseUrl: 'https://api.example.com',
  token: 'my-token',
  timeout: 30000,
});

const user = await api.get<User>('/users/1');
```

---

## 🔗 Связанные заметки

- [[TypeScript-MOC]] — индекс раздела
- [[TypeScript-Functions]] — полезные функции (retry, debounce)
- [[TypeScript-Patterns]] — паттерны (Result, Option)

---

> [!TIP] Совет
> 4. **Кэшируйте с TTL** — инвалидируйте кэш после мутаций
> 5. **Identity keys для нормализации** — избегайте дублирования данных
