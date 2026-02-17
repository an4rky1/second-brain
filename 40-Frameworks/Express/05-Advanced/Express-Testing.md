---
created: 2026-02-17
tags:
  - express
  - testing
  - jest
  - supertest
  - e2e
aliases:
  - Express Testing
  - Express Jest Supertest
related:
  - Express-MOC
  - Express-Architecture
---

# Express.js — Testing

> [!SUMMARY] Обзор
> Unit и E2E тесты в Express.js. Jest, Supertest, mocks.

---

## Установка

```bash
npm install --save-dev jest ts-jest @types/jest supertest @types/supertest
```

---

## Unit Tests

```typescript
// users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { PrismaService } from '../prisma/prisma.service';
import { NotFoundException } from '@nestjs/common';

describe('UsersService', () => {
  let service: UsersService;
  let prisma: PrismaService;

  const mockPrisma = {
    user: {
      findUnique: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    },
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      const mockUser = { id: 1, email: 'test@example.com' };
      mockPrisma.user.findUnique.mockResolvedValue(mockUser);

      const result = await service.findOne(1);
      
      expect(result).toEqual(mockUser);
      expect(prisma.user.findUnique).toHaveBeenCalledWith({ where: { id: 1 } });
    });

    it('should throw NotFoundException', async () => {
      mockPrisma.user.findUnique.mockResolvedValue(null);

      await expect(service.findOne(1)).rejects.toThrow(NotFoundException);
    });
  });

  describe('create', () => {
    it('should create a user', async () => {
      const createUserDto = { email: 'test@example.com', password: 'password' };
      const mockUser = { id: 1, ...createUserDto };
      
      mockPrisma.user.create.mockResolvedValue(mockUser);

      const result = await service.create(createUserDto);
      
      expect(result).toEqual(mockUser);
      expect(prisma.user.create).toHaveBeenCalledWith({ data: createUserDto });
    });
  });
});
```

---

## E2E Tests

```typescript
// users/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from './../src/app.module';

describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe());
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('/users (POST)', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({ email: 'test@example.com', password: 'password' })
        .expect(201)
        .expect((res) => {
          expect(res.body.email).toBe('test@example.com');
        });
    });

    it('should fail validation', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({ email: 'invalid' })
        .expect(400);
    });
  });

  describe('/users (GET)', () => {
    it('should return all users', () => {
      return request(app.getHttpServer())
        .get('/users')
        .expect(200)
        .expect((res) => {
          expect(Array.isArray(res.body.data)).toBe(true);
        });
    });
  });

  describe('/users/:id (GET)', () => {
    it('should return a user', () => {
      return request(app.getHttpServer())
        .get('/users/1')
        .expect(200);
    });

    it('should return 404 for non-existent user', () => {
      return request(app.getHttpServer())
        .get('/users/999')
        .expect(404);
    });
  });
});
```

---

## Jest Config

```typescript
// jest.config.json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testRegex": ".*\\.spec\\.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "collectCoverageFrom": ["**/*.(t|j)s"],
  "coverageDirectory": "./coverage",
  "testEnvironment": "node",
  "moduleNameMapper": {
    "^src/(.*)$": "<rootDir>/src/$1"
  }
}
```

---

## Package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  }
}
```

---

## Mocks

```typescript
// __mocks__/prisma.service.ts
export const mockPrismaService = {
  user: {
    findUnique: jest.fn(),
    findMany: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
  },
  $transaction: jest.fn((fn) => fn(mockPrismaService)),
};

// Usage
providers: [
  { provide: PrismaService, useValue: mockPrismaService },
]
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[Testing-Patterns]] — Testing паттерны
