---
created: 2026-02-17
tags:
  - oauth
  - oauth2
  - oidc
  - authentication
aliases:
  - OAuth2 Flows
  - OAuth2 OIDC
related:
  - Auth-Patterns
  - NestJS-OAuth
  - Express-OAuth
---

# OAuth2 & OIDC Flows

> [!SUMMARY] Обзор
> OAuth2 — стандарт авторизации. OIDC (OpenID Connect) — слой аутентификации над OAuth2.

---

## 🏗️ OAuth2 Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Client    │ ───► │   Auth      │ ───► │  Resource   │
│  (App/Site) │      │   Server    │      │   Server    │
│             │      │ (OAuth2/OIDC)│     │  (API/User) │
└─────────────┘      └─────────────┘      └─────────────┘
      │                    │                    │
      │                    │                    │
      ▼                    ▼                    ▼
  Redirects            Tokens               Resources
```

### Roles

| Роль | Описание |
|------|----------|
| **Resource Owner** | Пользователь (владелец данных) |
| **Client** | Приложение (web/mobile) |
| **Resource Server** | API с данными |
| **Authorization Server** | Сервер авторизации (Auth0, Okta) |

---

## 🔐 OAuth2 Flows

### Authorization Code (Web Apps)

```
1. User clicks "Login"
   Client → https://auth.com/authorize?
     response_type=code&
     client_id=CLIENT_ID&
     redirect_uri=https://app.com/callback&
     scope=read+write&
     state=RANDOM_STATE

2. User logs in and approves

3. Auth Server redirects back
   https://app.com/callback?code=AUTH_CODE&state=RANDOM_STATE

4. Exchange code for tokens
   POST https://auth.com/oauth/token
   {
     "grant_type": "authorization_code",
     "code": "AUTH_CODE",
     "redirect_uri": "https://app.com/callback",
     "client_id": "CLIENT_ID",
     "client_secret": "CLIENT_SECRET"
   }

5. Response
   {
     "access_token": "eyJ...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "refresh_token": "dGhpc...",
     "scope": "read write"
   }
```

**Use when:** Server-side web applications

---

### Authorization Code + PKCE (SPA/Mobile)

```
1. Generate code verifier
   const verifier = generateRandomString(32);

2. Generate code challenge
   const challenge = SHA256(verifier);

3. Start auth flow
   https://auth.com/authorize?
     response_type=code&
     client_id=CLIENT_ID&
     redirect_uri=https://app.com/callback&
     code_challenge=CHALLENGE&
     code_challenge_method=S256

4. User approves, get code

5. Exchange with verifier
   POST https://auth.com/oauth/token
   {
     "grant_type": "authorization_code",
     "code": "AUTH_CODE",
     "redirect_uri": "https://app.com/callback",
     "client_id": "CLIENT_ID",
     "code_verifier": "VERIFIER"  // Instead of client_secret
   }
```

**Use when:** SPAs, mobile apps, public clients

---

### Client Credentials (Machine-to-Machine)

```
1. Request token
   POST https://auth.com/oauth/token
   {
     "grant_type": "client_credentials",
     "client_id": "CLIENT_ID",
     "client_secret": "CLIENT_SECRET",
     "audience": "https://api.example.com"
   }

2. Response
   {
     "access_token": "eyJ...",
     "token_type": "Bearer",
     "expires_in": 3600
   }

3. Use token
   GET https://api.example.com/data
   Authorization: Bearer eyJ...
```

**Use when:** Server-to-server, APIs, daemons

---

### Refresh Token

```
1. Access token expires
   GET https://api.example.com/data
   Authorization: Bearer EXPIRED_TOKEN
   
   Response: 401 Unauthorized

2. Request new token
   POST https://auth.com/oauth/token
   {
     "grant_type": "refresh_token",
     "refresh_token": "REFRESH_TOKEN",
     "client_id": "CLIENT_ID",
     "client_secret": "CLIENT_SECRET"
   }

3. Response
   {
     "access_token": "NEW_ACCESS_TOKEN",
     "token_type": "Bearer",
     "expires_in": 3600,
     "refresh_token": "NEW_REFRESH_TOKEN"  // Optional: rotating
   }
```

**Use when:** Long-lived sessions

---

### Device Code (IoT/TV)

```
1. Request device code
   POST https://auth.com/oauth/device/code
   {
     "client_id": "CLIENT_ID",
     "scope": "read"
   }

2. Response
   {
     "device_code": "DEVICE_CODE",
     "user_code": "ABCD-1234",
     "verification_uri": "https://auth.com/activate",
     "expires_in": 900,
     "interval": 5
   }

3. Show user code to user
   "Visit https://auth.com/activate and enter: ABCD-1234"

4. Poll for completion
   POST https://auth.com/oauth/token
   {
     "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
     "device_code": "DEVICE_CODE",
     "client_id": "CLIENT_ID"
   }

5. Response (when user approves)
   {
     "access_token": "eyJ...",
     "token_type": "Bearer"
   }
```

**Use when:** Smart TVs, IoT, command-line tools

---

## 🔑 OIDC (OpenID Connect)

### ID Token vs Access Token

```typescript
// Access Token (для API)
{
  "iss": "https://auth.com",
  "sub": "user123",
  "aud": "https://api.example.com",
  "exp": 1234567890,
  "scope": "read write"
}

// ID Token (для аутентификации)
{
  "iss": "https://auth.com",
  "sub": "user123",
  "aud": "CLIENT_ID",
  "exp": 1234567890,
  "iat": 1234567800,
  "name": "John Doe",
  "email": "john@example.com",
  "email_verified": true,
  "picture": "https://..."
}
```

### OIDC Scopes

```
openid          // Required for OIDC
profile         // name, family_name, given_name, etc.
email           // email, email_verified
address         // address (structured)
phone           // phone_number, phone_number_verified
```

---

## 🎯 Implementation Examples

### Node.js (Passport)

```typescript
// passport-google-oauth20
import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback',
    scope: ['profile', 'email'],
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      // Find or create user
      const user = await findOrCreateUser(profile);
      return done(null, user);
    } catch (error) {
      return done(error);
    }
  }
));
```

### React (Auth0)

```typescript
// @auth0/auth0-react
import { Auth0Provider, useAuth0 } from '@auth0/auth0-react';

function App() {
  return (
    <Auth0Provider
      domain="dev-xxx.auth0.com"
      clientId="CLIENT_ID"
      redirectUri={window.location.origin}
      scope="openid profile email"
      audience="https://api.example.com"
    >
      <YourApp />
    </Auth0Provider>
  );
}

function LoginButton() {
  const { loginWithRedirect, user, isAuthenticated } = useAuth0();
  
  return (
    <>
      {!isAuthenticated ? (
        <button onClick={() => loginWithRedirect()}>Log In</button>
      ) : (
        <span>Welcome, {user.name}</span>
      )}
    </>
  );
}
```

---

## 📋 Environment

```bash
# .env
# OAuth2
OAUTH_CLIENT_ID=your-client-id
OAUTH_CLIENT_SECRET=your-client-secret
OAUTH_REDIRECT_URI=https://app.com/callback

# OIDC
OIDC_ISSUER=https://auth.com
OIDC_AUTHORITY=https://auth.com
OIDC_CLIENT_ID=your-client-id

# Tokens
TOKEN_EXPIRY=3600
REFRESH_TOKEN_EXPIRY=604800
```

---

## 🔗 Связанные заметки

- [[Auth-Patterns]] — Authentication patterns
- [[NestJS-OAuth]] — NestJS OAuth
- [[Express-OAuth]] — Express OAuth
- [[JWT-Auth]] — JWT authentication
