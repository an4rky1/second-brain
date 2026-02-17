---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - basics
aliases:
  - TS Basics
  - TypeScript Основы
related:
  - TypeScript-MOC
  - TypeScript-Tsconfig
  - TypeScript-DataFetching
---

# TypeScript — Основы и синтаксис

> [!SUMMARY] Обзор
> Базовый синтаксис TypeScript, типы данных, интерфейсы и дженерики.

---

## Примитивные типы

```typescript
let name: string = "John";
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;
let unique: symbol = Symbol("id");
let big: bigint = 9007199254740991n;
```

## Специальные типы

```typescript
let anyValue: any = "anything";           // Отключает проверку типов
let unknownValue: unknown = "anything";   // Требует проверку типа
let neverValue: never;                    // Никогда не возвращается
let voidValue: void = undefined;          // Для функций без return
```

## Массивы и кортежи

```typescript
// Массивы
let numbers: number[] = [1, 2, 3];
let genericArray: Array<string> = ["a", "b"];
const readonlyArray: readonly number[] = [1, 2, 3];

// Кортежи (фиксированная длина и типы)
let tuple: [string, number] = ["hello", 42];
let namedTuple: [name: string, age: number] = ["John", 30];

// Кортежи с rest
type StringAndMore = [string, ...number[]];
const valid: StringAndMore = ["hello", 1, 2, 3];

// Опциональные элементы кортежа
type OptionalTuple = [string, number?];
```

## Enum

```typescript
// Numeric enum (по умолчанию с 0)
enum Status {
  Pending,    // 0
  Approved,   // 1
  Rejected    // 2
}

// С явными значениями
enum HttpStatus {
  Ok = 200,
  NotFound = 404,
  ServerError = 500
}

// String enum
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT"
}

// Const enum (удаляется при компиляции)
const enum Colors {
  Red = "#ff0000",
  Green = "#00ff00",
  Blue = "#0000ff"
}

// Reverse mapping (только для numeric)
const status = Status.Pending;
const statusName = Status[status]; // "Pending"
```

## Union и Intersection

```typescript
// Union (ИЛИ)
let id: string | number;
id = "abc123";  // ✅
id = 123;       // ✅

// Intersection (И)
type A = { a: string; common: number };
type B = { b: number; common: number };
type C = A & B;  // { a: string; b: number; common: number }

// Discriminated unions
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "keydown"; key: string }
  | { type: "submit"; formData: FormData };

function handleEvent(event: Event) {
  switch (event.type) {
    case "click":
      console.log(event.x, event.y);  // type narrowed
      break;
    case "keydown":
      console.log(event.key);
      break;
  }
}
```

## Type Aliases vs Interfaces

```typescript
// Type Aliases
type User = {
  id: number;
  name: string;
  email?: string;
};

type Admin = User & { role: "admin" };
type ID = string | number;

// Interfaces
interface IUser {
  id: number;
  name: string;
  readonly createdAt: Date;
  greet(): string;
}

interface IAdmin extends IUser {
  role: "admin";
}

// Declaration merging (только interface)
interface IUser {
  age: number;
}
// IUser теперь имеет: id, name, createdAt, greet, age
```

| Type | Interface |
|------|-----------|
| Примитивы, union, intersection | Только объекты, классы |
| `type A = B & C` | `interface A extends B, C` |
| Нельзя переопределить | Можно мержить |
| `type A = string \| number` | ❌ |

**Правило:** `interface` для объектов и классов, `type` для union/intersection/primitives.

---

## Дженерики (Generics)

```typescript
// Базовый дженерик
function identity<T>(arg: T): T {
  return arg;
}
identity<string>("hello");
identity<number>(42);

// Дженерик с ограничением
function logLength<T extends { length: number }>(arg: T): number {
  return arg.length;
}
logLength("hello"); // 5
logLength([1, 2, 3]); // 3

// Дженерик интерфейс
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

const userResponse: ApiResponse<User> = {
  data: { id: 1, name: "John" },
  status: 200,
  message: "OK"
};

// Дженерик класс
class Container<T> {
  constructor(public value: T) {}
  getValue(): T {
    return this.value;
  }
}
```

---

## Type Guards

```typescript
// typeof guard
function printId(id: number | string) {
  if (typeof id === "string") {
    console.log(id.toUpperCase());
  } else {
    console.log(id);
  }
}

// instanceof guard
function printDate(date: Date | string) {
  if (date instanceof Date) {
    console.log(date.toISOString());
  } else {
    console.log(date);
  }
}

// in guard
interface Cat { meow(): void }
interface Dog { bark(): void }

function makeSound(animal: Cat | Dog) {
  if ("meow" in animal) {
    animal.meow();
  } else {
    animal.bark();
  }
}

// Custom type guard
function isString(value: unknown): value is string {
  return typeof value === "string";
}
```

---

## Быстрый старт

```bash
# Установка
npm install -D typescript @types/node

# Инициализация
npx tsc --init

# Компиляция
npx tsc
npx tsc --watch

# Запуск через ts-node
npx ts-node file.ts

# Проверка типов без компиляции
npx tsc --noEmit
```

---

## Best Practices

### ✅ Делать

```typescript
// 1. Используйте strict mode
// tsconfig.json: "strict": true

// 2. Избегайте any, используйте unknown
function process(value: unknown) {
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  throw new Error("Invalid type");
}

// 3. Используйте const assertions для литералов
const roles = ["admin", "user"] as const;
type Role = typeof roles[number]; // "admin" | "user"

// 4. Типизируйте промисы явно
async function fetchUser(id: number): Promise<User> {
  // ...
}

// 5. Используйте discriminated unions
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

// 6. Экспортируйте типы из отдельных файлов
// types/user.ts
export interface User { /* ... */ }
// Использование: import type { User } from './types/user';
```

### ❌ Не делать

```typescript
// 1. Избегайте any
let data: any; // ❌
let data: unknown; // ✅

// 2. Не используйте non-null assertion без необходимости
let value: string | null = null;
value!.length; // ❌ (может упасть)
if (value) value.length; // ✅

// 3. Не дублируйте типы
interface User {
  id: number;
  name: string;
}
// ❌
type UserDTO = {
  id: number;
  name: string;
};
// ✅
type UserDTO = User;
```

---

## Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Property 'x' does not exist on type 'T'` | T не имеет ограничения | Добавьте `extends { x: type }` |
| `Type 'any' is not assignable` | Strict режим | Избегайте any, используйте unknown |
| `Module has no default export` | ES modules vs CommonJS | Используйте `import * as` или включите `esModuleInterop` |
| `Implicit any` | Не выведен тип | Добавьте явную аннотацию |
| `Type instantiation is excessively deep` | Сложные дженерики | Упростите типы или увеличьте `--maxNodeModuleJsDepth` |

---

## 🔗 Связанные заметки

- [[TypeScript-MOC]] — индекс раздела
- [[TypeScript-Tsconfig]] — конфигурация
- [[TypeScript-Utility-Types]] — utility типы
- [[TypeScript-DataFetching]] — fetch, API

---

> [!TIP] Совет
>
> 1. **Начинайте с простых типов** — не усложняйте сразу
> 2. **Используйте strict режим** — ловит ошибки на этапе компиляции
> 3. **Типизируйте границы приложения** — API responses, events, props
> 4. **Избегайте any как чумы** — unknown всегда лучше
> 5. **Дженерики — ваш друг** — но используйте умеренно
