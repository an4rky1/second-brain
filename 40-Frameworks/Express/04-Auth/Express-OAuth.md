---
created: 2026-02-17
tags:
  - express
  - oauth
  - google
  - github
  - facebook
aliases:
  - Express OAuth
  - Express Google GitHub Login
related:
  - Express-MOC
  - Express-Passport
---

# Express.js — OAuth (Google, GitHub, Facebook)

> [!SUMMARY] Обзор
> OAuth 2.0 аутентификация в Express: Google, GitHub, Facebook без Passport.

---

## Google OAuth

### Setup

```bash
npm install googleapis
```

### OAuth Config

```typescript
// config/google.oauth.ts
import { google } from 'googleapis';
import { ConfigService } from '../config/config.service';

const configService = new ConfigService();

export const oauth2Client = new google.auth.OAuth2(
  configService.get('GOOGLE_CLIENT_ID'),
  configService.get('GOOGLE_CLIENT_SECRET'),
  configService.get('GOOGLE_CALLBACK_URL')
);

export const getGoogleAuthUrl = () => {
  return oauth2Client.generateAuthUrl({
    access_type: 'offline',
    scope: [
      'https://www.googleapis.com/auth/userinfo.email',
      'https://www.googleapis.com/auth/userinfo.profile',
    ],
  });
};

export const getGoogleUser = async (accessToken: string) => {
  oauth2Client.setCredentials({ access_token: accessToken });
  
  const oauth2 = google.oauth2({ version: 'v2', auth: oauth2Client });
  const { data } = await oauth2.userinfo.get();
  
  return {
    id: data.id,
    email: data.email,
    name: data.name,
    avatar: data.picture,
  };
};
```

### OAuth Routes

```typescript
// routes/google.oauth.routes.ts
import { Router } from 'express';
import { getGoogleAuthUrl, getGoogleUser } from '../config/google.oauth';
import { AuthService } from '../services/auth.service';

const router = Router();
const authService = new AuthService();

// Redirect to Google
router.get('/login', (req, res) => {
  const authUrl = getGoogleAuthUrl();
  res.redirect(authUrl);
});

// Callback
router.get('/callback', async (req, res) => {
  const { code } = req.query;
  
  try {
    const { tokens } = await oauth2Client.getToken(code as string);
    const googleUser = await getGoogleUser(tokens.access_token!);
    
    // Find or create user
    let user = await authService.findOrCreateOAuthUser({
      provider: 'google',
      providerId: googleUser.id,
      email: googleUser.email,
      name: googleUser.name,
      avatar: googleUser.avatar,
    });
    
    // Generate JWT
    const jwtToken = authService.generateToken(user);
    
    // Redirect to frontend with token
    res.redirect(`https://example.com/auth/callback?token=${jwtToken}`);
  } catch (error) {
    res.redirect('https://example.com/login?error=oauth_failed');
  }
});

export default router;
```

---

## GitHub OAuth

### Setup

```bash
npm install @octokit/auth-oauth
```

### OAuth Config

```typescript
// config/github.oauth.ts
import { createOAuthAppAuth } from '@octokit/auth-oauth';
import { ConfigService } from '../config/config.service';

const configService = new ConfigService();

const auth = createOAuthAppAuth({
  clientId: configService.get('GITHUB_CLIENT_ID'),
  clientSecret: configService.get('GITHUB_CLIENT_SECRET'),
});

export const getGitHubAuthUrl = () => {
  return `https://github.com/login/oauth/authorize?client_id=${configService.get('GITHUB_CLIENT_ID')}&scope=user:email`;
};

export const getGitHubUser = async (code: string) => {
  const { token } = await auth({ type: 'oauth-user', code });
  
  // Get user info
  const response = await fetch('https://api.github.com/user', {
    headers: {
      Authorization: `token ${token}`,
      Accept: 'application/json',
    },
  });
  
  const user = await response.json();
  
  // Get emails
  const emailsResponse = await fetch('https://api.github.com/user/emails', {
    headers: {
      Authorization: `token ${token}`,
      Accept: 'application/json',
    },
  });
  
  const emails = await emailsResponse.json();
  const primaryEmail = emails.find((e: any) => e.primary)?.email;
  
  return {
    id: String(user.id),
    email: primaryEmail || user.email,
    name: user.name || user.login,
    avatar: user.avatar_url,
  };
};
```

### OAuth Routes

```typescript
// routes/github.oauth.routes.ts
import { Router } from 'express';
import { getGitHubAuthUrl, getGitHubUser } from '../config/github.oauth';
import { AuthService } from '../services/auth.service';

const router = Router();
const authService = new AuthService();

router.get('/login', (req, res) => {
  res.redirect(getGitHubAuthUrl());
});

router.get('/callback', async (req, res) => {
  const { code } = req.query;
  
  try {
    const githubUser = await getGitHubUser(code as string);
    
    let user = await authService.findOrCreateOAuthUser({
      provider: 'github',
      providerId: githubUser.id,
      email: githubUser.email,
      name: githubUser.name,
      avatar: githubUser.avatar,
    });
    
    const jwtToken = authService.generateToken(user);
    res.redirect(`https://example.com/auth/callback?token=${jwtToken}`);
  } catch (error) {
    res.redirect('https://example.com/login?error=oauth_failed');
  }
});

export default router;
```

---

## Facebook OAuth

### Setup

```bash
npm install fb
```

### OAuth Config

```typescript
// config/facebook.oauth.ts
import Facebook from 'fb';
import { ConfigService } from '../config/config.service';

const configService = new ConfigService();

Facebook.options({
  appId: configService.get('FACEBOOK_APP_ID'),
  appSecret: configService.get('FACEBOOK_APP_SECRET'),
  version: 'v18.0',
});

export const getFacebookAuthUrl = () => {
  return `https://www.facebook.com/v18.0/dialog/oauth?client_id=${configService.get('FACEBOOK_APP_ID')}&redirect_uri=${configService.get('FACEBOOK_CALLBACK_URL')}&scope=email,public_profile`;
};

export const getFacebookUser = async (accessToken: string) => {
  Facebook.setAccessToken(accessToken);
  
  const { id, email, name, picture } = await Facebook.api('/me', {
    fields: 'id, email, name, picture.type(large)',
  });
  
  return {
    id,
    email,
    name,
    avatar: picture.data.url,
  };
};
```

---

## Environment

```bash
# .env
# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_CALLBACK_URL=https://example.com/auth/google/callback

# GitHub OAuth
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
GITHUB_CALLBACK_URL=https://example.com/auth/github/callback

# Facebook OAuth
FACEBOOK_APP_ID=your-facebook-app-id
FACEBOOK_APP_SECRET=your-facebook-app-secret
FACEBOOK_CALLBACK_URL=https://example.com/auth/facebook/callback

# Frontend URL
FRONTEND_URL=https://example.com
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Passport]] — Passport.js OAuth
- [[Express-Sessions]] — Sessions
