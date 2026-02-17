---
created: 2026-02-16
tags:
  - testing
  - integration
  - api
  - database
aliases:
  - Integration Testing
  - API & Database Tests
related:
  - Testing-Patterns
  - Unit-Testing-Jest
  - NestJS-Cheatsheet
---

# Integration Testing

> [!SUMMARY] Обзор
> Интеграционное тестирование: API endpoints, database interactions, external services. Проверяет взаимодействие компонентов.

---

## 📚 Типы интеграционных тестов

```
┌─────────────────────────────────────────────────────┐
│              Integration Test Types                  │
│                                                      │
│  API Tests       → HTTP endpoints (Supertest)       │
│  Database Tests  → Repository, queries              │
│  Service Tests   → Business logic + dependencies    │
│  Contract Tests  → API contracts (Pact)             │
│  Component Tests → Изолированные компоненты         │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 API Integration Tests

### Supertest Setup

```typescript
// tests/users.api.test.ts
import request from 'supertest';
import { INestApplication } from '@nestjs/common';
import { Test, TestingModule } from '@nestjs/testing';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('Users API (e2e)', () => {
  let app: INestApplication;
  let prisma: PrismaService;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
    
    prisma = app.get(PrismaService);
  });

  beforeEach(async () => {
    // Clear database
    await prisma.user.deleteMany();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /users', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          name: 'John Doe',
          email: 'john@example.com',
        })
        .expect(201)
        .expect((res) => {
          expect(res.body).toMatchObject({
            name: 'John Doe',
            email: 'john@example.com',
          });
          expect(res.body).toHaveProperty('id');
          expect(res.body).not.toHaveProperty('password');
        });
    });

    it('should return 400 for missing fields', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({ name: 'John' })  // Missing email
        .expect(400);
    });

    it('should return 409 for duplicate email', async () => {
      await prisma.user.create({
        data: { name: 'Existing', email: 'existing@example.com' },
      });

      return request(app.getHttpServer())
        .post('/users')
        .send({ name: 'John', email: 'existing@example.com' })
        .expect(409);
    });
  });

  describe('GET /users/:id', () => {
    it('should return a user', async () => {
      const user = await prisma.user.create({
        data: { name: 'John', email: 'john@example.com' },
      });

      return request(app.getHttpServer())
        .get(`/users/${user.id}`)
        .expect(200)
        .expect((res) => {
          expect(res.body.name).toBe('John');
        });
    });

    it('should return 404 for non-existent user', () => {
      return request(app.getHttpServer())
        .get('/users/999')
        .expect(404);
    });
  });
});
```

### Authenticated API Tests

```typescript
// tests/auth.api.test.ts
import request from 'supertest';

describe('Auth API', () => {
  let authToken: string;

  beforeAll(async () => {
    // Login and get token
    const response = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'admin@example.com', password: 'password' });
    
    authToken = response.body.access_token;
  });

  describe('Protected endpoints', () => {
    it('should access protected resource', () => {
      return request(app.getHttpServer())
        .get('/users/me')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
    });

    it('should return 401 without token', () => {
      return request(app.getHttpServer())
        .get('/users/me')
        .expect(401);
    });

    it('should return 401 with invalid token', () => {
      return request(app.getHttpServer())
        .get('/users/me')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401);
    });
  });
});
```

---

## 🗄️ Database Integration Tests

### Test Database Setup

```typescript
// test-setup.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { PrismaClient } from '@prisma/client';

let container: PostgreSqlContainer;
let prisma: PrismaClient;

export async function setupTestDatabase() {
  container = await new PostgreSqlContainer()
    .withDatabase('test_db')
    .withUsername('test')
    .withPassword('test')
    .start();

  prisma = new PrismaClient({
    datasources: {
      db: { url: container.getConnectionUri() },
    },
  });

  await prisma.$connect();
  await prisma.$executeRawUnsafe(`
    CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
  `);
  
  // Run migrations
  const { execSync } = require('child_process');
  execSync(`DATABASE_URL=${container.getConnectionUri()} npx prisma migrate deploy`);

  return prisma;
}

export async function teardownTestDatabase() {
  await prisma?.$disconnect();
  await container?.stop();
}

export async function clearDatabase() {
  const tables = ['users', 'posts', 'comments'];
  for (const table of tables) {
    await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${table}" CASCADE`);
  }
}
```

### Repository Tests

```typescript
// tests/user.repository.test.ts
import { UserRepository } from '../src/users/user.repository';
import { prisma, clearDatabase } from './test-setup';

describe('UserRepository', () => {
  let repository: UserRepository;

  beforeEach(async () => {
    await clearDatabase();
    repository = new UserRepository(prisma);
  });

  describe('create', () => {
    it('should create a user', async () => {
      const userData = {
        name: 'John',
        email: 'john@example.com',
        password: 'hashed_password',
      };

      const user = await repository.create(userData);

      expect(user).toMatchObject({
        name: 'John',
        email: 'john@example.com',
      });
      expect(user).toHaveProperty('id');
      expect(user).not.toHaveProperty('password');
    });

    it('should throw on duplicate email', async () => {
      await repository.create({
        name: 'John',
        email: 'john@example.com',
        password: 'hash',
      });

      await expect(
        repository.create({
          name: 'John 2',
          email: 'john@example.com',
          password: 'hash',
        })
      ).rejects.toThrow();
    });
  });

  describe('findById', () => {
    it('should find user by id', async () => {
      const created = await repository.create({
        name: 'John',
        email: 'john@example.com',
        password: 'hash',
      });

      const found = await repository.findById(created.id);

      expect(found).toEqual(created);
    });

    it('should return null for non-existent user', async () => {
      const found = await repository.findById('non-existent');
      expect(found).toBeNull();
    });
  });

  describe('findByEmail', () => {
    it('should find user by email', async () => {
      await repository.create({
        name: 'John',
        email: 'john@example.com',
        password: 'hash',
      });

      const found = await repository.findByEmail('john@example.com');

      expect(found).toHaveProperty('name', 'John');
    });
  });

  describe('update', () => {
    it('should update user', async () => {
      const created = await repository.create({
        name: 'John',
        email: 'john@example.com',
        password: 'hash',
      });

      const updated = await repository.update(created.id, {
        name: 'Jane',
      });

      expect(updated.name).toBe('Jane');
      expect(updated.email).toBe('john@example.com');
    });
  });

  describe('delete', () => {
    it('should delete user', async () => {
      const created = await repository.create({
        name: 'John',
        email: 'john@example.com',
        password: 'hash',
      });

      await repository.delete(created.id);

      const found = await repository.findById(created.id);
      expect(found).toBeNull();
    });
  });
});
```

---

## 🧩 Component Integration Tests

### Testing with Mock External Services

```typescript
// tests/orders.service.test.ts
import { OrdersService } from '../src/orders/orders.service';
import { OrdersRepository } from '../src/orders/orders.repository';
import { PaymentService } from '../src/payments/payment.service';
import { EmailService } from '../src/email/email.service';

describe('OrdersService (Integration)', () => {
  let service: OrdersService;
  let mockPaymentService: jest.Mocked<PaymentService>;
  let mockEmailService: jest.Mocked<EmailService>;
  let mockOrderRepo: jest.Mocked<OrdersRepository>;

  beforeEach(() => {
    mockOrderRepo = {
      create: jest.fn(),
      findById: jest.fn(),
      update: jest.fn(),
    } as jest.Mocked<OrdersRepository>;

    mockPaymentService = {
      charge: jest.fn(),
      refund: jest.fn(),
    } as jest.Mocked<PaymentService>;

    mockEmailService = {
      sendOrderConfirmation: jest.fn(),
      sendShippingNotification: jest.fn(),
    } as jest.Mocked<EmailService>;

    service = new OrdersService(
      mockOrderRepo,
      mockPaymentService,
      mockEmailService,
    );
  });

  describe('createOrder', () => {
    it('should create order and charge payment', async () => {
      const orderData = {
        userId: '1',
        items: [{ productId: '1', quantity: 2 }],
        total: 100,
      };

      mockPaymentService.charge.mockResolvedValue({
        id: 'pay_123',
        status: 'succeeded',
      });

      mockOrderRepo.create.mockResolvedValue({
        id: 'order_123',
        ...orderData,
        status: 'pending',
      });

      const order = await service.createOrder(orderData);

      expect(mockPaymentService.charge).toHaveBeenCalledWith(
        orderData.userId,
        orderData.total,
      );
      expect(mockOrderRepo.create).toHaveBeenCalledWith(orderData);
      expect(mockEmailService.sendOrderConfirmation).toHaveBeenCalled();
      expect(order.status).toBe('pending');
    });

    it('should not create order if payment fails', async () => {
      mockPaymentService.charge.mockRejectedValue(new Error('Payment failed'));

      await expect(
        service.createOrder({ userId: '1', items: [], total: 100 })
      ).rejects.toThrow('Payment failed');

      expect(mockOrderRepo.create).not.toHaveBeenCalled();
      expect(mockEmailService.sendOrderConfirmation).not.toHaveBeenCalled();
    });
  });
});
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Isolate tests
beforeEach(async () => {
  await clearDatabase();
  jest.clearAllMocks();
});

// 2. Use test containers
const container = await new PostgreSqlContainer()
  .withDatabase('test_db')
  .start();

// 3. Test realistic scenarios
it('should handle concurrent orders', async () => {
  await Promise.all([
    service.createOrder(orderData),
    service.createOrder(orderData),
    service.createOrder(orderData),
  ]);
});

// 4. Clean up resources
afterAll(async () => {
  await app.close();
  await prisma.$disconnect();
  await container.stop();
});

// 5. Use factories for test data
const user = createUser({ role: 'admin' });
const order = createOrder({ status: 'pending' });
```

### ❌ Не делать

```typescript
// 1. Don't test against production DB
// Используйте test containers или in-memory DB

// 2. Don't skip cleanup
// Всегда очищайте данные между тестами

// 3. Don't test too much in one test
// Один тест — один сценарий

// 4. Don't ignore async
await service.createOrder(data);  // ✅
service.createOrder(data);  // ❌
```

---

## 🔗 Связанные заметки

- [[Testing-Patterns]] — Паттерны тестирования
- [[Unit-Testing-Jest]] — Unit тесты
- [[E2E-Testing-Playwright]] — E2E тесты

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Test containers** — изолированная БД для тестов
> 2. **Clear between tests** — очистка данных
> 3. **Mock external services** — не тестируйте чужой код
> 4. **Real HTTP server** — тестируйте через HTTP
> 5. **Fast feedback** — integration тесты медленнее unit
