---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - types
  - generics
aliases:
  - TS Utility Types
  - TypeScript Generics
related:
  - TypeScript-MOC
  - TypeScript-Basics
  - TypeScript-DataFetching
---

# TypeScript — Utility Types и дженерики

> [!SUMMARY] Обзор
> Встроенные utility types, кастомные типы, дженерики и продвинутые техники работы с типами.

---

## Встроенные Utility Types

### Basic Utilities

```typescript
// Partial<T> — все поля опциональные
interface User { id: number; name: string; email: string; }
type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; }

// Required<T> — все поля обязательные
type RequiredUser = Required<PartialUser>;
// { id: number; name: string; email: string; }

// Readonly<T> — все поля readonly
type ReadonlyUser = Readonly<User>;
// { readonly id: number; readonly name: string; readonly email: string; }

// Record<K, T> — объект с ключами K и значениями T
type UserRoles = Record<string, boolean>;
type Permissions = Record<'read' | 'write' | 'delete', boolean>;

// Pick<T, K> — выбрать поля K из T
type UserId = Pick<User, 'id'>;
type UserIdentity = Pick<User, 'id' | 'name'>;

// Omit<T, K> — исключить поля K из T
type UserNoId = Omit<User, 'id'>;
type UserNoEmail = Omit<User, 'email'>;

// Exclude<T, U> — исключить U из T
type OnlyStrings = Exclude<string | number | boolean, number | boolean>;
// string

// Extract<T, U> — извлечь U из T
type OnlyNumbers = Extract<string | number | boolean, number>;
// number

// NonNullable<T> — исключить null и undefined
type NotNull = NonNullable<string | null | undefined>;
// string

// Parameters<T> — параметры функции как кортеж
type FnParams = Parameters<(a: string, b: number) => void>;
// [a: string, b: number]

// ReturnType<T> — возвращаемое значение функции
type FnReturn = ReturnType<() => string>;
// string

// InstanceType<T> — тип экземпляра класса
class User {}
type UserType = InstanceType<typeof User>;

// ConstructorParameters<T> — параметры конструктора
class MyClass {
  constructor(public a: string, public b: number) {}
}
type MyParams = ConstructorParameters<typeof MyClass>;
// [a: string, b: number]

// Awaited<T> — разворачивает Promise (как await)
type AsyncValue = Awaited<Promise<string>>;  // string
type NestedAsync = Awaited<Promise<Promise<number>>>;  // number
```

### Template Literal Types

```typescript
// Конкатенация строк
type Greeting = `Hello ${string}`;

// Union в template
type EventName = `on${'Click' | 'Hover' | 'Focus'}`;
// "onClick" | "onHover" | "onFocus"

// С несколькими union
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type FullEndpoint = `${HttpMethod} /api/${string}`;

// Capitalize, Lowercase, Uppercase, Uncapitalize
type A = Capitalize<'hello'>;    // "Hello"
type B = Uppercase<'hello'>;     // "HELLO"
type C = Lowercase<'HELLO'>;     // "hello"
type D = Uncapitalize<'Hello'>;  // "hello"

// Extract patterns
type GetEventName<T extends string> = T extends `on${infer Name}` ? Name : never;
type A = GetEventName<'onClick'>;  // "Click"
```

---

## Кастомные Utility Types

### Deep Utilities

```typescript
// DeepPartial — рекурсивно все поля опциональные
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface User {
  name: string;
  address: { city: string; zip: number; };
}
type PartialUser = DeepPartial<User>;
// { name?: string; address?: { city?: string; zip?: number; }; }

// DeepRequired — рекурсивно все поля обязательные
type DeepRequired<T> = {
  [P in keyof T]-?: T[P] extends object ? DeepRequired<T[P]> : T[P];
};

// DeepReadonly — рекурсивно все поля readonly
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

// Writable — убрать readonly со всех уровней
type Writable<T> = {
  -readonly [P in keyof T]: T[P] extends object ? Writable<T[P]> : T[P];
};
```

### Nullable Utilities

```typescript
// Nullable<T> — добавить null
type Nullable<T> = T | null;
type A = Nullable<string>;  // string | null

// Maybe<T> — добавить null и undefined
type Maybe<T> = T | null | undefined;
type A = Maybe<string>;  // string | null | undefined

// Defined<T> — убрать undefined
type Defined<T> = Exclude<T, undefined>;

// NotNull<T> — убрать null
type NotNull<T> = Exclude<T, null>;
```

### Function Utilities

```typescript
// AsyncFunction<T> — сделать функцию асинхронной
type AsyncFunction<T extends (...args: any[]) => any> =
  (...args: Parameters<T>) => Promise<ReturnType<T>>;

type SyncFn = (a: string) => number;
type AsyncFn = AsyncFunction<SyncFn>;  // (a: string) => Promise<number>

// SyncFunction<T> — unwrap promise из возвращаемого значения
type SyncFunction<T extends (...args: any[]) => any> =
  (...args: Parameters<T>) => Awaited<ReturnType<T>>;

// FirstArgument<T> — первый параметр
type FirstArgument<T extends (...args: any[]) => any> = Parameters<T>[0];

// WithoutFirst<T> — функция без первого параметра
type WithoutFirst<T extends (...args: any[]) => any> =
  (...args: Parameters<T>.slice(1)) => ReturnType<T>;

// WithoutLast<T> — функция без последнего параметра
type WithoutLast<T extends (...args: any[]) => any> =
  T extends (...args: [...infer Args, any]) => infer R 
    ? (...args: Args) => R 
    : never;
```

### Array Utilities

```typescript
// ElementOf<T> — тип элемента массива
type ElementOf<T extends readonly any[]> = T[number];
type A = ElementOf<string[]>;  // string
type B = ElementOf<readonly [1, 2, 3]>;  // 1 | 2 | 3

// Mutable<T> — убрать readonly
type Mutable<T> = { -readonly [P in keyof T]: T[P]; };
type A = Mutable<readonly number[]>;  // number[]

// Immutable<T> — добавить readonly
type Immutable<T> = {
  readonly [P in keyof T]: T[P] extends object ? Immutable<T[P]> : T[P];
};

// Tuple<T, N> — кортеж длины N
type Tuple<T, N extends number, R extends readonly T[] = []> =
  R['length'] extends N ? R : Tuple<T, N, readonly [...R, T]>;
type A = Tuple<string, 3>;  // readonly [string, string, string]

// DropFirst<T> — убрать первый элемент
type DropFirst<T extends readonly any[]> =
  T extends readonly [any, ...infer Rest] ? Rest : never;
type A = DropFirst<[1, 2, 3]>;  // [2, 3]

// DropLast<T> — убрать последний элемент
type DropLast<T extends readonly any[]> =
  T extends readonly [...infer Rest, any] ? Rest : never;
type A = DropLast<[1, 2, 3]>;  // [1, 2]

// Reverse<T> — перевернуть кортеж
type Reverse<T extends readonly any[], R extends readonly any[] = []> =
  T extends readonly [infer First, ...infer Rest]
    ? Reverse<Rest, readonly [First, ...R]>
    : R;
type A = Reverse<[1, 2, 3]>;  // [3, 2, 1]

// Unique<T> — уникальные элементы кортежа
type Unique<T extends readonly any[], R extends readonly any[] = []> =
  T extends readonly [infer First, ...infer Rest]
    ? First extends R[number]
      ? Unique<Rest, R>
      : Unique<Rest, readonly [...R, First]>
    : R;
type A = Unique<[1, 2, 2, 3, 1]>;  // [1, 2, 3]
```

### Branding / Nominal Types

```typescript
// Brand<T, B> — добавить бренд к типу
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, 'UserId'>;
type PostId = Brand<number, 'PostId'>;
type Email = Brand<string, 'Email'>;
type UUID = Brand<string, 'UUID'>;
type JwtToken = Brand<string, 'JwtToken'>;

function getUser(id: UserId) { /* ... */ }

const userId: UserId = 1 as UserId;
const postId: PostId = 1 as PostId;

getUser(userId);  // ✅
getUser(postId);  // ❌ Ошибка типа

// Создать branded значение
function createUserId(id: number): UserId {
  return id as UserId;
}

function createEmail(email: string): Email | null {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email) ? (email as Email) : null;
}

// Unbrand
type Unbrand<T> = T extends Brand<infer U, string> ? U : T;
```

### String Utilities

```typescript
// Trim — убрать пробелы по краям
type Trim<T extends string> = T extends ` ${infer S}`
  ? Trim<S>
  : T extends `${infer S} `
  ? Trim<S>
  : T;
type A = Trim<'  hello  '>;  // "hello"

// Split — разделить строку
type Split<T extends string, D extends string> = 
  T extends `${infer F}${D}${infer R}`
    ? [F, ...Split<R, D>]
    : [T];
type A = Split<'a,b,c', ','>;  // ["a", "b", "c"]

// Join — соединить кортеж
type Join<T extends readonly any[], D extends string = ''> =
  T extends readonly [infer F, ...infer R]
    ? R extends readonly any[]
      ? `${F & string}${D}${Join<R, D>}`
      : `${F & string}`
    : '';
type A = Join<['a', 'b', 'c'], '-'>;  // "a-b-c"

// StartsWith / EndsWith
type StartsWith<T extends string, S extends string> = 
  T extends `${S}${string}` ? true : false;
type EndsWith<T extends string, S extends string> = 
  T extends `${string}${S}` ? true : false;
type A = StartsWith<'hello world', 'hello'>;  // true
type B = EndsWith<'hello world', 'world'>;    // true
```

### JSON Types

```typescript
// JSONify<T> — тип для JSON сериализации
type JSONPrimitive = string | number | boolean | null | undefined;
type JSONValue = JSONPrimitive | JSONObject | JSONArray;
type JSONObject = { [key: string]: JSONValue };
type JSONArray = readonly JSONValue[];

type JSONify<T> = {
  [K in keyof T]: T[K] extends Date
    ? string
    : T[K] extends Function
    ? never
    : T[K] extends object
    ? JSONify<T[K]>
    : T[K] extends JSONPrimitive
    ? T[K]
    : never;
}[keyof T] extends never ? never : {
  [K in keyof T]: T[K] extends JSONValue ? T[K] : string
};

// Serializable<T> — только сериализуемые поля
type Serializable<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};
```

---

## Продвинутые паттерны

### Conditional Types

```typescript
// Базовый условный тип
type IsString<T> = T extends string ? true : false;
type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Distributive conditional types
type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>;  // string[] | number[]

// Infer keyword
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any;
type A = ReturnType<() => string>;  // string

// Nested infer
type UnwrapPromise<T> = T extends Promise<infer U> ? UnwrapPromise<U> : T;
type A = UnwrapPromise<Promise<Promise<string>>>;  // string

// Multiple infer
type Flatten<T> = T extends Array<infer U> ? U : T;
type A = Flatten<number[]>;  // number
```

### Mapped Types

```typescript
// Базовый mapped type
type Readonly<T> = { readonly [P in keyof T]: T[P] };

// С conditional types
type Nullable<T> = { [P in keyof T]: T[P] | null };

// С key remapping
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

interface Person {
  name: string;
  age: number;
}
type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }
```

---

## 🔗 Связанные заметки

- [[TypeScript-MOC]] — индекс раздела
- [[TypeScript-Basics]] — основы
- [[TypeScript-Functions]] — полезные функции
- [[TypeScript-Patterns]] — паттерны
- [[TypeScript-DataFetching]] — fetch, API

---

> [!TIP] Совет
> 4. **Template literal types** — для типизации строк
> 5. **Conditional types** — для сложной логики типов
