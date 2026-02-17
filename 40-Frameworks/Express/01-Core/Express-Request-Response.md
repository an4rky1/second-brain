---
created: 2026-02-17
tags:
  - express
  - request
  - response
  - req
  - res
aliases:
  - Express Request Response
  - Express req res
related:
  - Express-MOC
  - Express-Routing
---

# Express.js — Request & Response

> [!SUMMARY] Обзор
> Request (req) и Response (res) объекты в Express: методы, свойства, примеры.

---

## 📥 Request Object

### Basic Properties

```typescript
app.get('/example', (req, res) => {
  // Route params: /users/123
  const { id } = req.params;
  
  // Query params: /users?search=john
  const { search } = req.query;
  
  // Body (requires body-parser)
  const userData = req.body;
  
  // Headers
  const authHeader = req.headers.authorization;
  const contentType = req.headers['content-type'];
  
  // Cookies (requires cookie-parser)
  const sessionId = req.cookies?.sessionId;
  
  // IP address
  const ip = req.ip;
  
  // User agent
  const userAgent = req.get('user-agent');
  
  // Protocol & host
  const protocol = req.protocol;
  const host = req.get('host');
  const fullUrl = `${protocol}://${host}${req.originalUrl}`;
  
  res.json({ received: true });
});
```

### Request Methods

```typescript
// Check if content type is JSON
req.is('json');
req.is('application/json');

// Check accepted content type
req.accepts('json');
req.accepts('html');

// Get referrer
req.get('referer');
req.get('referrer');

// Check if fresh (cache)
if (req.fresh) {
  return res.status(304).send();
}

// Check if stale
if (req.stale) {
  // Fetch fresh data
}

// Subdomains
const subdomains = req.subdomains;
// myapp.dev.example.com => ['dev', 'myapp']

// Check if XHR/Ajax
if (req.xhr) {
  // Ajax request
}
```

---

## 📤 Response Object

### Sending Data

```typescript
// JSON response
res.json({ message: 'Success', data: {} });

// Send string
res.send('Hello World');

// Send HTML
res.send('<h1>Hello</h1>');

// Send buffer
res.send(Buffer.from('binary data'));

// Send stream
res.send(fs.createReadStream('file.txt'));

// No content
res.status(204).send();

// Download file
res.download('/path/to/file.pdf');

// Send file
res.sendFile('/path/to/file.html');
```

### Status & Headers

```typescript
// Status code
res.status(200).json({ message: 'OK' });
res.status(201).json({ message: 'Created' });
res.status(204).send();
res.status(400).json({ error: 'Bad Request' });
res.status(401).json({ error: 'Unauthorized' });
res.status(403).json({ error: 'Forbidden' });
res.status(404).json({ error: 'Not Found' });
res.status(500).json({ error: 'Internal Server Error' });

// Custom headers
res.set('X-Custom-Header', 'value');
res.set({
  'X-Custom-Header': 'value',
  'X-Another-Header': 'value2',
});

// Content type
res.type('json');
res.type('text/html');
res.type('png');
res.type('application/pdf');

// Content disposition (download)
res.attachment('file.pdf');

// Cache control
res.set('Cache-Control', 'public, max-age=3600');
```

### Redirect & Cookies

```typescript
// Redirect
res.redirect('/new-location');
res.redirect(301, '/permanent-location');
res.redirect('https://example.com');
res.redirect('back'); // Back to referrer

// Cookies
res.cookie('sessionId', 'abc123', {
  maxAge: 900000,
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
});

// Clear cookie
res.clearCookie('sessionId');

// Download with custom name
res.download('/path/to/file.pdf', 'custom-name.pdf');
```

---

## 🔄 Request/Response Flow

### Full Example

```typescript
// src/controllers/users.controller.ts
import { Request, Response } from 'express';

export class UsersController {
  // GET /users
  async getAll(req: Request, res: Response) {
    try {
      const { page = 1, limit = 10, search } = req.query;
      
      const users = await UserService.findAll({
        page: Number(page),
        limit: Number(limit),
        search: search as string,
      });
      
      res.json({
        data: users,
        meta: {
          page: Number(page),
          limit: Number(limit),
        },
      });
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch users' });
    }
  }

  // GET /users/:id
  async getOne(req: Request, res: Response) {
    try {
      const { id } = req.params;
      
      const user = await UserService.findById(Number(id));
      
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      
      res.json({ data: user });
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch user' });
    }
  }

  // POST /users
  async create(req: Request, res: Response) {
    try {
      const userData = req.body;
      
      const user = await UserService.create(userData);
      
      res.status(201).json({
        message: 'User created',
        data: user,
      });
    } catch (error: any) {
      if (error.message === 'Email already exists') {
        return res.status(400).json({ error: error.message });
      }
      res.status(500).json({ error: 'Failed to create user' });
    }
  }

  // PUT /users/:id
  async update(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const userData = req.body;
      
      const user = await UserService.update(Number(id), userData);
      
      res.json({
        message: 'User updated',
        data: user,
      });
    } catch (error: any) {
      if (error.message === 'User not found') {
        return res.status(404).json({ error: error.message });
      }
      res.status(500).json({ error: 'Failed to update user' });
    }
  }

  // DELETE /users/:id
  async delete(req: Request, res: Response) {
    try {
      const { id } = req.params;
      
      await UserService.delete(Number(id));
      
      res.status(204).send();
    } catch (error: any) {
      if (error.message === 'User not found') {
        return res.status(404).json({ error: error.message });
      }
      res.status(500).json({ error: 'Failed to delete user' });
    }
  }
}
```

---

## 📊 Response Helpers

```typescript
// Format response based on Accept header
app.get('/users', (req, res) => {
  const users = [{ id: 1, name: 'John' }];
  
  res.format({
    'text/html': () => res.send(`<ul>${users.map(u => `<li>${u.name}</li>`).join('')}</ul>`),
    'application/json': () => res.json({ data: users }),
    'text/plain': () => res.send(users.map(u => u.name).join(', ')),
  });
});

// Send file with options
res.sendFile('/path/to/file.pdf', {
  maxAge: '1d',
  lastModified: true,
  dotfiles: 'deny',
}, (err) => {
  if (err) {
    res.status(500).json({ error: 'Failed to send file' });
  }
});

// Range requests (for large files)
res.sendFile('/path/to/large-video.mp4', {
  acceptRanges: true,
});
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Routing]] — Роутинг
- [[Express-Middleware-Core]] — Middleware
