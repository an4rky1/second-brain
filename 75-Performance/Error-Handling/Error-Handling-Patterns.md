---
created: 2026-02-16
tags:
  - error-handling
  - patterns
  - resilience
aliases:
  - Error Handling Patterns
  - Resilience Patterns
related:
  - Testing-Patterns
  - NestJS-Cheatsheet
  - Microservices-Patterns
---

# Error Handling Patterns

> [!SUMMARY] Обзор
> Паттерны обработки ошибок: try-catch, Result type, Retry, Circuit Breaker, Fallback.

---

## 📚 Error Types

```
┌─────────────────────────────────────────────────────────────────┐
│  Error Categories                                                │
│                                                                  │
│  Operational   → Expected errors (validation, not found)        │
│  Programming   → Bugs (null reference, type error)              │
│  System        → Infrastructure (DB down, network)              │
│  External      → Third-party API failures                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Basic Patterns

### Try-Catch-Finally

```typescript
// Basic try-catch
try {
  const user = await userRepository.findById(id);
  return user;
} catch (error) {
  logger.error('Failed to get user', { id, error });
  throw new InternalServerErrorException();
}

// Catch specific errors
try {
  await operation();
} catch (error) {
  if (error instanceof NotFoundException) {
    // Handle not found
  } else if (error instanceof BadRequestException) {
    // Handle validation
  } else {
    // Handle unexpected
    logger.error('Unexpected error', { error });
    throw error;
  }
}

// Finally (cleanup)
const resource = await acquireResource();
try {
  await useResource(resource);
} finally {
  await releaseResource(resource);  // Always runs
}
```

### Error Classes

```typescript
// Custom error classes
class AppError extends Error {
  constructor(
    public code: string,
    public status: number,
    message: string,
    public details?: any,
  ) {
    super(message);
    this.name = 'AppError';
  }
}

class ValidationError extends AppError {
  constructor(details: any) {
    super('VALIDATION_ERROR', 400, 'Validation failed', details);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super('NOT_FOUND', 404, `${resource} with id ${id} not found`);
    this.name = 'NotFoundError';
  }
}

class ConflictError extends AppError {
  constructor(message: string) {
    super('CONFLICT', 409, message);
    this.name = 'ConflictError';
  }
}

// Usage
async function createUser(data: CreateUserDto) {
  const existing = await userRepository.findByEmail(data.email);
  
  if (existing) {
    throw new ConflictError('Email already exists');
  }
  
  const validation = validateUser(data);
  if (!validation.valid) {
    throw new ValidationError(validation.errors);
  }
  
  return userRepository.create(data);
}
```

---

## 🎯 Result Type Pattern

### Implementation

```typescript
// Result type
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

// Helper functions
function ok<T>(data: T): Result<T, never> {
  return { success: true, data };
}

function err<E>(error: E): Result<never, E> {
  return { success: false, error };
}

// Usage
async function getUser(id: string): Promise<Result<User, AppError>> {
  try {
    const user = await userRepository.findById(id);
    
    if (!user) {
      return err(new NotFoundError('User', id));
    }
    
    return ok(user);
  } catch (error) {
    return err(new InternalError('Database error'));
  }
}

// Handle result
const result = await getUser('123');

if (result.success) {
  console.log('User:', result.data);
} else {
  console.error('Error:', result.error);
}

// Chain operations
async function processUser(id: string): Promise<Result<void, AppError>> {
  const userResult = await getUser(id);
  if (!userResult.success) return userResult;
  
  const validateResult = validateUser(userResult.data);
  if (!validateResult.success) return validateResult;
  
  const saveResult = await saveUser(userResult.data);
  return saveResult;
}
```

### Rust-Style Result

```typescript
// Result with map/flatMap
class Result<T, E> {
  private constructor(
    private readonly value: T | null,
    private readonly error: E | null,
    private readonly isSuccess: boolean,
  ) {}

  static ok<T>(value: T): Result<T, never> {
    return new Result(value, null, true);
  }

  static err<E>(error: E): Result<never, E> {
    return new Result(null, error, false);
  }

  map<U>(fn: (value: T) => U): Result<U, E> {
    if (this.isSuccess) {
      return Result.ok(fn(this.value!));
    }
    return Result.err(this.error!);
  }

  flatMap<U>(fn: (value: T) => Result<U, E>): Result<U, E> {
    if (this.isSuccess) {
      return fn(this.value!);
    }
    return Result.err(this.error!);
  }

  match<U>(onSuccess: (value: T) => U, onError: (error: E) => U): U {
    if (this.isSuccess) {
      return onSuccess(this.value!);
    }
    return onError(this.error!);
  }
}

// Usage
const result = await getUser('123')
  .flatMap(user => validateUser(user))
  .flatMap(user => saveUser(user))
  .match(
    user => ({ success: true, user }),
    error => ({ success: false, error }),
  );
```

---

## 🔄 Retry Pattern

### Basic Retry

```typescript
async function retry<T>(
  fn: () => Promise<T>,
  retries: number = 3,
  delay: number = 1000,
): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    if (retries <= 0) {
      throw error;
    }
    
    logger.warn('Retrying operation', { retries, error });
    await sleep(delay);
    
    return retry(fn, retries - 1, delay * 2);  // Exponential backoff
  }
}

// Usage
const data = await retry(
  () => fetchExternalApi(),
  3,
  1000,
);
```

### Retry with Policy

```typescript
class RetryPolicy {
  constructor(
    private maxRetries: number = 3,
    private baseDelay: number = 1000,
    private maxDelay: number = 30000,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    let lastError: Error;
    
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;
        
        if (!this.isRetryable(error)) {
          throw error;  // Don't retry non-retryable errors
        }
        
        if (attempt === this.maxRetries) {
          throw error;
        }
        
        const delay = this.calculateDelay(attempt);
        logger.warn('Retrying', { attempt, delay, error });
        await sleep(delay);
      }
    }
    
    throw lastError!;
  }

  private isRetryable(error: Error): boolean {
    // Don't retry on validation errors, auth errors, etc.
    const nonRetryable = ['ValidationError', 'AuthenticationError'];
    return !nonRetryable.includes(error.name);
  }

  private calculateDelay(attempt: number): number {
    // Exponential backoff with jitter
    const exponential = this.baseDelay * Math.pow(2, attempt - 1);
    const jitter = Math.random() * 0.3 * exponential;
    return Math.min(exponential + jitter, this.maxDelay);
  }
}

// Usage
const retryPolicy = new RetryPolicy(3, 1000, 30000);
const data = await retryPolicy.execute(() => fetchExternalApi());
```

---

## ⚡ Circuit Breaker Pattern

### Implementation

```typescript
enum CircuitState {
  CLOSED = 'CLOSED',      // Normal operation
  OPEN = 'OPEN',          // Failing, reject immediately
  HALF_OPEN = 'HALF_OPEN', // Testing if recovered
}

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private lastFailureTime?: number;
  private successCount = 0;

  constructor(
    private failureThreshold: number = 5,
    private resetTimeout: number = 30000,
    private halfOpenMaxCalls: number = 3,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailureTime! > this.resetTimeout) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
      } else {
        throw new CircuitOpenError('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.halfOpenMaxCalls) {
        this.state = CircuitState.CLOSED;
      }
    }
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
      logger.warn('Circuit breaker opened', { failureCount: this.failureCount });
    }
  }
}

// Usage
const circuitBreaker = new CircuitBreaker(5, 30000, 3);

async function callExternalApi() {
  return circuitBreaker.execute(async () => {
    return fetch('https://external-api.com');
  });
}
```

---

## 🛡️ Fallback Pattern

### Implementation

```typescript
async function withFallback<T>(
  primary: () => Promise<T>,
  fallback: () => Promise<T>,
): Promise<T> {
  try {
    return await primary();
  } catch (error) {
    logger.warn('Primary failed, using fallback', { error });
    return fallback();
  }
}

// Usage
const products = await withFallback(
  () => fetchFromPrimaryDatabase(),
  () => fetchFromCache(),
);

// Multiple fallbacks
async function withFallbackChain<T>(
  strategies: Array<() => Promise<T>>,
): Promise<T> {
  let lastError: Error;
  
  for (const strategy of strategies) {
    try {
      return await strategy();
    } catch (error) {
      lastError = error;
      logger.warn('Strategy failed', { error });
    }
  }
  
  throw lastError!;
}

// Usage
const data = await withFallbackChain([
  () => fetchFromPrimary(),
  () => fetchFromSecondary(),
  () => fetchFromCache(),
  () => getDefaultValue(),
]);
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Catch specific errors
try {
  await operation();
} catch (error) {
  if (error instanceof NotFoundError) {
    // Handle
  }
}

// 2. Log with context
logger.error('Database query failed', {
  query: sql,
  params,
  error: error.message,
  stack: error.stack,
});

// 3. Use error boundaries (React)
<ErrorBoundary fallback={<ErrorPage />}>
  <Dashboard />
</ErrorBoundary>

// 4. Graceful degradation
try {
  const recommendations = await getRecommendations(userId);
  return { recommendations };
} catch {
  return { recommendations: [] };  // Empty but working
}

// 5. Validate early
function createUser(data: CreateUserDto) {
  if (!data.email) {
    throw new ValidationError('Email is required');
  }
  // ...
}
```

### ❌ Не делать

```typescript
// 1. Empty catch
try {
  await operation();
} catch (error) {
  // Silent failure ❌
}

// 2. Catch all without handling
try {
  await operation();
} catch (error) {
  throw error;  // Just rethrow ❌
}

// 3. Swallow errors
await operation().catch(() => {});  // ❌

// 4. Generic errors
throw new Error('Something went wrong');  // ❌
throw new NotFoundError('User', id);  // ✅

// 5. Ignore async errors
someAsyncOperation();  // ❌ No .catch()
someAsyncOperation().catch(handleError);  // ✅
```

---

## 🔗 Связанные заметки

- [[Testing-Patterns]] — Тестирование ошибок
- [[NestJS-Cheatsheet]] — NestJS exception filters
- [[Microservices-Patterns]] — Resilience в микросервисах

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Specific errors** — не generic Error
> 2. **Log context** — что, где, почему
> 3. **Retry with backoff** — exponential + jitter
> 4. **Circuit breaker** — protect from cascading failures
> 5. **Fallback** — graceful degradation

> [!INFO] Error Budget
> 
> - 99.9% uptime = 43 min downtime/month
> - 99.99% = 4.3 min
> - 99.999% = 26 sec
