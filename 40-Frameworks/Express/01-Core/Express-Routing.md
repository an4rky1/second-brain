---
created: 2026-02-17
tags:
  - express
  - routing
  - routes
  - router
aliases:
  - Express Routing
  - Express Routes
related:
  - Express-MOC
  - Express-Getting-Started
---

# Express.js — Routing

> [!SUMMARY] Обзор
> Роутинг в Express: routes, params, query, handlers, router.

---

## 🎮 Basic Routes

```typescript
import express, { Request, Response } from 'express';

const app = express();

// GET /
app.get('/', (req: Request, res: Response) => {
  res.send('Hello World!');
});

// GET /json
app.get('/json', (req, res) => {
  res.json({ message: 'Hello JSON' });
});

// POST /users
app.post('/users', (req, res) => {
  const user = req.body;
  res.status(201).json({ message: 'User created', data: user });
});

// PUT /users/:id
app.put('/users/:id', (req, res) => {
  const { id } = req.params;
  const data = req.body;
  res.json({ message: `User ${id} updated`, data });
});

// DELETE /users/:id
app.delete('/users/:id', (req, res) => {
  const { id } = req.params;
  res.status(204).send();
});
```

---

## 📍 Route Parameters

```typescript
// Single param: /users/123
app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  res.json({ userId: id });
});

// Multiple params: /users/123/posts/456
app.get('/users/:userId/posts/:postId', (req, res) => {
  const { userId, postId } = req.params;
  res.json({ userId, postId });
});

// Optional param: /users/optional или /users/optional/123
app.get('/users/optional/:id?', (req, res) => {
  const { id } = req.params;
  res.json({ id: id || 'default' });
});

// Wildcard: /users/anything/here
app.get('/users/*', (req, res) => {
  const [0] = req.params;
  res.json({ path: [0] });
});

// Regex: /file/123.pdf
app.get('/file/:fileId(\\d+).:format(\\w+)', (req, res) => {
  const { fileId, format } = req.params;
  res.json({ fileId, format });
});
```

---

## 🔍 Query Parameters

```typescript
// /search?name=john&age=30&limit=10
app.get('/search', (req, res) => {
  const { name, age, limit } = req.query;
  res.json({ name, age, limit });
});

// Array query: /tags?tags=js&tags=node&tags=express
app.get('/tags', (req, res) => {
  const { tags } = req.query;
  // tags = ['js', 'node', 'express']
  res.json({ tags });
});

// Nested query: /filter[user][name]=john
app.get('/filter', (req, res) => {
  const { user } = req.query;
  // user = { name: 'john' }
  res.json({ user });
});

// Pagination: /users?page=1&limit=10&sort=name&order=asc
app.get('/users', (req, res) => {
  const { page = 1, limit = 10, sort = 'createdAt', order = 'desc' } = req.query;
  
  res.json({
    page: Number(page),
    limit: Number(limit),
    sort,
    order,
  });
});
```

---

## 🔄 Route Handlers

```typescript
// Single handler
app.get('/example', (req, res) => {
  res.json({ message: 'Done' });
});

// Multiple handlers
app.get('/example',
  (req, res, next) => {
    console.log('Handler 1');
    next();
  },
  (req, res, next) => {
    console.log('Handler 2');
    next();
  },
  (req, res) => {
    res.json({ message: 'Done!' });
  }
);

// Handler array
const handler1 = (req, res, next) => {
  console.log('Handler 1');
  next();
};

const handler2 = (req, res, next) => {
  console.log('Handler 2');
  next();
};

app.get('/example', [handler1, handler2], (req, res) => {
  res.json({ message: 'Done!' });
});
```

---

## 🚦 Router

```typescript
// src/routes/users.routes.ts
import { Router, Request, Response } from 'express';

const router = Router();

// GET /users
router.get('/', (req, res) => {
  res.json({ message: 'Get all users' });
});

// GET /users/:id
router.get('/:id', (req, res) => {
  res.json({ message: `Get user ${req.params.id}` });
});

// POST /users
router.post('/', (req, res) => {
  res.status(201).json({ message: 'User created' });
});

// PUT /users/:id
router.put('/:id', (req, res) => {
  res.json({ message: `Update user ${req.params.id}` });
});

// DELETE /users/:id
router.delete('/:id', (req, res) => {
  res.status(204).send();
});

export default router;
```

### Mounting Router

```typescript
// src/app.ts
import express from 'express';
import usersRouter from './routes/users.routes';
import postsRouter from './routes/posts.routes';
import authRouter from './routes/auth.routes';

const app = express();

// Mount routers
app.use('/api/users', usersRouter);
app.use('/api/posts', postsRouter);
app.use('/api/auth', authRouter);

// Or with path
app.use('/api/v1', usersRouter);
```

---

## ⚡ Route Methods

```typescript
// HTTP Methods
app.get('/path', handler);      // GET
app.post('/path', handler);     // POST
app.put('/path', handler);      // PUT
app.patch('/path', handler);    // PATCH
app.delete('/path', handler);   // DELETE
app.options('/path', handler);  // OPTIONS
app.head('/path', handler);     // HEAD

// All methods
app.all('/path', handler);

// Chaining
app.route('/users')
  .get(getUsers)
  .post(createUser)
  .put(updateUsers);

app.route('/users/:id')
  .get(getUser)
  .put(updateUser)
  .delete(deleteUser);
```

---

## 🔗 Связанные заметки

- [[Express-MOC]] — Express индекс
- [[Express-Getting-Started]] — Быстрый старт
- [[Express-Request-Response]] — Request/Response
