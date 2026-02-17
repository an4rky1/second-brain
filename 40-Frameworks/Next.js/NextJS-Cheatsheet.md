---
created: 2026-02-16
tags:
  - cheat-sheet
  - nextjs
  - react
  - ssr
  - frontend
aliases:
  - Next.js Cheatsheet
  - NextJS Reference
related:
  - React-Cheatsheet
  - TypeScript-Cheatsheet
  - Vercel-Deployment
---

# Next.js — Полная шпаргалка

> [!SUMMARY] Обзор
> Next.js — React фреймворк для production. SSR, SSG, ISR, API routes, оптимизация изображений, роутинг из коробки. Стандарт для React приложений в 2024+.

---

## 📚 Теория

### Рендеринг стратегии

| Стратегия | Когда | Где |
|-----------|-------|-----|
| **CSR** (Client-Side) | Динамический контент, после загрузки | useEffect, useState |
| **SSR** (Server-Side) | Персонализированный контент | getServerSideProps |
| **SSG** (Static) | Статический контент | getStaticProps |
| **ISR** (Incremental) | Обновляемый статический | getStaticProps + revalidate |
| **Streaming** | Частичная загрузка | Suspense + React 18 |

### App Router vs Pages Router

```
Pages Router (старый):
pages/
├── index.tsx
├── about.tsx
└── api/
    └── users.ts

App Router (новый, Next.js 13+):
app/
├── layout.tsx
├── page.tsx
├── about/
│   └── page.tsx
└── api/
    └── users/
        └── route.ts
```

---

## ⚡ Быстрый старт

```bash
# Создание проекта
npx create-next-app@latest my-app --typescript --tailwind --eslint --app --src-dir

# Структура
cd my-app
npm run dev    # http://localhost:3000
npm run build
npm run start  # Production server

# Деплой на Vercel
npm i -g vercel
vercel
```

---

## 🔧 Практические примеры

### App Router

```tsx
// app/layout.tsx — Root Layout
export const metadata: Metadata = {
  title: 'My App',
  description: 'My application',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

// app/page.tsx — Страница
export default function HomePage() {
  return <h1>Hello World</h1>;
}

// app/blog/[slug]/page.tsx — Dynamic Route
interface Props {
  params: { slug: string };
}

export default async function BlogPost({ params }: Props) {
  const post = await getPost(params.slug);
  return <article>{post.content}</article>;
}

// app/blog/[slug]/page.tsx — Generate Static Params
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map(post => ({ slug: post.slug }));
}

// app/dashboard/layout.tsx — Nested Layout
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}

// app/dashboard/@analytics — Parallel Routes
export default function Analytics({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

// app/dashboard/(auth) — Route Groups (не влияют на URL)
// app/(auth)/login/page.tsx → /login
```

### Data Fetching

```tsx
// Server Component (по умолчанию)
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache',  // SSG
    // cache: 'no-store',   // SSR (dynamic)
    // next: { revalidate: 3600 }, // ISR (1 hour)
  });
  const posts = await data.json();
  
  return <Posts posts={posts} />;
}

// Client Component
'use client';

import { useEffect, useState } from 'react';

export default function ClientComponent() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);
  }, []);
  
  return <div>{data}</div>;
}

// useQuery (React Query / TanStack Query)
'use client';

import { useQuery } from '@tanstack/react-query';

function ClientComponent() {
  const { data, isLoading } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(r => r.json()),
  });
  
  if (isLoading) return <Loading />;
  return <Posts posts={data} />;
}

// Server + Client паттерн
// Server Component
async function Page() {
  const data = await getData();
  return <ClientComponent data={data} />;
}

// Client Component
'use client';

export default function ClientComponent({ data }: { data: Data }) {
  // Можно использовать state, effects
  return <div>{data.title}</div>;
}
```

### API Routes (Route Handlers)

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const query = searchParams.get('q');
  
  const users = await getUsers(query);
  return NextResponse.json(users);
}

// POST
export async function POST(request: NextRequest) {
  const body = await request.json();
  
  // Валидация
  if (!body.email) {
    return NextResponse.json(
      { error: 'Email required' },
      { status: 400 }
    );
  }
  
  const user = await createUser(body);
  return NextResponse.json(user, { status: 201 });
}

// PUT/PATCH
export async function PUT(request: NextRequest) {
  const { id } = await request.json();
  const user = await updateUser(id);
  return NextResponse.json(user);
}

// DELETE
export async function DELETE(request: NextRequest) {
  const { id } = await request.json();
  await deleteUser(id);
  return NextResponse.json({ success: true });
}

// Dynamic Route
// app/api/users/[id]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await getUser(params.id);
  return NextResponse.json(user);
}

// Middleware
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Auth check
  const token = request.cookies.get('token');
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  
  // Add headers
  const response = NextResponse.next();
  response.headers.set('X-Custom-Header', 'value');
  return response;
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

### Оптимизация

```tsx
// Image Component
import Image from 'next/image';

<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={630}
  priority  // LCP изображение
  quality={85}
  sizes="(max-width: 768px) 100vw, 1200px"
/>;

// Remote Images
// next.config.js
module.exports = {
  images: {
    domains: ['images.example.com'],
  },
};

// Font Optimization
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export default function Layout({ children }) {
  return <body className={inter.className}>{children}</body>;
}

// Script Optimization
import Script from 'next/script';

<Script
  src="https://analytics.example.com/script.js"
  strategy="lazyOnload"  // after page load
/>;

// Dynamic Import (Code Splitting)
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(
  () => import('../components/HeavyComponent'),
  { 
    loading: () => <Loading />,
    ssr: false,  // Только клиент
  }
);

// Metadata
export const metadata: Metadata = {
  title: {
    default: 'My App',
    template: '%s | My App',
  },
  description: 'My application description',
  openGraph: {
    title: 'My App',
    description: 'Description',
    images: ['/og.png'],
  },
  twitter: {
    card: 'summary_large_image',
  },
};

// Canonical URL
import { Metadata } from 'next';

export const metadata: Metadata = {
  alternates: {
    canonical: 'https://example.com/page',
  },
};
```

### Кэширование

```ts
// Fetch cache options
fetch('https://api.example.com/data', {
  cache: 'force-cache',     // Кэшировать (SSG)
  cache: 'no-store',        // Не кэшировать (SSR)
  cache: 'no-cache',        // Кэш с валидацией
});

// next options
fetch('https://api.example.com/data', {
  next: {
    revalidate: 3600,       // ISR: обновить через 1 час
    tags: ['posts'],        // Cache tags для revalidateTag
  },
});

// Revalidate
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request: Request) {
  const { path, tag } = await request.json();
  
  if (path) {
    revalidatePath(path);
  }
  
  if (tag) {
    revalidateTag(tag);
  }
  
  return NextResponse.json({ revalidated: true });
}

// Server Actions
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title');
  
  await db.post.create({ data: { title } });
  revalidatePath('/posts');
}

// Использование в форме
<form action={createPost}>
  <input name="title" />
  <button type="submit">Create</button>
</form>
```

---

## 🎯 Best Practices

### ✅ Делать

```tsx
// 1. Server Components по умолчанию
// Используйте 'use client' только когда нужно state/effects

// 2. Streaming + Suspense
<Suspense fallback={<Loading />}>
  <HeavyComponent />
</Suspense>;

// 3. Error Boundaries
// app/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// 4. Loading States
// app/dashboard/loading.tsx
export default function Loading() {
  return <LoadingSpinner />;
}

// 5. SEO Metadata
export const metadata: Metadata = {
  title: 'Page Title',
  description: 'Description',
};

// 6. Environment Variables
// .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgresql://...

// Использование
const url = process.env.NEXT_PUBLIC_API_URL;
```

### ❌ Не делать

```tsx
// 1. Избегать useEffect для data fetching
useEffect(() => {
  fetch('/api/data').then(...); // ❌ (водопад запросов)
}, []);

// Используйте Server Components или React Query ✅

// 2. Не используйте Client Components без необходимости
'use client'; // ❌ если не нужен state

// 3. Избегать больших bundle
import * as lodash from 'lodash'; // ❌
import debounce from 'lodash/debounce'; // ✅

// 4. Не забывайте про loading/error states
async function Page() {
  const data = await getData(); // ❌ нет loading state
  return <Data data={data} />;
}
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Hydration failed` | Мismatch SSR/CSR | Проверьте условный рендеринг, используйте useEffect |
| `Module not found` | Неправильный import | Проверьте путь, используйте alias |
| `getServerSideProps is not defined` | App Router | Используйте Server Components |
| `Image optimization error` | Не настроены domains | Добавьте в next.config.js |
| `Middleware infinite loop` | Redirect в middleware | Проверьте условия redirect |

---

## 🔗 Связанные заметки

- [[React-Cheatsheet]] — React основы
- [[TypeScript-Cheatsheet]] — TypeScript
- [[Vercel-Deployment]] — Деплой на Vercel

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **App Router > Pages Router** — используйте новый подход
> 2. **Server Components по умолчанию** — меньше JS на клиенте
> 3. **Streaming для UX** — показывайте контент постепенно
> 4. **Vercel для деплоя** — лучшая интеграция
> 5. **Next.js Image** — всегда используйте для оптимизации

> [!INFO] Полезные команды
> ```bash
> # Анализ bundle
> npm run build
> npx next-bundle-analyzer
> 
> # Проверка типов
> npx tsc --noEmit
> 
> # Линтинг
> npm run lint
> 
> # Деплой
> vercel --prod
> ```
