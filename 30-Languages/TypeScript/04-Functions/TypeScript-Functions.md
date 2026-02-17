---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - functions
  - utilities
aliases:
  - TS Functions
  - TypeScript Helpers
related:
  - TypeScript-MOC
  - TypeScript-Utility-Types
  - TypeScript-DataFetching
---

# TypeScript — Полезные функции и хелперы

> [!SUMMARY] Обзор
> Готовые функции и утилиты для повседневной разработки: форматирование, валидация, асинхронность, работа с объектами.

---

## Форматирование

### Деньги и валюта

```typescript
// Форматирование валюты
function formatCurrency(
  amount: number,
  currency: string = 'USD',
  locale: string = 'en-US'
): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(amount);
}

formatCurrency(1234.56, 'USD', 'en-US');  // "$1,234.56"
formatCurrency(1234.56, 'EUR', 'de-DE');  // "1.234,56 €"
formatCurrency(1234.56, 'RUB', 'ru-RU');  // "1 234,56 ₽"

// Сокращённое форматирование (тыс., млн., млрд.)
function formatCompactCurrency(
  amount: number,
  currency: string = 'USD',
  locale: string = 'en-US'
): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    notation: 'compact',
    compactDisplay: 'short',
    maximumFractionDigits: 1,
  }).format(amount);
}

formatCompactCurrency(1234567, 'USD', 'en-US');  // "$1.2M"
formatCompactCurrency(1234567, 'RUB', 'ru-RU');  // "1,2 млн ₽"
```

### Даты и время

```typescript
// Форматирование даты
function formatDate(
  date: Date | string | number,
  options?: Intl.DateTimeFormatOptions,
  locale: string = 'ru-RU'
): string {
  const d = new Date(date);
  return new Intl.DateTimeFormat(locale, options).format(d);
}

formatDate(new Date());  // "17.02.2026"
formatDate(new Date(), { year: 'numeric', month: 'long', day: 'numeric' });
// "17 февраля 2026 г."

// Относительное время ("5 минут назад")
function formatRelativeTime(
  date: Date | string | number,
  locale: string = 'ru-RU'
): string {
  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
  const now = new Date();
  const then = new Date(date);
  const diffMs = then.getTime() - now.getTime();
  const diffSecs = Math.round(diffMs / 1000);
  const diffMins = Math.round(diffSecs / 60);
  const diffHours = Math.round(diffMins / 60);
  const diffDays = Math.round(diffHours / 24);
  const diffWeeks = Math.round(diffDays / 7);
  const diffMonths = Math.round(diffDays / 30);
  const diffYears = Math.round(diffDays / 365);

  if (Math.abs(diffSecs) < 60) return rtf.format(diffSecs, 'second');
  if (Math.abs(diffMins) < 60) return rtf.format(diffMins, 'minute');
  if (Math.abs(diffHours) < 24) return rtf.format(diffHours, 'hour');
  if (Math.abs(diffDays) < 7) return rtf.format(diffDays, 'day');
  if (Math.abs(diffWeeks) < 4) return rtf.format(diffWeeks, 'week');
  if (Math.abs(diffMonths) < 12) return rtf.format(diffMonths, 'month');
  return rtf.format(diffYears, 'year');
}

formatRelativeTime(Date.now() - 5 * 60 * 1000);  // "5 минут назад"
formatRelativeTime(Date.now() + 2 * 60 * 60 * 1000);  // "через 2 часа"

// Полная дата с временем
function formatDateTime(
  date: Date | string | number,
  locale: string = 'ru-RU'
): string {
  return formatDate(date, {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }, locale);
}

formatDateTime(new Date());  // "17.02.2026, 14:30"
```

### Числа и размеры

```typescript
// Форматирование чисел с разделителями
function formatNumber(
  value: number,
  options?: Intl.NumberFormatOptions,
  locale: string = 'ru-RU'
): string {
  return new Intl.NumberFormat(locale, options).format(value);
}

formatNumber(1234567.89);  // "1 234 567,89"
formatNumber(0.5, { style: 'percent' });  // "50 %"

// Сокращённое форматирование (1.2K, 3.4M)
function formatCompactNumber(value: number, locale: string = 'en-US'): string {
  return new Intl.NumberFormat(locale, {
    notation: 'compact',
    compactDisplay: 'short',
    maximumFractionDigits: 1,
  }).format(value);
}

formatCompactNumber(1234);      // "1.2K"
formatCompactNumber(1234567);   // "1.2M"
formatCompactNumber(1234567890); // "1.2B"

// Форматирование размеров файлов
function formatFileSize(bytes: number): string {
  const units = ['B', 'KB', 'MB', 'GB', 'TB', 'PB'];
  let unitIndex = 0;
  let size = bytes;

  while (size >= 1024 && unitIndex < units.length - 1) {
    size /= 1024;
    unitIndex++;
  }

  return `${size.toFixed(2)} ${units[unitIndex]}`;
}

formatFileSize(1500);        // "1.46 KB"
formatFileSize(1500000);     // "1.43 MB"
formatFileSize(1500000000);  // "1.40 GB"

// Форматирование процентов
function formatPercent(value: number, locale: string = 'ru-RU'): string {
  return new Intl.NumberFormat(locale, {
    style: 'percent',
    minimumFractionDigits: 1,
    maximumFractionDigits: 2,
  }).format(value / 100);
}
```

---

## Валидация и парсинг

```typescript
// Безопасный JSON parse
function safeJsonParse<T>(json: string, fallback: T): T {
  try {
    return JSON.parse(json) as T;
  } catch {
    return fallback;
  }
}

// Result type для обработки ошибок
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function parseJson<T>(json: string): Result<T, SyntaxError> {
  try {
    return { ok: true, value: JSON.parse(json) as T };
  } catch (e) {
    return { ok: false, error: e as SyntaxError };
  }
}

// Валидация email
function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

// Валидация URL
function isValidUrl(url: string): boolean {
  try {
    new URL(url);
    return true;
  } catch {
    return false;
  }
}

// Валидация UUID
function isValidUUID(uuid: string): boolean {
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
  return uuidRegex.test(uuid);
}

// Парсинг целого числа
function parseIntSafe(value: string, radix = 10): number | null {
  const parsed = parseInt(value, radix);
  return isNaN(parsed) ? null : parsed;
}

// Парсинг числа с плавающей точкой
function parseFloatSafe(value: string): number | null {
  const parsed = parseFloat(value);
  return isNaN(parsed) ? null : parsed;
}
```

---

## Type Guards

```typescript
// Проверка на null/undefined
function isDefined<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}

function isNullish(value: unknown): value is null | undefined {
  return value === null || value === undefined;
}

// Проверка типов
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function isNumber(value: unknown): value is number {
  return typeof value === 'number' && !isNaN(value);
}

function isBoolean(value: unknown): value is boolean {
  return typeof value === 'boolean';
}

function isFunction(value: unknown): value is Function {
  return typeof value === 'function';
}

function isArray<T>(value: unknown, guard?: (v: unknown) => v is T): value is T[] {
  if (!Array.isArray(value)) return false;
  if (guard) return value.every(guard);
  return true;
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value);
}

function isDate(value: unknown): value is Date {
  return value instanceof Date && !isNaN(value.getTime());
}

function isPromise(value: unknown): value is Promise<unknown> {
  return value instanceof Promise;
}

function isMap(value: unknown): value is Map<unknown, unknown> {
  return value instanceof Map;
}

function isSet(value: unknown): value is Set<unknown> {
  return value instanceof Set;
}

// Проверка на пустое значение
function isEmpty(value: unknown): boolean {
  if (value === null || value === undefined) return true;
  if (typeof value === 'string') return value.length === 0;
  if (Array.isArray(value)) return value.length === 0;
  if (typeof value === 'object') return Object.keys(value).length === 0;
  return false;
}

// Assert функции
function assert(condition: unknown, message?: string): asserts condition {
  if (!condition) {
    throw new Error(message ?? 'Assertion failed');
  }
}

function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function assertDefined<T>(value: T | null | undefined, message?: string): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(message ?? 'Value is null or undefined');
  }
}
```

---

## Работа с объектами

```typescript
// Pick выбранных ключей
function pick<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;
  for (const key of keys) {
    if (key in obj) {
      result[key] = obj[key];
    }
  }
  return result;
}

// Omit исключённых ключей
function omit<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Omit<T, K> {
  const result = { ...obj };
  for (const key of keys) {
    delete (result as Record<K, unknown>)[key];
  }
  return result as Omit<T, K>;
}

// Глубокое клонирование
function deepClone<T>(obj: T): T {
  if (obj === null || typeof obj !== 'object') return obj;
  if (obj instanceof Date) return new Date(obj.getTime()) as T;
  if (obj instanceof Array) return obj.map(item => deepClone(item)) as T;
  if (obj instanceof Object) {
    const cloned = {} as T;
    for (const key in obj) {
      if (Object.hasOwn(obj, key)) {
        (cloned as Record<string, unknown>)[key] = deepClone((obj as Record<string, unknown>)[key]);
      }
    }
    return cloned;
  }
  return obj;
}

// Глубокое слияние
function deepMerge<T extends object, U extends object>(
  target: T,
  source: U
): T & U {
  const result = { ...target } as T & U;
  
  for (const key in source) {
    if (Object.hasOwn(source, key)) {
      const sourceValue = source[key];
      const targetValue = target[key as keyof T];
      
      if (
        sourceValue instanceof Object &&
        targetValue instanceof Object &&
        !Array.isArray(sourceValue) &&
        !Array.isArray(targetValue) &&
        targetValue !== null
      ) {
        (result as Record<string, unknown>)[key] = deepMerge(
          targetValue as object,
          sourceValue as object
        );
      } else {
        (result as Record<string, unknown>)[key] = sourceValue;
      }
    }
  }
  
  return result;
}

// Инвертирование объекта
function invert<T extends string | number | symbol, K extends string | number | symbol>(
  obj: Record<T, K>
): Record<K, T> {
  const result = {} as Record<K, T>;
  for (const key in obj) {
    if (Object.hasOwn(obj, key)) {
      result[obj[key]] = key as T;
    }
  }
  return result;
}

// Группировка массива объектов
function groupBy<T, K extends keyof T | ((item: T) => string)>(
  array: T[],
  key: K
): Record<string, T[]> {
  return array.reduce((result, item) => {
    const groupKey = typeof key === 'function' 
      ? key(item) 
      : String(item[key as keyof T]);
    if (!result[groupKey]) {
      result[groupKey] = [];
    }
    result[groupKey].push(item);
    return result;
  }, {} as Record<string, T[]>);
}

// Уникализация массива объектов по ключу
function uniqueBy<T>(array: T[], key: keyof T | ((item: T) => string)): T[] {
  const seen = new Set<string>();
  return array.filter(item => {
    const keyValue = typeof key === 'function' 
      ? key(item) 
      : String(item[key]);
    if (seen.has(keyValue)) return false;
    seen.add(keyValue);
    return true;
  });
}

// Safe доступ к вложенным свойствам
function getNested<T>(obj: Record<string, any>, path: string): any {
  return path.split('.').reduce((acc, key) => acc?.[key], obj);
}
```

---

## Асинхронные утилиты

```typescript
// Sleep
function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Retry с экспоненциальной задержкой
async function retry<T>(
  fn: () => Promise<T>,
  options: {
    retries?: number;
    delay?: number;
    maxDelay?: number;
    factor?: number;
  } = {}
): Promise<T> {
  const {
    retries = 3,
    delay = 1000,
    maxDelay = 10000,
    factor = 2,
  } = options;

  try {
    return await fn();
  } catch (error) {
    if (retries <= 0) throw error;
    await sleep(Math.min(delay * factor ** (3 - retries), maxDelay));
    return retry(fn, { retries: retries - 1, delay, maxDelay, factor });
  }
}

// Timeout для промиса
async function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
  timeoutError?: Error
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(timeoutError ?? new Error('Timeout')), ms)
  );
  return Promise.race([promise, timeout]);
}

// Debounce
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): ((...args: Parameters<T>) => void) & { cancel: () => void; flush: (...args: Parameters<T>) => ReturnType<T> } {
  let timeoutId: NodeJS.Timeout;
  
  const debounced = (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
  
  debounced.cancel = () => clearTimeout(timeoutId);
  debounced.flush = (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    return fn(...args);
  };
  
  return debounced;
}

// Throttle
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;
  return (...args: Parameters<T>) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

// Memoize
function memoize<T extends (...args: any[]) => any>(
  fn: T,
  resolver?: (...args: Parameters<T>) => string
): T {
  const cache = new Map<string, ReturnType<T>>();
  return ((...args: Parameters<T>) => {
    const key = resolver ? resolver(...args) : JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// Parallel execution с ограничением
async function mapWithConcurrency<T, R>(
  items: T[],
  fn: (item: T, index: number) => Promise<R>,
  concurrency: number = 5
): Promise<R[]> {
  const results: R[] = [];
  let index = 0;

  async function worker() {
    while (index < items.length) {
      const currentIndex = index++;
      const result = await fn(items[currentIndex], currentIndex);
      results[currentIndex] = result;
    }
  }

  await Promise.all(Array.from({ length: concurrency }, worker));
  return results;
}

// Batch processing
async function processInBatches<T, R>(
  items: T[],
  fn: (item: T) => Promise<R>,
  batchSize: number = 10
): Promise<R[]> {
  const results: R[] = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(fn));
    results.push(...batchResults);
  }
  return results;
}
```

---

## 🔗 Связанные заметки

- [[TypeScript-MOC]] — индекс раздела
- [[TypeScript-Utility-Types]] — типы и дженерики
- [[TypeScript-Patterns]] — паттерны
- [[TypeScript-DataFetching]] — fetch, API

---

> [!TIP] Совет
> 4. **Debounce/throttle для UI событий** — оптимизация производительности
> 5. **Retry для сетевых запросов** — устойчивость к временным ошибкам
