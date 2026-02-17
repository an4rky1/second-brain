---
created: 2026-02-16
tags:
  - testing
  - unit
  - jest
aliases:
  - Unit Testing with Jest
  - Jest Testing Guide
related:
  - Testing-Patterns
  - TypeScript-Cheatsheet
  - NestJS-Cheatsheet
---

# Unit Testing — Jest

> [!SUMMARY] Обзор
> Jest — фреймворк для тестирования JavaScript/TypeScript. Zero config, snapshot testing, mocking из коробки.

---

## ⚡ Быстрый старт

```bash
# Установка
npm install -D jest @types/jest ts-jest

# Инициализация
npx jest --init

# tsconfig.json для Jest
{
  "compilerOptions": {
    "types": ["jest", "node"]
  }
}

# jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/*.test.ts'],
  transform: {
    '^.+\\.ts$': 'ts-jest',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.module.ts',
  ],
  coverageDirectory: 'coverage',
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

---

## 🔧 Основы Jest

### Describe и Test

```typescript
// sum.test.ts
import { sum } from './sum';

describe('sum function', () => {
  it('should add two numbers', () => {
    expect(sum(2, 3)).toBe(5);
  });

  it('should handle negative numbers', () => {
    expect(sum(-1, -1)).toBe(-2);
  });

  it('should handle zero', () => {
    expect(sum(0, 0)).toBe(0);
  });
});

// Test blocks
describe('UserService', () => {
  beforeAll(async () => {
    // Запуск один раз перед всеми тестами
    await db.connect();
  });

  beforeEach(() => {
    // Запуск перед каждым тестом
    mockClear();
  });

  afterEach(() => {
    // После каждого теста
    jest.clearAllMocks();
  });

  afterAll(async () => {
    // Один раз после всех тестов
    await db.disconnect();
  });

  describe('createUser', () => {
    it('should create a user', () => {
      // Test
    });
  });
});
```

### Matchers

```typescript
// Equality
expect(value).toBe(5);           // Strict equality (===)
expect(value).toEqual({ a: 1 }); // Deep equality
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();
expect(value).toBeTruthy();
expect(value).toBeFalsy();

// Numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeGreaterThanOrEqual(3);
expect(value).toBeLessThan(5);
expect(value).toBeLessThanOrEqual(5);
expect(value).toBeCloseTo(0.3);  // Float comparison

// Strings
expect(value).toMatch(/regex/);
expect(value).toContain('substring');
expect(value).toHaveLength(5);

// Arrays & Objects
expect(array).toContain('item');
expect(array).toContainEqual({ a: 1 });
expect(object).toHaveProperty('key');
expect(object).toHaveProperty('key', 'value');

// Exceptions
expect(fn).toThrow();
expect(fn).toThrow(Error);
expect(fn).toThrow('error message');
expect(fn).toThrow(/regex/);

// Negation
expect(value).not.toBe(0);
```

### Async Testing

```typescript
// Callbacks
test('data callback', (done) => {
  function callback(data) {
    expect(data).toBe('result');
    done();
  }
  fetchData(callback);
});

// Promises
test('data promise', () => {
  return fetchData().then(data => {
    expect(data).toBe('result');
  });
});

// Async/Await
test('data async', async () => {
  const data = await fetchData();
  expect(data).toBe('result');
});

// Async with errors
test('async error', async () => {
  await expect(asyncFn()).rejects.toThrow('error');
});
```

---

## 🎭 Mocking

### Manual Mocks

```typescript
// __mocks__/axios.ts
export default {
  get: jest.fn(() => Promise.resolve({ data: {} })),
  post: jest.fn(() => Promise.resolve({ data: {} })),
};

// Usage in test
import axios from 'axios';

jest.mock('axios');

test('fetches data', async () => {
  const data = await fetchData();
  expect(axios.get).toHaveBeenCalledWith('/api/data');
});
```

### jest.fn()

```typescript
// Mock function
const mockFn = jest.fn();
mockFn('a', 'b');

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('a', 'b');
expect(mockFn).toHaveBeenCalledTimes(1);

// Mock implementation
const mockFn = jest.fn((x) => 42 + x);
expect(mockFn(1)).toBe(43);

// Mock resolved promise
const mockFn = jest.fn().mockResolvedValue({ data: 'result' });

// Mock rejected promise
const mockFn = jest.fn().mockRejectedValue(new Error('error'));

// Mock once
mockFn.mockImplementationOnce(() => 'first');
mockFn.mockImplementationOnce(() => 'second');

expect(mockFn()).toBe('first');
expect(mockFn()).toBe('second');
expect(mockFn()).toBe('default');
```

### Module Mocks

```typescript
// Mock entire module
jest.mock('../utils', () => ({
  formatDate: jest.fn(() => '2024-01-01'),
  parseDate: jest.fn(() => new Date()),
}));

// Partial mock
jest.mock('../utils', () => {
  const actual = jest.requireActual('../utils');
  return {
    ...actual,
    formatDate: jest.fn(() => '2024-01-01'),
  };
});

// Mock constructor
jest.mock('../Database', () => {
  return jest.fn().mockImplementation(() => ({
    connect: jest.fn(),
    disconnect: jest.fn(),
  }));
});
```

### Timer Mocks

```typescript
// Fake timers
jest.useFakeTimers();

test('calls setTimeout', () => {
  const callback = jest.fn();
  setTimeout(callback, 1000);

  expect(callback).not.toHaveBeenCalled();

  jest.runAllTimers();
  expect(callback).toHaveBeenCalled();

  // Or run to specific time
  jest.runOnlyPendingTimers();
  jest.advanceTimersByTime(500);
});

// Restore real timers
jest.useRealTimers();
```

---

## 📦 Testing Classes

```typescript
// service.test.ts
import { UserService } from './UserService';
import { UserRepository } from './UserRepository';

jest.mock('./UserRepository');

describe('UserService', () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = new UserRepository() as jest.Mocked<UserRepository>;
    service = new UserService(mockRepo);
  });

  describe('createUser', () => {
    it('should create a user', async () => {
      const userData = { name: 'John', email: 'john@example.com' };
      const expectedUser = { id: '1', ...userData };

      mockRepo.create.mockResolvedValue(expectedUser);

      const result = await service.createUser(userData);

      expect(result).toEqual(expectedUser);
      expect(mockRepo.create).toHaveBeenCalledWith(userData);
    });

    it('should throw if email exists', async () => {
      mockRepo.findByEmail.mockResolvedValue({ id: '1', name: 'Existing' });

      await expect(
        service.createUser({ name: 'John', email: 'existing@example.com' })
      ).rejects.toThrow('Email already exists');
    });
  });
});
```

---

## 📊 Coverage

```bash
# Run with coverage
npm test -- --coverage

# Specific file coverage
npm test -- --coverage --collectCoverageFrom='src/utils/*.ts'

# Open coverage report
open coverage/lcov-report/index.html
```

### Coverage Thresholds

```javascript
// jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Описательные названия тестов
describe('UserService', () => {
  describe('createUser', () => {
    it('should create a user with valid data', async () => {});
    it('should throw if email already exists', async () => {});
  });
});

// 2. Arrange-Act-Assert pattern
test('should create user', () => {
  // Arrange
  const userData = { name: 'John' };
  
  // Act
  const result = await service.create(userData);
  
  // Assert
  expect(result).toEqual(userData);
});

// 3. One assertion per test (в идеале)
test('should set name', () => {
  expect(user.name).toBe('John');
});

test('should set email', () => {
  expect(user.email).toBe('john@example.com');
});

// 4. Mock external dependencies
jest.mock('../database');
jest.mock('../external-api');

// 5. Use test utilities
function createTestUser(overrides = {}) {
  return {
    id: '1',
    name: 'Test User',
    email: 'test@example.com',
    ...overrides,
  };
}
```

### ❌ Не делать

```typescript
// 1. Слишком сложные тесты
test('should do everything', () => {
  // 50 lines of test code ❌
});

// 2. Тестирование реализации
expect(mockFn).toHaveBeenCalled();  // ✅ Что сделано
expect(internalState).toBe(...);    // ❌ Внутренности

// 3. Игнорирование async ошибок
test('async test', async () => {
  fetchData();  // ❌ Нет await/return
});

// 4. Shared state between tests
let user = null;  // ❌

beforeEach(() => {
  user = createUser();  // ✅ Изоляция
});
```

---

## 🔗 Связанные заметки

- [[Testing-Patterns]] — Паттерны тестирования
- [[Mocking-Strategies]] — Mocks, Stubs, Spies
- [[NestJS-Cheatsheet]] — NestJS тестирование
- [[React-Cheatsheet]] — React тесты

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Test isolation** — каждый тест независим
> 2. **Fast tests** — unit тесты должны быть быстрыми
> 3. **Descriptive names** — из названия понятна цель
> 4. **AAA pattern** — Arrange, Act, Assert
> 5. **80% coverage** — хороший target для production

> [!INFO] Команды
> ```bash
> # Запуск тестов
> npm test
> npm test -- --watch      # Watch mode
> npm test -- --coverage   # С покрытием
> npm test -- --verbose    # Подробно
> 
> # Specific tests
> npm test -- testName     # По имени
> npm test -- path/to/file # По файлу
> npm test -- -t "pattern" # По паттерну
> ```
