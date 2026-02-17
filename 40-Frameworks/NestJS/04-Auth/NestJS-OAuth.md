---
created: 2026-02-17
tags:
  - nestjs
  - oauth
  - google
  - github
  - facebook
aliases:
  - NestJS OAuth
  - NestJS Google GitHub Login
related:
  - NestJS-MOC
  - NestJS-Passport
  - NestJS-JWT-Auth
---

# NestJS — OAuth (Google, GitHub, Facebook)

> [!SUMMARY] Обзор
> OAuth 2.0 аутентификация в NestJS: Google, GitHub, Facebook через Passport.js.

---

## 📦 Установка

```bash
# Core
npm install @nestjs/passport passport passport-oauth2

# Strategies
npm install passport-google-oauth20 passport-github2 passport-facebook
```

---

## 🔗 Google OAuth Strategy

```typescript
// auth/strategies/google.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback } from 'passport-google-oauth20';
import { ConfigService } from '@nestjs/config';
import { AuthService } from '../auth.service';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(
    private configService: ConfigService,
    private authService: AuthService,
  ) {
    super({
      clientID: configService.get('GOOGLE_CLIENT_ID'),
      clientSecret: configService.get('GOOGLE_CLIENT_SECRET'),
      callbackURL: configService.get('GOOGLE_CALLBACK_URL'),
      scope: ['email', 'profile'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    const { id, emails, displayName, photos } = profile;

    try {
      // Find or create user
      let user = await this.authService.findOrCreateOAuthUser({
        provider: 'google',
        providerId: id,
        email: emails[0].value,
        name: displayName,
        avatar: photos[0].value,
      });

      done(null, user);
    } catch (error) {
      done(error, false);
    }
  }
}
```

---

## 🔗 GitHub OAuth Strategy

```typescript
// auth/strategies/github.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-github2';
import { ConfigService } from '@nestjs/config';
import { AuthService } from '../auth.service';

@Injectable()
export class GitHubStrategy extends PassportStrategy(Strategy, 'github') {
  constructor(
    private configService: ConfigService,
    private authService: AuthService,
  ) {
    super({
      clientID: configService.get('GITHUB_CLIENT_ID'),
      clientSecret: configService.get('GITHUB_CLIENT_SECRET'),
      callbackURL: configService.get('GITHUB_CALLBACK_URL'),
      scope: ['user:email'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    const { id, emails, displayName, photos } = profile;

    try {
      let user = await this.authService.findOrCreateOAuthUser({
        provider: 'github',
        providerId: id,
        email: emails[0].value,
        name: displayName,
        avatar: photos[0].value,
      });

      done(null, user);
    } catch (error) {
      done(error, false);
    }
  }
}
```

---

## 🔗 Facebook OAuth Strategy

```typescript
// auth/strategies/facebook.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-facebook';
import { ConfigService } from '@nestjs/config';
import { AuthService } from '../auth.service';

@Injectable()
export class FacebookStrategy extends PassportStrategy(Strategy, 'facebook') {
  constructor(
    private configService: ConfigService,
    private authService: AuthService,
  ) {
    super({
      clientID: configService.get('FACEBOOK_APP_ID'),
      clientSecret: configService.get('FACEBOOK_APP_SECRET'),
      callbackURL: configService.get('FACEBOOK_CALLBACK_URL'),
      scope: ['email', 'public_profile'],
      profileFields: ['id', 'emails', 'name', 'photos'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    const { id, emails, displayName, photos } = profile;

    try {
      let user = await this.authService.findOrCreateOAuthUser({
        provider: 'facebook',
        providerId: id,
        email: emails[0].value,
        name: displayName,
        avatar: photos[0].value,
      });

      done(null, user);
    } catch (error) {
      done(error, false);
    }
  }
}
```

---

## 🎮 OAuth Controller

```typescript
// auth/auth.controller.ts
import {
  Controller,
  Get,
  UseGuards,
  Request,
  Response,
  Query,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  // ─────────────────────────────────────────────────────
  // GOOGLE OAUTH
  // ─────────────────────────────────────────────────────
  @Get('google')
  @UseGuards(AuthGuard('google'))
  googleLogin() {
    // Redirects to Google
  }

  @Get('google/callback')
  @UseGuards(AuthGuard('google'))
  async googleCallback(@Request() req: any, @Response() res: any) {
    const token = this.authService.generateToken(req.user);
    
    // Redirect to frontend with token
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
  }

  // ─────────────────────────────────────────────────────
  // GITHUB OAUTH
  // ─────────────────────────────────────────────────────
  @Get('github')
  @UseGuards(AuthGuard('github'))
  githubLogin() {
    // Redirects to GitHub
  }

  @Get('github/callback')
  @UseGuards(AuthGuard('github'))
  async githubCallback(@Request() req: any, @Response() res: any) {
    const token = this.authService.generateToken(req.user);
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
  }

  // ─────────────────────────────────────────────────────
  // FACEBOOK OAUTH
  // ─────────────────────────────────────────────────────
  @Get('facebook')
  @UseGuards(AuthGuard('facebook'))
  facebookLogin() {
    // Redirects to Facebook
  }

  @Get('facebook/callback')
  @UseGuards(AuthGuard('facebook'))
  async facebookCallback(@Request() req: any, @Response() res: any) {
    const token = this.authService.generateToken(req.user);
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
  }
}
```

---

## 🏛️ Auth Module

```typescript
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { GoogleStrategy } from './strategies/google.strategy';
import { GitHubStrategy } from './strategies/github.strategy';
import { FacebookStrategy } from './strategies/facebook.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    ConfigModule,
  ],
  providers: [
    AuthService,
    JwtStrategy,
    GoogleStrategy,
    GitHubStrategy,
    FacebookStrategy,
  ],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

---

## 🔄 OAuth Flow

```
1. User clicks "Login with Google"
   ↓
2. Redirect to Google OAuth consent screen
   ↓
3. User grants permission
   ↓
4. Google redirects to callback URL with code
   ↓
5. Exchange code for access token
   ↓
6. Fetch user profile from Google
   ↓
7. Find or create user in database
   ↓
8. Generate JWT token
   ↓
9. Redirect to frontend with token
```

---

## 📋 Environment

```bash
# .env
# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_CALLBACK_URL=https://api.example.com/auth/google/callback

# GitHub OAuth
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
GITHUB_CALLBACK_URL=https://api.example.com/auth/github/callback

# Facebook OAuth
FACEBOOK_APP_ID=your-facebook-app-id
FACEBOOK_APP_SECRET=your-facebook-app-secret
FACEBOOK_CALLBACK_URL=https://api.example.com/auth/facebook/callback

# Frontend
FRONTEND_URL=https://example.com

# JWT
JWT_SECRET=your-jwt-secret
JWT_EXPIRES_IN=1d
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Passport]] — Passport.js
- [[NestJS-JWT-Auth]] — JWT Authentication
- [[NestJS-Sessions]] — Sessions
