---
created: 2026-02-16
tags:
  - testing
  - e2e
  - playwright
  - browser
aliases:
  - E2E Testing Playwright
  - Playwright Guide
related:
  - Testing-Patterns
  - React-Cheatsheet
  - CI-CD-Pipeline
---

# E2E Testing — Playwright

> [!SUMMARY] Обзор
> Playwright — фреймворк для E2E тестирования браузеров. Поддержка Chrome, Firefox, Safari. Auto-wait, screenshots, video.

---

## ⚡ Быстрый старт

```bash
# Установка
npm init playwright@latest

# Или вручную
npm install -D @playwright/test

# Install browsers
npx playwright install

# Запуск тестов
npx playwright test

# Запуск с UI
npx playwright test --ui

# Запуск конкретного теста
npx playwright test --grep "login"

# Запуск в headed mode
npx playwright test --headed

# Запуск с видео
npx playwright test --video=on
```

### Конфигурация

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30 * 1000,
  expect: {
    timeout: 5000,
  },
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results.json' }],
    ['junit', { outputFile: 'junit-results.xml' }],
  ],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## 🔧 Основы

### Первый тест

```typescript
// tests/example.spec.ts
import { test, expect } from '@playwright/test';

test('has title', async ({ page }) => {
  await page.goto('https://example.com');
  await expect(page).toHaveTitle(/Example/);
});

test('get started link', async ({ page }) => {
  await page.goto('https://example.com');
  await page.getByText('Get started').click();
  await expect(page).toHaveURL(/.*get-started/);
});
```

### Locators

```typescript
// By role
page.getByRole('button', { name: 'Submit' });
page.getByRole('link', { name: 'Home' });
page.getByRole('textbox', { name: 'Email' });

// By label
page.getByLabel('Email address');

// By placeholder
page.getByPlaceholder('john@example.com');

// By text
page.getByText('Welcome');
page.getByText('Welcome', { exact: true });

// By test id
page.getByTestId('submit-button');

// CSS selectors
page.locator('.button');
page.locator('#submit');
page.locator('input[name="email"]');
page.locator('div.card >> text=Title');

// XPath
page.locator('//button[@type="submit"]');
```

### Assertions

```typescript
import { expect } from '@playwright/test';

// Visibility
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();

// Content
await expect(locator).toHaveText('Hello');
await expect(locator).toContainText('Hello');
await expect(locator).toHaveAttribute('type', 'submit');
await expect(locator).toHaveValue('input value');
await expect(locator).toHaveClass('active');

// Count
await expect(locator).toHaveCount(3);

// URL
await expect(page).toHaveURL(/.*dashboard/);
await expect(page).toHaveURL('https://example.com/dashboard');

// Title
await expect(page).toHaveTitle(/Dashboard/);
```

---

## 🎭 Actions

```typescript
// Click
await page.getByText('Submit').click();
await page.getByText('Submit').click({ button: 'right' });
await page.getByText('Submit').dblclick();

// Type
await page.getByLabel('Email').fill('john@example.com');
await page.getByLabel('Bio').fill('Developer');
await page.keyboard.press('Enter');

// Select
await page.getByLabel('Country').selectOption('US');
await page.getByLabel('Country').selectOption({ label: 'United States' });

// Check/Radio
await page.getByLabel('Accept').check();
await page.getByLabel('Male').check();
await page.getByLabel('Accept').uncheck();

// Drag and Drop
await page.locator('#draggable').dragTo(page.locator('#droppable'));

// File Upload
await page.getByLabel('Upload').setInputFiles('file.pdf');
await page.getByLabel('Upload').setInputFiles(['file1.pdf', 'file2.pdf']);

// Wait for
await page.waitForSelector('.loaded');
await page.waitForURL('/dashboard');
await page.waitForResponse('**/api/users');
await page.waitForTimeout(1000);  // Избегать!
```

---

## 📦 Page Object Model

```typescript
// tests/pages/login.page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByText('Invalid credentials');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.goto();
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async loginWithInvalidCredentials() {
    await this.login('invalid@example.com', 'wrongpassword');
    await this.errorMessage.waitFor();
  }
}

// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page';
import { DashboardPage } from './pages/dashboard.page';

test.describe('Login', () => {
  let loginPage: LoginPage;
  let dashboardPage: DashboardPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    dashboardPage = new DashboardPage(page);
  });

  test('should login with valid credentials', async ({ page }) => {
    await loginPage.login('user@example.com', 'password123');
    await expect(page).toHaveURL('/dashboard');
    await expect(dashboardPage.welcomeMessage).toBeVisible();
  });

  test('should show error with invalid credentials', async () => {
    await loginPage.loginWithInvalidCredentials();
    await expect(loginPage.errorMessage).toBeVisible();
  });
});
```

---

## 🔐 Auth & Storage

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');
  
  await page.context().storageState({ path: authFile });
});

// tests/dashboard.spec.ts
import { test, expect } from '@playwright/test';

test.use({ storageState: 'playwright/.auth/user.json' });

test('should show dashboard', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page.getByText('Welcome')).toBeVisible();
});
```

---

## 📸 Screenshots & Video

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
  },
});

// In test
await page.screenshot({ path: 'screenshot.png' });
await page.screenshot({ path: 'full.png', fullPage: true });

// Manual video
const context = await browser.newContext({
  recordVideo: { dir: 'videos/', size: { width: 1280, height: 720 } },
});
```

---

## 🌐 API Testing

```typescript
// tests/api/users.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Users API', () => {
  test('should get users', async ({ request }) => {
    const response = await request.get('/api/users');
    
    expect(response.ok()).toBeTruthy();
    expect(response.status()).toBe(200);
    
    const users = await response.json();
    expect(users.length).toBeGreaterThan(0);
    expect(users[0]).toHaveProperty('id');
    expect(users[0]).toHaveProperty('name');
  });

  test('should create user', async ({ request }) => {
    const response = await request.post('/api/users', {
      data: {
        name: 'John',
        email: 'john@example.com',
      },
    });
    
    expect(response.status()).toBe(201);
    
    const user = await response.json();
    expect(user.name).toBe('John');
  });

  test('should update user', async ({ request }) => {
    const response = await request.put('/api/users/1', {
      data: { name: 'Updated' },
    });
    
    expect(response.status()).toBe(200);
  });

  test('should delete user', async ({ request }) => {
    const response = await request.delete('/api/users/1');
    expect(response.status()).toBe(204);
  });
});
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Use test IDs
<button data-testid="submit-button">Submit</button>
await page.getByTestId('submit-button').click();

// 2. Page Object Model
class LoginPage {
  constructor(private page: Page) {}
  async login(email: string, password: string) { /* ... */ }
}

// 3. Reusable fixtures
test('login', async ({ page, loginPage }) => {
  await loginPage.login('user@example.com', 'password');
});

// 4. Wait for network idle
await page.goto('/dashboard', { waitUntil: 'networkidle' });

// 5. Use beforeEach for setup
test.beforeEach(async ({ page }) => {
  await page.goto('/login');
});
```

### ❌ Не делать

```typescript
// 1. Hardcoded waits
await page.waitForTimeout(5000);  // ❌
await page.waitForSelector('.loaded');  // ✅

// 2. Fragile selectors
await page.locator('div > span:nth-child(3)').click();  // ❌
await page.getByRole('button', { name: 'Submit' }).click();  // ✅

// 3. Testing implementation details
// Тестируйте поведение, не реализацию

// 4. Too many assertions per test
// Один тест — одна концепция
```

---

## 🔗 Связанные заметки

- [[Testing-Patterns]] — Паттерны тестирования
- [[Unit-Testing-Jest]] — Unit тесты
- [[CI-CD-Pipeline]] — Тесты в CI/CD

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Test IDs** — стабильные селекторы
> 2. **Page Object** — переиспользуемый код
> 3. **Auto-wait** — Playwright ждёт сам
> 4. **Screenshots on fail** — для debugging
> 5. **Parallel execution** — быстрее прогон

> [!INFO] Команды
> ```bash
> # Запуск
> npx playwright test
> npx playwright test --headed    # Виден браузер
> npx playwright test --debug     # Debug mode
> npx playwright test --ui        # UI mode
> 
> # Specific
> npx playwright test login       # По имени файла
> npx playwright test -g "login"  # По паттерну
> npx playwright test --project=chromium
> 
> # Reports
> npx playwright show-report      # HTML report
> ```
