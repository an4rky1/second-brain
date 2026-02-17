---
created: 2026-02-16
tags:
  - api
  - trpc
  - typescript
aliases:
  - tRPC API
  - tRPC End-to-End Types
related:
  - TypeScript-Cheatsheet
  - NestJS-Cheatsheet
  - React-Cheatsheet
---

# tRPC API

> [!SUMMARY] Обзор
> tRPC — end-to-end type safety для TypeScript приложений. Без code generation, работает на уровне типов. Идеален для fullstack TypeScript проектов.

---

## 📚 Основы

```
┌─────────────────────────────────────────────────────┐
│  tRPC vs GraphQL vs REST                             │
│                                                      │
│  tRPC:                                              │
│  ├─ End-to-end type safety (TypeScript)             │
│  ├─ Нет code generation                             │
│  ├─ Только TypeScript                               │
│  └─ Простая настройка                               │
│                                                      │
│  Use when: Fullstack TypeScript проект              │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Backend Setup

### Router Definition

```typescript
// server/trpc/router.ts
import { initTRPC, TRPCError } from '@trpc/server';
import { z } from 'zod';

const t = initTRPC.create();

// Base router and procedure
export const router = t.router;
export const publicProcedure = t.procedure;

// Auth middleware
const isAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({
    ctx: { ...ctx, user: ctx.user },
  });
});

export const protectedProcedure = t.procedure.use(isAuthed);

// User router
export const userRouter = router({
  // Query
  list: publicProcedure.query(async ({ ctx }) => {
    return ctx.prisma.user.findMany();
  }),
  
  byId: publicProcedure
    .input(z.object({
      id: z.string().uuid(),
    }))
    .query(async ({ ctx, input }) => {
      return ctx.prisma.user.findUnique({
        where: { id: input.id },
      });
    }),
  
  // Mutation
  create: publicProcedure
    .input(z.object({
      name: z.string().min(2),
      email: z.string().email(),
      password: z.string().min(8),
    }))
    .mutation(async ({ ctx, input }) => {
      const hashedPassword = await bcrypt.hash(input.password, 10);
      return ctx.prisma.user.create({
        data: {
          ...input,
          password: hashedPassword,
        },
      });
    }),
  
  update: protectedProcedure
    .input(z.object({
      id: z.string().uuid(),
      name: z.string().min(2).optional(),
      email: z.string().email().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      return ctx.prisma.user.update({
        where: { id: input.id },
        data: input,
      });
    }),
  
  delete: protectedProcedure
    .input(z.object({ id: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      return ctx.prisma.user.delete({
        where: { id: input.id },
      });
    }),
});

// Post router
export const postRouter = router({
  list: publicProcedure
    .input(z.object({
      cursor: z.string().nullish(),
      limit: z.number().min(1).max(100).default(20),
    }))
    .query(async ({ ctx, input }) => {
      const items = await ctx.prisma.post.findMany({
        take: input.limit + 1,
        cursor: input.cursor ? { id: input.cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });
      
      let nextCursor: typeof input.cursor = undefined;
      if (items.length > input.limit) {
        const nextItem = items.pop();
        nextCursor = nextItem?.id;
      }
      
      return { items, nextCursor };
    }),
  
  byId: publicProcedure
    .input(z.object({ id: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      return ctx.prisma.post.findUnique({
        where: { id: input.id },
        include: { author: true, comments: true },
      });
    }),
  
  create: protectedProcedure
    .input(z.object({
      title: z.string().min(1),
      content: z.string().min(1),
      tags: z.array(z.string()).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      return ctx.prisma.post.create({
        data: {
          ...input,
          authorId: ctx.user.id,
        },
      });
    }),
});

// Root router
export const appRouter = router({
  user: userRouter,
  post: postRouter,
  
  // Health check
  health: publicProcedure.query(() => {
    return { status: 'ok', timestamp: new Date() };
  }),
});

export type AppRouter = typeof appRouter;
```

### Server Setup (Express)

```typescript
// server/index.ts
import express from 'express';
import * as trpcExpress from '@trpc/server/adapters/express';
import { appRouter } from './trpc/router';
import { prisma } from './prisma';

const app = express();

app.use('/trpc', trpcExpress.createExpressMiddleware({
  router: appRouter,
  createContext: async ({ req }) => {
    // Auth from header
    const token = req.headers.authorization?.split(' ')[1];
    const user = token ? await verifyToken(token) : null;
    
    return { user, prisma };
  },
}));

app.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});
```

---

## 💻 Frontend Setup

### React + Query

```typescript
// client/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../shared/api';

export const trpc = createTRPCReact<AppRouter>();

// hooks/useTrpc.ts
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { useState } from 'react';
import { trpc } from './trpc';

function getBaseUrl() {
  return process.env.NODE_ENV === 'production'
    ? 'https://api.example.com'
    : 'http://localhost:3000';
}

export function TrpcProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: `${getBaseUrl()}/trpc`,
          headers: () => {
            const token = localStorage.getItem('token');
            return {
              authorization: token ? `Bearer ${token}` : undefined,
            };
          },
        }),
      ],
    })
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

### Usage Examples

```typescript
// components/UsersList.tsx
function UsersList() {
  const { data: users, isLoading } = trpc.user.list.useQuery();
  const createUser = trpc.user.create.useMutation();

  if (isLoading) return <Loading />;

  return (
    <div>
      {users?.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
      <button
        onClick={() => createUser.mutate({
          name: 'New User',
          email: 'new@example.com',
          password: 'password123',
        })}
      >
        Create User
      </button>
    </div>
  );
}

// components/PostInfinite.tsx
function PostInfinite() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = trpc.post.list.useInfiniteQuery(
    { limit: 10 },
    {
      getNextPageOut: (lastPage) => ({ cursor: lastPage.nextCursor }),
    }
  );

  const posts = data?.pages.flatMap(page => page.items) || [];

  return (
    <div>
      {posts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}

// components/PostForm.tsx
function PostForm() {
  const utils = trpc.useContext();
  
  const createPost = trpc.post.create.useMutation({
    onSuccess: () => {
      // Invalidate queries
      utils.post.list.invalidate();
    },
  });

  const handleSubmit = (data: FormData) => {
    createPost.mutate({
      title: data.title,
      content: data.content,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" />
      <textarea name="content" />
      <button type="submit">Create</button>
    </form>
  );
}
```

---

## 📡 Subscriptions

```typescript
// server/trpc/router.ts
import { observable } from '@trpc/server/observable';

export const notificationRouter = router({
  onUserCreated: publicProcedure.subscription(() => {
    return observable<{ id: string; name: string }>((emit) => {
      const onUserCreated = (user: { id: string; name: string }) => {
        emit.next(user);
      };

      // Subscribe to event emitter
      eventEmitter.on('user.created', onUserCreated);

      return () => {
        eventEmitter.off('user.created', onUserCreated);
      };
    });
  }),
});

// Frontend
function UserNotifications() {
  trpc.notification.onUserCreated.useSubscription(undefined, {
    onData: (user) => {
      console.log('New user created:', user);
      // Show toast notification
    },
    onError: (error) => {
      console.error('Subscription error:', error);
    },
  });

  return <div>Listening for new users...</div>;
}
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Zod валидация
.input(z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  age: z.number().min(0).max(150),
}))

// 2. Error handling
import { TRPCError } from '@trpc/server';

throw new TRPCError({
  code: 'NOT_FOUND',
  message: 'User not found',
  cause: error,
});

// 3. Middleware для auth
const isAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({ ctx: { ...ctx, user: ctx.user } });
});

// 4. Context typing
type Context = {
  user: { id: string; role: string } | null;
  prisma: PrismaClient;
};

// 5. Router organization
export const appRouter = router({
  user: userRouter,
  post: postRouter,
  auth: authRouter,
});
```

### ❌ Не делать

```typescript
// 1. Избегать сложной логики в resolvers
// Логика должна быть в сервисах

// 2. Не пропускать валидацию
.input(z.any())  // ❌

// 3. Не забывать про error handling
try {
  await db.user.create(data);
} catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    throw new TRPCError({
      code: 'CONFLICT',
      message: 'Email already exists',
    });
  }
  throw error;
}
```

---

## 🔗 Связанные заметки

- [[TypeScript-Cheatsheet]] — TypeScript типы
- [[React-Cheatsheet]] — React hooks
- [[NestJS-Cheatsheet]] — NestJS интеграция
- [[GraphQL-API]] — GraphQL альтернатива

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Zod для валидации** — типобезопасные input
> 2. **Middleware для auth** — переиспользуйте логику
> 3. **React Query кэш** — настраивайте staleTime
> 4. **Error codes** — используйте TRPCError codes
> 5. **Invalidate queries** — обновляйте кэш после мутаций
