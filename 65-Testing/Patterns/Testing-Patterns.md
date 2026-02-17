---
created: 2026-02-16
tags:
  - testing
  - patterns
  - mocking
aliases:
  - Testing Patterns
  - Test Strategies
related:
  - Unit-Testing-Jest
  - E2E-Testing-Playwright
  - NestJS-Cheatsheet
---

# Testing Patterns

> [!SUMMARY] Обзор
> Паттерны и стратегии тестирования: mocking, test data, fixtures, integration тесты.

---

## 🎭 Mocking Strategies

### Mock vs Stub vs Spy

```
┌─────────────────────────────────────────────────────┐
│  Mock   → Проверяет взаимодействия (calls)          │
│  Stub   → Возвращет заранее заданные значения       │
│  Spy    → Наблюдает за реальным объектом            │
└─────────────────────────────────────────────────────┘
```

### Mock Patterns

```typescript
// 1. Function Mock
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ data: 'result' });
mockFn.mockRejectedValue(new Error('error'));

// 2. Object Mock
const mockUser = {
  save: jest.fn().mockResolvedValue({ id: 1 }),
  delete: jest.fn().mockResolvedValue(true),
};

// 3. Module Mock
jest.mock('axios', () => ({
  get: jest.fn(() => Promise.resolve({ data: {} })),
}));

// 4. Constructor Mock
jest.mock('../Database', () => {
  return jest.fn().mockImplementation(() => ({
    connect: jest.fn(),
    query: jest.fn(),
  }));
});

// 5. Partial Mock
jest.mock('../utils', () => {
  const actual = jest.requireActual('../utils');
  return {
    ...actual,
    formatDate: jest.fn(() => '2024-01-01'),
  };
});
```

### Dependency Injection для тестов

```typescript
// Service с DI
class UserService {
  constructor(
    private userRepository: UserRepository,
    private emailService: EmailService,
    private logger: Logger,
  ) {}
}

// Test с моками
const mockUserRepo = {
  findById: jest.fn(),
  create: jest.fn(),
} as jest.Mocked<UserRepository>;

const mockEmailService = {
  send: jest.fn(),
} as jest.Mocked<EmailService>;

const service = new UserService(mockUserRepo, mockEmailService);
```

---

## 📊 Test Data Patterns

### Factory Pattern

```typescript
// factories/user.factory.ts
import { faker } from '@faker-js/faker';

export function createUser(overrides = {}) {
  return {
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'user',
    createdAt: new Date(),
    ...overrides,
  };
}

export function createAdmin(overrides = {}) {
  return createUser({
    role: 'admin',
    permissions: ['read', 'write', 'delete'],
    ...overrides,
  });
}

// Usage
const user = createUser({ name: 'Custom Name' });
const admin = createAdmin();
```

### Builder Pattern

```typescript
// builders/user.builder.ts
class UserBuilder {
  private data: Partial<User> = {};

  withId(id: string): this {
    this.data.id = id;
    return this;
  }

  withName(name: string): this {
    this.data.name = name;
    return this;
  }

  withEmail(email: string): this {
    this.data.email = email;
    return this;
  }

  asAdmin(): this {
    this.data.role = 'admin';
    this.data.permissions = ['read', 'write', 'delete'];
    return this;
  }

  build(): User {
    return {
      id: faker.string.uuid(),
      name: faker.person.fullName(),
      email: faker.internet.email(),
      role: 'user',
      ...this.data,
    } as User;
  }
}

// Usage
const user = new UserBuilder()
  .withId('123')
  .withName('John')
  .asAdmin()
  .build();
```

### Fixture Pattern

```typescript
// fixtures/users.fixture.ts
export const users = {
  valid: {
    id: '1',
    name: 'John Doe',
    email: 'john@example.com',
  },
  invalid: {
    id: 'invalid',
    name: '',
    email: 'not-an-email',
  },
  admin: {
    id: '2',
    name: 'Admin',
    email: 'admin@example.com',
    role: 'admin',
  },
};

// Usage
import { users } from './fixtures/users.fixture';

test('should create user', () => {
  const result = createUser(users.valid);
  expect(result.email).toBe(users.valid.email);
});
```

---

## 🔗 Integration Testing

### API Integration Tests

```typescript
// tests/users.api.test.ts
import request from 'supertest';
import { app } from '../src/app';
import { db } from '../src/db';

describe('Users API', () => {
  beforeAll(async () => {
    await db.connect();
  });

  beforeEach(async () => {
    await db.clear();
  });

  afterAll(async () => {
    await db.disconnect();
  });

  describe('POST /api/users', () => {
    it('should create a user', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({
          name: 'John',
          email: 'john@example.com',
        })
        .expect(201);

      expect(response.body).toMatchObject({
        name: 'John',
        email: 'john@example.com',
      });
      expect(response.body).toHaveProperty('id');
    });

    it('should return 400 for invalid email', async () => {
      await request(app)
        .post('/api/users')
        .send({ name: 'John', email: 'invalid' })
        .expect(400);
    });

    it('should return 409 for duplicate email', async () => {
      await request(app)
        .post('/api/users')
        .send({ name: 'John', email: 'john@example.com' })
        .expect(201);

      await request(app)
        .post('/api/users')
        .send({ name: 'John 2', email: 'john@example.com' })
        .expect(409);
    });
  });
});
```

### Database Integration Tests

```typescript
// Test container (Testcontainers)
import { PostgreSqlContainer } from '@testcontainers/postgresql';

describe('UserRepository', () => {
  let container: PostgreSqlContainer;
  let db: Database;

  beforeAll(async () => {
    container = await new PostgreSqlContainer()
      .withDatabase('test_db')
      .withUsername('test')
      .withPassword('test')
      .start();

    db = new Database(container.getConnectionUri());
    await db.migrate();
  });

  afterAll(async () => {
    await container.stop();
  });

  it('should create user', async () => {
    const user = await db.users.create({
      name: 'John',
      email: 'john@example.com',
    });

    expect(user).toHaveProperty('id');
    expect(user.name).toBe('John');
  });
});
```

---

## 🧪 Test Organization

### Test File Structure

```typescript
// users.service.test.ts
import { UserService } from './users.service';
import { UserRepository } from './users.repository';

describe('UserService', () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    } as jest.Mocked<UserRepository>;

    service = new UserService(mockRepo);
  });

  describe('createUser', () => {
    describe('when data is valid', () => {
      it('should create a user', async () => {
        // Test
      });

      it('should send welcome email', async () => {
        // Test
      });
    });

    describe('when email exists', () => {
      it('should throw ConflictException', async () => {
        // Test
      });
    });

    describe('when email is invalid', () => {
      it('should throw BadRequestException', async () => {
        // Test
      });
    });
  });

  describe('updateUser', () => {
    // ...
  });
});
```

### Test Utilities

```typescript
// test-utils.ts
import { render, RenderResult } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

export function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
      },
    },
  });
}

export function renderWithProviders(
  ui: React.ReactElement,
  { preloadedState = {}, queryClient = createTestQueryClient() } = {}
): RenderResult {
  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
}

// Custom matchers
expect.extend({
  toBeValidDate(received) {
    const pass = received instanceof Date && !isNaN(received.getTime());
    return {
      pass,
      message: () => `expected ${received} ${pass ? 'not ' : ''}to be a valid date`,
    };
  },
});
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Test isolation
beforeEach(() => {
  jest.clearAllMocks();
  db.clear();
});

// 2. Descriptive test names
it('should create user with valid data and send welcome email', () => {});

// 3. AAA pattern
it('should update user', () => {
  // Arrange
  const user = createUser();
  
  // Act
  const result = await service.update(user.id, { name: 'New' });
  
  // Assert
  expect(result.name).toBe('New');
});

// 4. Test edge cases
it('should handle empty array', () => {});
it('should handle null input', () => {});
it('should handle very large numbers', () => {});

// 5. Use factories
const user = createUser({ role: 'admin' });
```

### ❌ Не делать

```typescript
// 1. Testing implementation
expect(mockInternalFn).toHaveBeenCalled();  // ❌
expect(result).toEqual(expected);  // ✅

// 2. Multiple concepts in one test
it('should create user and send email and log and...', () => {});  // ❌

// 3. Hardcoded values
const user = { id: '1', name: 'John' };  // ❌
const user = createUser({ id: '1' });  // ✅

// 4. Ignoring async
test('async test', () => {
  fetchData();  // ❌ Нет return/await
});

// 5. Shared state
let user = null;  // ❌ Глобальное состояние
```

---

## 🔗 Связанные заметки

- [[Unit-Testing-Jest]] — Jest основы
- [[E2E-Testing-Playwright]] — E2E тесты
- [[Mocking-Strategies]] — Mocks详解

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Mock boundaries** — мокай внешние зависимости
> 2. **Test real behavior** — не implementation
> 3. **Factories > Fixtures** — гибче для изменений
> 4. **Integration tests** — тестируй критичные пути
> 5. **Fast feedback** — unit тесты быстрые, integration медленнее
