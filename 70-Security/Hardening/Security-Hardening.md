---
created: 2026-02-16
tags:
  - cheat-sheet
  - security
  - auth
  - owasp
aliases:
  - Security Cheatsheet
  - Application Security Reference
related:
  - NestJS-Cheatsheet
  - Docker-Security
  - Kubernetes-Security
---

# Security — Полная шпаргалка

> [!SUMMARY] Обзор
> Безопасность приложений и инфраструктуры. OWASP Top 10, аутентификация, авторизация, шифрование, security headers. Защита от распространённых атак.

---

## 📚 Теория

### Security Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Security Layers                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Network Security                            │    │
│  │  (Firewall, WAF, DDoS protection, Network Policies)     │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Application Security                        │    │
│  │  (Auth, Input Validation, Output Encoding, CSRF)        │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Data Security                               │    │
│  │  (Encryption at rest/in transit, Secrets management)    │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Infrastructure Security                     │    │
│  │  (Hardening, Patching, Container security)              │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### CIA Triad

```
┌─────────────────────────────────────────────────────┐
│                  CIA Triad                           │
│                                                      │
│         ┌──────────────┐                             │
│         │ Confidentiality │ ← Шифрование, доступ     │
│         └──────┬───────┘                             │
│                │                                      │
│  ┌─────────────┼─────────────┐                        │
│  │             │             │                        │
│  ▼             ▼             ▼                        │
│ Integrity ←── Core ──→ Availability                   │
│ (Целостность)  │    (Доступность)                     │
│                │                                      │
│         Hash, Sign         Backup, Redundancy        │
└─────────────────────────────────────────────────────┘
```

---

## ⚡ OWASP Top 10 (2021)

### A01: Broken Access Control

```typescript
// ❌ Уязвимо
app.get('/api/users/:id', (req, res) => {
  const user = await db.user.find(req.params.id);
  res.json(user);  // Любой может получить любого пользователя
});

// ✅ Защищено
app.get('/api/users/:id', auth, (req, res) => {
  const user = await db.user.find(req.params.id);
  
  // Проверка прав
  if (user.id !== req.user.id && !req.user.isAdmin) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  res.json(user);
});

// RBAC Middleware
function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

// Использование
app.delete('/api/users/:id', 
  auth, 
  requireRole('admin'), 
  deleteUser
);
```

### A02: Cryptographic Failures

```typescript
// ❌ Уязвимо
// Хранение паролей в открытом виде
// HTTP вместо HTTPS
// Слабые алгоритмы (MD5, SHA1)

// ✅ Защищено
// Hash паролей
import bcrypt from 'bcrypt';

const saltRounds = 12;
const hash = await bcrypt.hash(password, saltRounds);
const isValid = await bcrypt.compare(password, hash);

// HTTPS только
app.use((req, res, next) => {
  if (!req.secure && process.env.NODE_ENV === 'production') {
    return res.redirect(`https://${req.headers.host}${req.url}`);
  }
  next();
});

// Шифрование данных
import crypto from 'crypto';

function encrypt(text: string, key: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return `${iv.toString('hex')}:${encrypted}`;
}

// Security headers
app.use(helmet({
  hsts: { maxAge: 31536000, includeSubDomains: true },
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
}));
```

### A03: Injection

```typescript
// ❌ SQL Injection
const user = await db.query(
  `SELECT * FROM users WHERE email = '${email}'`  // Уязвимо!
);

// ✅ Prepared Statements
const user = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]  // Параметры экранируются
);

// ❌ NoSQL Injection
const user = await db.users.findOne({
  $where: `this.email === '${email}'`  // Уязвимо!
});

// ✅ Safe Query
const user = await db.users.findOne({ email });

// ❌ Command Injection
exec(`ls ${userInput}`);  // Уязвимо!

// ✅ Safe
import { escape } from 'shell-quote';
exec(`ls ${escape(userInput)}`);

// Input Validation
import { z } from 'zod';

const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(50),
  age: z.number().min(0).max(150),
});

const validated = userSchema.parse(input);
```

### A04: Insecure Design

```typescript
// Rate Limiting
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 минут
  max: 5,  // 5 попыток
  message: 'Too many login attempts',
  standardHeaders: true,
  legacyHeaders: false,
});

app.post('/api/login', loginLimiter, loginHandler);

// Account Lockout
class AccountLockout {
  private attempts = new Map<string, { count: number; lockedUntil: number }>();
  
  checkLockout(userId: string): boolean {
    const lock = this.attempts.get(userId);
    if (lock && lock.lockedUntil > Date.now()) {
      return true;  // Заблокирован
    }
    return false;
  }
  
  recordFailure(userId: string): void {
    const current = this.attempts.get(userId) || { count: 0, lockedUntil: 0 };
    current.count++;
    
    if (current.count >= 5) {
      current.lockedUntil = Date.now() + 30 * 60 * 1000;  // 30 минут
    }
    
    this.attempts.set(userId, current);
  }
}
```

### A05: Security Misconfiguration

```yaml
# ❌ Уязвимый Docker
FROM node:latest
USER root
EXPOSE 22
ENV DB_PASSWORD=secret

# ✅ Безопасный Docker
FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000/health || exit 1
```

```typescript
// CORS Configuration
import cors from 'cors';

// ❌ Уязвимо
app.use(cors());  // Разрешает всё

// ✅ Защищено
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
}));
```

### A06: Vulnerable Components

```bash
# Проверка уязвимостей
npm audit
npm audit fix
npm audit fix --force

# Snyk
npm install -g snyk
snyk test
snyk monitor

# Dependabot (GitHub)
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### A07: Authentication Failures

```typescript
// ✅ Secure Authentication
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

// JWT с коротким TTL
const accessToken = jwt.sign(
  { userId: user.id, role: user.role },
  process.env.JWT_SECRET!,
  { expiresIn: '15m' }  // Короткое время жизни
);

// Refresh token
const refreshToken = crypto.randomBytes(40).toString('hex');
await db.refreshToken.create({
  data: {
    userId: user.id,
    token: hash(refreshToken),
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),  // 7 дней
  },
});

// Password Policy
const passwordSchema = z.string()
  .min(12, 'Password must be at least 12 characters')
  .regex(/[a-z]/, 'Must contain lowercase')
  .regex(/[A-Z]/, 'Must contain uppercase')
  .regex(/[0-9]/, 'Must contain number')
  .regex(/[^a-zA-Z0-9]/, 'Must contain special character');

// MFA
import speakeasy from 'speakeasy';
import qrcode from 'qrcode';

async function setupMFA(userId: string) {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${user.email})`,
    length: 32,
  });
  
  const qrCode = await qrcode.toDataURL(secret.otpauth_url!);
  
  await db.user.update({
    where: { id: userId },
    data: {
      mfaSecret: secret.base32,
      mfaEnabled: false,  // Включить после подтверждения
    },
  });
  
  return { secret: secret.base32, qrCode };
}

function verifyMFA(secret: string, token: string): boolean {
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 1,  // ±1 период
  });
}
```

### A08: Data Integrity

```typescript
// Digital Signatures
import crypto from 'crypto';

function sign(data: string, privateKey: string): string {
  const sign = crypto.createSign('SHA256');
  sign.write(data);
  sign.end();
  return sign.sign(privateKey, 'hex');
}

function verify(data: string, signature: string, publicKey: string): boolean {
  const verify = crypto.createVerify('SHA256');
  verify.write(data);
  verify.end();
  return verify.verify(publicKey, signature, 'hex');
}

// Checksums
function hash(data: string): string {
  return crypto.createHash('sha256').update(data).digest('hex');
}
```

### A09: Logging Failures

```typescript
// ✅ Secure Logging
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Не логировать чувствительные данные
function sanitize(obj: any): any {
  const sensitive = ['password', 'token', 'secret', 'apiKey', 'creditCard'];
  const copy = { ...obj };
  
  for (const key of sensitive) {
    if (key in copy) {
      copy[key] = '[REDACTED]';
    }
  }
  
  return copy;
}

logger.info('User login', { userId: user.id, ip: req.ip });  // ✅
logger.info('User login', { password: user.password });  // ❌
```

### A10: SSRF

```typescript
// ❌ Уязвимо
app.get('/api/fetch', async (req, res) => {
  const url = req.query.url as string;
  const response = await fetch(url);  // Может запросить внутренний ресурс!
  res.send(await response.text());
});

// ✅ Защищено
import ipaddr from 'ipaddr.js';

function isPrivateIP(ip: string): boolean {
  try {
    const addr = ipaddr.parse(ip);
    return addr.range() !== 'unicast';
  } catch {
    return true;  // Блокировать при ошибке
  }
}

async function safeFetch(url: string): Promise<string> {
  const parsed = new URL(url);
  
  // Разрешить только http/https
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    throw new Error('Invalid protocol');
  }
  
  // Проверка IP
  const dns = require('dns').promises;
  const addresses = await dns.lookup(parsed.hostname, { all: true });
  
  for (const addr of addresses) {
    if (isPrivateIP(addr.address)) {
      throw new Error('Private IP not allowed');
    }
  }
  
  const response = await fetch(url, { timeout: 5000 });
  return response.text();
}
```

---

## 🔐 Auth Patterns

### JWT Flow

```
┌─────────┐                              ┌─────────┐
│  Client │                              │  Server │
└────┬────┘                              └────┬────┘
     │                                        │
     │  POST /login (email, password)         │
     │───────────────────────────────────────>│
     │                                        │
     │           Verify credentials           │
     │                                        │
     │  Access Token (15m) + Refresh Token    │
     │<───────────────────────────────────────│
     │                                        │
     │  GET /api/data (Access Token)          │
     │───────────────────────────────────────>│
     │                                        │
     │           Validate token               │
     │                                        │
     │  Data                                  │
     │<───────────────────────────────────────│
     │                                        │
     │  POST /refresh (Refresh Token)         │
     │───────────────────────────────────────>│
     │                                        │
     │  New Access Token                      │
     │<───────────────────────────────────────│
```

### OAuth 2.0 Flow

```typescript
// OAuth 2.0 (Authorization Code)
// 1. Redirect to provider
app.get('/auth/google', (req, res) => {
  const params = new URLSearchParams({
    client_id: process.env.GOOGLE_CLIENT_ID,
    redirect_uri: process.env.GOOGLE_REDIRECT_URI,
    response_type: 'code',
    scope: 'email profile',
    state: crypto.randomBytes(16).toString('hex'),  // CSRF protection
  });
  
  res.redirect(`https://accounts.google.com/o/oauth2/v2/auth?${params}`);
});

// 2. Handle callback
app.get('/auth/google/callback', async (req, res) => {
  const { code, state } = req.query;
  
  // Verify state
  if (state !== req.session.oauthState) {
    return res.status(400).json({ error: 'Invalid state' });
  }
  
  // Exchange code for token
  const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      code: code as string,
      client_id: process.env.GOOGLE_CLIENT_ID,
      client_secret: process.env.GOOGLE_CLIENT_SECRET,
      redirect_uri: process.env.GOOGLE_REDIRECT_URI,
      grant_type: 'authorization_code',
    }),
  });
  
  const { access_token } = await tokenResponse.json();
  
  // Get user info
  const userResponse = await fetch('https://www.googleapis.com/oauth2/v2/userinfo', {
    headers: { Authorization: `Bearer ${access_token}` },
  });
  
  const userInfo = await userResponse.json();
  
  // Create or update user
  // ...
});
```

---

## 🔗 Связанные заметки

- [[NestJS-Cheatsheet]] — NestJS security
- [[Docker-Security]] — Безопасность контейнеров
- [[Kubernetes-Security]] — K8s security

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Defense in Depth** — многоуровневая защита
> 2. **Principle of Least Privilege** — минимальные права
> 3. **Never trust user input** — валидация везде
> 4. **Security by default** — безопасно по умолчанию
> 5. **Regular audits** — пентесты, code review

> [!INFO] Инструменты
> ```bash
> # SAST
> npm install -g semgrep
> semgrep --config auto .
>
> # DAST
> # OWASP ZAP: https://www.zaproxy.org/
>
> # Dependencies
> npm audit
> snyk test
>
> # Containers
> docker scan image
> trivy image image:tag
> ```

---

## 🔗 Связанные заметки

- [[Auth-Patterns]] — аутентификация
- [[MOC-Security]] — безопасность
- [[Docker-Cheatsheet]] — контейнеризация
- [[Kubernetes-Cheatsheet]] — оркестрация
