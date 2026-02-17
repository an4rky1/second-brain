---
created: 2026-02-16
tags:
  - security
  - auth
  - oauth
  - jwt
aliases:
  - Authentication Patterns
  - Auth Best Practices
related:
  - Security-Hardening
  - NestJS-Cheatsheet
  - API-Protocols-Networking
---

# Authentication Patterns

> [!SUMMARY] Обзор
> Паттерны аутентификации и авторизации: JWT, OAuth2, сессии, refresh tokens, MFA.

---

## 📚 Auth Types

```
┌─────────────────────────────────────────────────────────────────┐
│              Authentication Methods                              │
│                                                                  │
│  Session-Based  → Cookie + Server session                       │
│  Token-Based    → JWT, stateless                                │
│  OAuth2         → Third-party auth (Google, GitHub)             │
│  MFA            → Multi-factor (SMS, TOTP, WebAuthn)            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔐 JWT Authentication

### Token Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  JWT = Header.Payload.Signature                                 │
│                                                                  │
│  Header:  { "alg": "HS256", "typ": "JWT" }                      │
│  Payload: { "sub": "123", "name": "John", "iat": 1516239022 }   │
│  Signature: HMACSHA256(base64(header) + base64(payload), secret)│
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```typescript
// auth/jwt.service.ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private usersService: UsersService,
  ) {}

  async validateUser(email: string, password: string): Promise<User | null> {
    const user = await this.usersService.findByEmail(email);
    
    if (user && await bcrypt.compare(password, user.password)) {
      const { password, ...result } = user;
      return result;
    }
    
    return null;
  }

  async login(user: User) {
    const payload = { 
      sub: user.id, 
      email: user.email,
      role: user.role,
    };
    
    return {
      access_token: this.jwtService.sign(payload, {
        expiresIn: '15m',  // Short-lived
      }),
      refresh_token: this.jwtService.sign(payload, {
        expiresIn: '7d',   // Long-lived
        secret: process.env.REFRESH_TOKEN_SECRET,
      }),
    };
  }

  async refresh(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken, {
        secret: process.env.REFRESH_TOKEN_SECRET,
      });
      
      const user = await this.usersService.findById(payload.sub);
      if (!user) {
        throw new UnauthorizedException();
      }
      
      return this.login(user);
    } catch {
      throw new UnauthorizedException();
    }
  }
}

// auth/auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// auth/local.strategy.ts
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      usernameField: 'email',
    });
  }

  async validate(email: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(email, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

### Refresh Token Flow

```typescript
// auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  async login(@Body() loginDto: LoginDto) {
    const user = await this.authService.validateUser(
      loginDto.email,
      loginDto.password,
    );
    
    if (!user) {
      throw new UnauthorizedException();
    }
    
    const tokens = await this.authService.login(user);
    
    // Store refresh token in DB
    await this.authService.storeRefreshToken(user.id, tokens.refresh_token);
    
    // Set refresh token in HTTP-only cookie
    this.response.cookie('refreshToken', tokens.refresh_token, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 days
    });
    
    return { access_token: tokens.access_token };
  }

  @Post('refresh')
  async refresh(@Req() req: Request) {
    const refreshToken = req.cookies?.refreshToken;
    
    if (!refreshToken) {
      throw new UnauthorizedException();
    }
    
    return this.authService.refresh(refreshToken);
  }

  @Post('logout')
  async logout(@Req() req: Request) {
    const refreshToken = req.cookies?.refreshToken;
    
    if (refreshToken) {
      await this.authService.invalidateRefreshToken(refreshToken);
    }
    
    req.res?.clearCookie('refreshToken');
    return { message: 'Logged out' };
  }
}
```

---

## 🔑 OAuth2 Flow

### Authorization Code Flow

```
┌─────────┐                              ┌─────────┐
│  Client │                              │  Auth   │
│  (App)  │                              │ Server  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  1. Redirect to /authorize             │
     │───────────────────────────────────────>│
     │                                        │
     │  2. User logs in & consents            │
     │                                        │
     │  3. Redirect with code                 │
     │<───────────────────────────────────────│
     │                                        │
     │  4. POST /token (code + secret)        │
     │───────────────────────────────────────>│
     │                                        │
     │  5. Return access_token + refresh_token│
     │<───────────────────────────────────────│
     │                                        │
     │  6. API requests with access_token     │
     │───────────────────────────────────────>│
     │                                        │
```

### Google OAuth Implementation

```typescript
// auth/google.strategy.ts
@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor() {
    super({
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: 'http://localhost:3000/auth/google/callback',
      scope: ['email', 'profile'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: Profile,
  ): Promise<User> {
    const { id, emails, displayName, photos } = profile;
    
    // Find or create user
    let user = await this.usersService.findByProvider('google', id);
    
    if (!user) {
      user = await this.usersService.create({
        email: emails[0].value,
        name: displayName,
        avatar: photos[0].value,
        provider: 'google',
        providerId: id,
      });
    }
    
    return user;
  }
}

// auth.controller.ts
@Get('google')
@UseGuards(AuthGuard('google'))
async googleAuth() {}  // Redirects to Google

@Get('google/callback')
@UseGuards(AuthGuard('google'))
async googleAuthRedirect(@User() user: User, @Res() res: Response) {
  const tokens = await this.authService.login(user);
  
  res.cookie('refreshToken', tokens.refresh_token, {
    httpOnly: true,
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });
  
  res.redirect(`${process.env.FRONTEND_URL}/dashboard?token=${tokens.access_token}`);
}
```

---

## 📱 Multi-Factor Authentication

### TOTP (Time-based OTP)

```typescript
// mfa/mfa.service.ts
import { Injectable } from '@nestjs/common';
import * as speakeasy from 'speakeasy';
import * as qrcode from 'qrcode';

@Injectable()
export class MfaService {
  async generateSecret(userId: string, user: User) {
    const secret = speakeasy.generateSecret({
      name: `MyApp (${user.email})`,
      length: 32,
    });

    const qrCode = await qrcode.toDataURL(secret.otpauth_url!);

    // Store secret (don't enable MFA yet)
    await this.usersService.updateMfaSecret(userId, secret.base32);

    return {
      secret: secret.base32,
      qrCode,
    };
  }

  async verifyCode(userId: string, token: string): Promise<boolean> {
    const user = await this.usersService.findById(userId);
    
    if (!user.mfaSecret) {
      return false;
    }

    const verified = speakeasy.totp.verify({
      secret: user.mfaSecret,
      encoding: 'base32',
      token,
      window: 1,  // ±1 period (±30 seconds)
    });

    return verified;
  }

  async enableMfa(userId: string, token: string): Promise<void> {
    const verified = await this.verifyCode(userId, token);
    
    if (!verified) {
      throw new BadRequestException('Invalid code');
    }

    await this.usersService.enableMfa(userId);
  }

  async disableMfa(userId: string, token: string): Promise<void> {
    const verified = await this.verifyCode(userId, token);
    
    if (!verified) {
      throw new BadRequestException('Invalid code');
    }

    await this.usersService.disableMfa(userId);
  }
}

// mfa/mfa.guard.ts
@Injectable()
export class MfaGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return false;
    }

    const payload = this.jwtService.verify(token);
    
    // Check if MFA is required but not completed
    if (payload.mfaRequired && !payload.mfaVerified) {
      return false;
    }

    return true;
  }
}
```

---

## 🛡️ Security Best Practices

### Password Hashing

```typescript
// Use bcrypt with proper cost factor
const saltRounds = 12;
const hash = await bcrypt.hash(password, saltRounds);
const isValid = await bcrypt.compare(password, hash);

// Argon2 (more secure)
import { hash, verify } from 'argon2';

const hash = await hash(password, {
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 4,
});
const isValid = await verify(hash, password);
```

### Rate Limiting

```typescript
// Login rate limiting
@Injectable()
export class ThrottleGuard {
  private attempts = new Map<string, { count: number; lockedUntil: number }>();

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const ip = request.ip;
    
    const attempt = this.attempts.get(ip);
    
    if (attempt && attempt.lockedUntil > Date.now()) {
      throw new TooManyRequestsException('Too many login attempts');
    }
    
    // Record attempt
    // ...
    
    return true;
  }
}
```

### Security Headers

```typescript
// helmet configuration
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
}));
```

---

## 🔗 Связанные заметки

- [[Security-Hardening]] — Security best practices
- [[NestJS-Cheatsheet]] — NestJS guards
- [[REST-API]] — API security

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Short-lived access tokens** — 15-30 min
> 2. **Refresh tokens** — store securely, rotate
> 3. **HTTPS only** — never send tokens over HTTP
> 4. **HTTP-only cookies** — prevent XSS
> 5. **MFA** — offer to users

> [!WARNING] Never
>
> - Store passwords in plain text
> - Send tokens in URL
> - Use weak algorithms (MD5, SHA1)
> - Log sensitive data

---

## 🔗 Связанные заметки

- [[Security-Hardening]] — безопасность приложений
- [[MOC-Security]] — безопасность
- [[NestJS-Cheatsheet]] — NestJS фреймворк
