---
created: 2026-02-17
tags:
  - testing
  - contract-testing
  - pact
  - integration
aliases:
  - Contract Testing
  - Pact Testing
related:
  - Testing-Patterns
  - Integration-Testing
  - MOC-Testing
---

# Testing — Contract Testing

> [!SUMMARY] Обзор
> Contract Testing: проверка совместимости между сервисами. Pact, consumer-driven contracts.

---

## 🏗️ Что такое Contract Testing

```
┌─────────────────────────────────────────────────────────┐
│              Traditional Integration Tests               │
│                                                          │
│  ┌─────────┐         ┌─────────┐         ┌─────────┐   │
│  │ Service │ ──────► │ Service │ ──────► │ Service │   │
│  │    A    │         │    B    │         │    C    │   │
│  └─────────┘         └─────────┘         └─────────┘   │
│       │                   │                   │         │
│       └───────────────────┴───────────────────┘         │
│                    Slow & Brittle                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              Contract Testing (Pact)                     │
│                                                          │
│  ┌─────────┐         ┌─────────┐                        │
│  │Consumer │ ──────► │ Provider│                        │
│  │  Tests  │         │  Tests  │                        │
│  └────┬────┘         └────┬────┘                        │
│       │                   │                              │
│       │   ┌─────────┐     │                              │
│       └──►│  Pact   │◄────┘                              │
│           │ Contract│                                     │
│           └─────────┘                                     │
│              Fast & Reliable                              │
└─────────────────────────────────────────────────────────┘
```

---

## 📦 Установка (Pact)

```bash
# Consumer (TypeScript)
npm install --save-dev @pact-foundation/pact

# Provider (TypeScript)
npm install --save-dev @pact-foundation/pact
```

---

## 📝 Consumer Test

```typescript
// consumer/user.service.pact.test.ts
import { Pact, Matcher } from '@pact-foundation/pact';
import { UserService } from './user.service';

const pact = new Pact({
  consumer: 'UserConsumer',
  provider: 'UserProvider',
  port: 1234,
  log: './pact.log',
  dir: './pacts',
});

describe('UserService', () => {
  const userMatcher = {
    id: Matcher.like(1),
    email: Matcher.like('test@example.com'),
    name: Matcher.like('John Doe'),
    role: Matcher.like('user'),
  };

  beforeAll(async () => {
    await pact.setup();
  });

  afterAll(async () => {
    await pact.finalize();
  });

  describe('getUser', () => {
    beforeAll(async () => {
      await pact.addInteraction({
        state: 'user exists',
        uponReceiving: 'a request for a user',
        withRequest: {
          method: 'GET',
          path: '/api/users/1',
          headers: {
            'Authorization': 'Bearer token',
          },
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: userMatcher,
        },
      });
    });

    it('should return user', async () => {
      const userService = new UserService('http://localhost:1234');
      const user = await userService.getUser(1);
      
      expect(user.id).toBe(1);
      expect(user.email).toBe('test@example.com');
    });
  });

  describe('createUser', () => {
    beforeAll(async () => {
      await pact.addInteraction({
        state: 'user can be created',
        uponReceiving: 'a request to create a user',
        withRequest: {
          method: 'POST',
          path: '/api/users',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer token',
          },
          body: {
            email: Matcher.like('new@example.com'),
            name: Matcher.like('New User'),
          },
        },
        willRespondWith: {
          status: 201,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: Matcher.like(1),
            email: Matcher.like('new@example.com'),
            name: Matcher.like('New User'),
          },
        },
      });
    });

    it('should create user', async () => {
      const userService = new UserService('http://localhost:1234');
      const user = await userService.createUser({
        email: 'new@example.com',
        name: 'New User',
      });
      
      expect(user.id).toBe(1);
      expect(user.email).toBe('new@example.com');
    });
  });

  afterEach(async () => {
    await pact.verify();
  });
});
```

---

## 📤 Generate Pact File

```typescript
// After consumer tests run
import { Verifier } from '@pact-foundation/pact';

// Generate pact file
await pact.writePact();

// File created: ./pacts/UserConsumer-UserProvider.json
```

---

## ✅ Provider Test

```typescript
// provider/user.provider.pact.test.ts
import { Verifier } from '@pact-foundation/pact';
import { app } from './app';  // Your Express/NestJS app

describe('UserProvider', () => {
  const opts: VerifierOptions = {
    provider: 'UserProvider',
    providerBaseUrl: 'http://localhost:3000',
    pactUrls: ['../pacts/UserConsumer-UserProvider.json'],
    // Or from Pact Broker
    // pactBrokerUrl: 'https://pact-broker.example.com',
    // pactBrokerUsername: 'user',
    // pactBrokerPassword: 'password',
    
    // Provider states
    stateHandlers: {
      'user exists': async () => {
        await db.user.upsert({
          where: { id: 1 },
          update: { email: 'test@example.com' },
          create: { id: 1, email: 'test@example.com', name: 'John' },
        });
      },
      'user can be created': async () => {
        await db.user.deleteMany({});
      },
    },
    
    // Before each interaction
    beforeEach: async () => {
      await db.$transaction([
        db.user.deleteMany({}),
      ]);
    },
    
    // After all tests
    afterEach: async () => {
      await db.$transaction([
        db.user.deleteMany({}),
      ]);
    },
  };

  let server: any;

  beforeAll(async () => {
    server = app.listen(3000);
  });

  afterAll(async () => {
    server.close();
  });

  it('should validate Pact contracts', async () => {
    const output = await new Verifier(opts).verifyProvider();
    expect(output.success).toBe(true);
  });
});
```

---

## 🌐 Pact Broker

```yaml
# CI/CD Integration

# Consumer CI
- name: Run consumer tests
  run: npm test

- name: Publish pact to broker
  run: |
    npx pact-broker publish ./pacts \
      --broker-base-url https://pact-broker.example.com \
      --consumer-app-version $(git rev-parse HEAD) \
      --tag $(git branch --show-current)

# Provider CI
- name: Verify provider
  run: |
    npx pact-broker verify \
      --broker-base-url https://pact-broker.example.com \
      --provider-app-version $(git rev-parse HEAD) \
      --provider-branch $(git branch --show-current)
```

---

## 🎯 Best Practices

### 1. Test Consumer Side

```typescript
// ✅ Good: Test actual usage
await pact.addInteraction({
  uponReceiving: 'a request for a user by ID',
  withRequest: {
    method: 'GET',
    path: '/api/users/1',
  },
  willRespondWith: {
    status: 200,
    body: { id: 1, email: 'test@example.com' },
  },
});

// ❌ Bad: Too generic
await pact.addInteraction({
  uponReceiving: 'a request',
  withRequest: { method: 'GET' },
  willRespondWith: { status: 200 },
});
```

### 2. Use Matchers Wisely

```typescript
// ✅ Good: Specific matchers
{
  id: Matcher.integer(1),
  email: Matcher.regex('^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$', 'test@example.com'),
  createdAt: Matcher.datetime('yyyy-MM-dd HH:mm:ss', '2026-02-17 12:00:00'),
}

// ❌ Bad: Too loose
{
  id: Matcher.like(1),  // Could be string
  email: Matcher.like('test'),  // Not validated
}
```

### 3. Provider States

```typescript
// ✅ Good: Clear states
stateHandlers: {
  'user exists': setupUser,
  'user does not exist': cleanupUsers,
  'user is banned': setupBannedUser,
}

// ❌ Bad: Vague states
stateHandlers: {
  'setup': setupEverything,  // Too broad
}
```

---

## 🔗 Связанные заметки

- [[Testing-Patterns]] — Testing patterns
- [[Integration-Testing]] — Integration testing
- [[MOC-Testing]] — Testing MOC
