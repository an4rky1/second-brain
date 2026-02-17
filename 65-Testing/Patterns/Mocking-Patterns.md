---
created: 2026-02-17
tags:
  - testing
  - mocking
  - patterns
  - unit-tests
aliases:
  - Mocking Patterns
  - Testing Mocks Stubs Fakes
related:
  - Testing-Patterns
  - MOC-Testing
  - Unit-Testing-Jest
---

# Testing — Mocking Patterns

> [!SUMMARY] Обзор
> Mocking паттерны: Mocks, Stubs, Fakes, Spies. Когда и что использовать.

---

## 🎭 Test Doubles Types

```
┌─────────────────────────────────────────────────────────┐
│                  Test Doubles                            │
├───────────┬───────────┬───────────┬───────────┬─────────┤
│  Dummy    │   Fake    │   Stub    │   Mock    │  Spy    │
├───────────┼───────────┼───────────┼───────────┼─────────┤
│ Пустышка  │ Подделка  │ Заглушка  │ Имитация  │ Наблюд. │
└───────────┴───────────┴───────────┴───────────┴─────────┘
```

---

## 📦 Dummy

```typescript
// Dummy: просто заполнитель, не используется

// ❌ Bad: Magic values
const user = new User('test', 'test@test.com', 'password');

// ✅ Good: Dummy with clear intent
const dummyUser = new User('dummy', 'dummy@dummy.com', 'dummy');
const dummyEmail = 'dummy@example.com';
const dummyPassword = 'dummy';

// TypeScript helper
function createDummyUser(): User {
  return {
    id: 0,
    email: 'dummy@example.com',
    name: 'Dummy',
    role: 'user',
  };
}
```

---

## 🎭 Fake

```typescript
// Fake: рабочая реализация, но упрощённая

// In-memory repository (для тестов)
class InMemoryUserRepository implements IUserRepository {
  private users: User[] = [];

  async findById(id: string): Promise<User | null> {
    return this.users.find(u => u.id === id) || null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.users.find(u => u.email === email) || null;
  }

  async create(user: User): Promise<User> {
    this.users.push(user);
    return user;
  }

  async update(user: User): Promise<User> {
    const index = this.users.findIndex(u => u.id === user.id);
    this.users[index] = user;
    return user;
  }

  async delete(id: string): Promise<void> {
    this.users = this.users.filter(u => u.id !== id);
  }
}

// Usage in tests
const userRepository = new InMemoryUserRepository();
const userService = new UserService(userRepository);

await userService.create({ email: 'test@example.com', name: 'Test' });
const user = await userService.findByEmail('test@example.com');
expect(user).toBeDefined();
```

### Fake External Service

```typescript
// Fake email service
class FakeEmailService implements IEmailService {
  public sentEmails: { to: string; subject: string; body: string }[] = [];

  async sendWelcomeEmail(email: string): Promise<void> {
    this.sentEmails.push({
      to: email,
      subject: 'Welcome!',
      body: 'Welcome to our platform!',
    });
  }

  async sendPasswordReset(email: string, token: string): Promise<void> {
    this.sentEmails.push({
      to: email,
      subject: 'Password Reset',
      body: `Reset link: ${token}`,
    });
  }

  // Helper for assertions
  hasSentEmail(to: string): boolean {
    return this.sentEmails.some(e => e.to === to);
  }

  getSentEmails(to: string): Array<{ subject: string; body: string }> {
    return this.sentEmails
      .filter(e => e.to === to)
      .map(({ subject, body }) => ({ subject, body }));
  }
}

// Usage in tests
const emailService = new FakeEmailService();
const authService = new AuthService(userRepository, emailService);

await authService.register({ email: 'test@example.com', password: 'password' });

expect(emailService.hasSentEmail('test@example.com')).toBe(true);
expect(emailService.getSentEmails('test@example.com')[0].subject)
  .toBe('Welcome!');
```

---

## 📞 Stub

```typescript
// Stub: предопределённые ответы

// Jest
const mockUserRepository = {
  findById: jest.fn().mockResolvedValue({ id: '1', name: 'John' }),
  findByEmail: jest.fn().mockResolvedValue(null),
  create: jest.fn().mockResolvedValue({ id: '1', email: 'test@example.com' }),
  update: jest.fn().mockResolvedValue({ id: '1', name: 'Updated' }),
  delete: jest.fn().mockResolvedValue(undefined),
};

// Stub with different responses
mockUserRepository.findById
  .mockResolvedValueOnce({ id: '1', name: 'First' })
  .mockResolvedValueOnce({ id: '2', name: 'Second' })
  .mockResolvedValue({ id: '3', name: 'Default' });

// Stub that throws
mockUserRepository.findById
  .mockRejectedValueOnce(new Error('Database error'))
  .mockResolvedValue({ id: '1', name: 'Success' });

// Stub with custom implementation
mockUserRepository.findById.mockImplementation((id: string) => {
  if (id === '1') return Promise.resolve({ id: '1', name: 'John' });
  if (id === '2') return Promise.resolve(null);
  return Promise.reject(new Error('Unexpected id'));
});
```

### Stub External API

```typescript
// Stub HTTP client
const mockHttpClient = {
  get: jest.fn(),
  post: jest.fn(),
  put: jest.fn(),
  delete: jest.fn(),
};

mockHttpClient.get.mockResolvedValue({
  status: 200,
  data: { users: [{ id: 1, name: 'John' }] },
});

mockHttpClient.post.mockResolvedValue({
  status: 201,
  data: { id: 1, name: 'Created User' },
});

// Usage
const userService = new UserService(mockHttpClient);
const users = await userService.findAll();
expect(users).toHaveLength(1);
```

---

## 🎯 Mock

```typescript
// Mock: проверяет взаимодействия

const mockRepository = {
  create: jest.fn(),
};

const userService = new UserService(mockRepository);

await userService.createUser({ email: 'test@example.com' });

// Verify interaction
expect(mockRepository.create).toHaveBeenCalledTimes(1);
expect(mockRepository.create).toHaveBeenCalledWith({
  email: 'test@example.com',
});

// Verify order
expect(mockRepository.create).toHaveBeenCalledBefore(mockEmailService.send);

// Verify not called
expect(mockRepository.delete).not.toHaveBeenCalled();
```

### Mock with Jest

```typescript
// Manual mock
const mockDb = {
  query: jest.fn(),
  transaction: jest.fn(),
};

mockDb.transaction.mockImplementation(async (fn) => {
  return await fn(mockDb);
});

// Auto mock module
jest.mock('../database/prisma.service');
const mockPrisma = PrismaService as jest.Mocked<typeof PrismaService>;

mockPrisma.user.findUnique.mockResolvedValue(null);
mockPrisma.user.create.mockResolvedValue({ id: 1, email: 'test@example.com' });

// Partial mock
const partialMock = jest.fn().mockImplementation(() => ({
  findById: jest.fn(),
  create: jest.fn().mockResolvedValue({ id: 1 }),
  // Other methods not mocked
}));
```

---

## 🕵️ Spy

```typescript
// Spy: наблюдает за реальным объектом

// Spy on method
const spy = jest.spyOn(userService, 'createUser');

await userService.createUser({ email: 'test@example.com' });

expect(spy).toHaveBeenCalledTimes(1);
expect(spy).toHaveBeenCalledWith({ email: 'test@example.com' });

// Spy but keep original implementation
const spy = jest.spyOn(userService, 'createUser');
spy.mockImplementation(async (data) => {
  console.log('Creating user:', data);
  return userService.createUser(data);  // Call original
});

// Spy on getter
const spy = jest.spyOn(object, 'property', 'get');
spy.mockReturnValue('mocked value');

// Spy on constructor
const spy = jest.spyOn(Module, 'ClassName');
```

---

## 🎭 When to Use What

| Type | Use When | Example |
|------|----------|---------|
| **Dummy** | Нужно заполнить параметр | `func(dummyValue)` |
| **Fake** | Нужна рабочая реализация для тестов | In-memory DB |
| **Stub** | Нужны предопределённые ответы | Mock API response |
| **Mock** | Нужно проверить взаимодействие | Verify method called |
| **Spy** | Нужно наблюдать за реальным объектом | Spy on existing service |

---

## 🔗 Связанные заметки

- [[Testing-Patterns]] — Testing patterns
- [[MOC-Testing]] — Testing MOC
- [[Unit-Testing-Jest]] — Jest testing
