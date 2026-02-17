---
created: 2026-02-17
tags:
  - nextjs
  - react
  - app-router
  - server-components
aliases:
  - Next.js 14 App Router
  - Next.js Server Components
related:
  - NextJS-Cheatsheet
  - React-MOC
  - React-Query
---

# Next.js 14+ — App Router & Server Components

> [!SUMMARY] Обзор
> Next.js 14+ с App Router: Server Components, Server Actions, streaming, partial prerendering.

---

## 📦 Установка

```bash
npx create-next-app@latest my-app
# Select: TypeScript, ESLint, Tailwind, App Router

cd my-app
npm run dev
```

---

## 🏗️ App Router Structure

```
app/
├── layout.tsx              # Root layout (required)
├── page.tsx                # Home page
├── loading.tsx             # Loading UI
├── error.tsx               # Error boundary
├── not-found.tsx           # 404 page
├── globals.css             # Global styles
├── (auth)/                 # Route group (not in URL)
│   ├── login/
│   │   └── page.tsx
│   └── register/
│       └── page.tsx
├── (shop)/
│   ├── products/
│   │   ├── page.tsx
│   │   └── [slug]/
│   │       └── page.tsx
│   └── cart/
│       └── page.tsx
├── api/                    # API routes
│   └── users/
│       └── route.ts
└── dashboard/
    ├── layout.tsx          # Nested layout
    └── page.tsx
```

---

## 🖥️ Server Components (RSC)

```typescript
// app/users/page.tsx
// Server Component by default (async)

import { db } from '@/lib/db';

export default async function UsersPage() {
  const users = await db.user.findMany();
  
  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}

// Passing data to Client Component
import UserClient from './user-client';

export default async function UsersPage() {
  const users = await db.user.findMany();
  
  return <UserClient users={users} />;
}
```

---

## 🎯 Client Components

```typescript
// app/users/user-client.tsx
'use client';  // Required at top

import { useState } from 'react';
import { User } from '@prisma/client';

interface UserClientProps {
  users: User[];
}

export default function UserClient({ users }: UserClientProps) {
  const [filter, setFilter] = useState('');
  
  const filtered = users.filter(u => 
    u.name.toLowerCase().includes(filter.toLowerCase())
  );
  
  return (
    <div>
      <input 
        value={filter} 
        onChange={e => setFilter(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {filtered.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 🔄 Server Actions

```typescript
// app/actions/users.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { db } from '@/lib/db';

export async function createUser(formData: FormData) {
  const email = formData.get('email') as string;
  const name = formData.get('name') as string;
  
  await db.user.create({
    data: { email, name },
  });
  
  revalidatePath('/users');  // Revalidate cache
  redirect('/users');        // Redirect
}

export async function deleteUser(id: string) {
  await db.user.delete({ where: { id } });
  revalidatePath('/users');
}

// With validation
export async function updateUser(id: string, formData: FormData) {
  const email = formData.get('email') as string;
  const name = formData.get('name') as string;
  
  // Validation
  if (!email.includes('@')) {
    return { error: 'Invalid email' };
  }
  
  await db.user.update({
    where: { id },
    data: { email, name },
  });
  
  revalidatePath('/users');
  return { success: true };
}
```

### Using Server Actions

```typescript
// app/users/create-form.tsx
'use client';

import { createUser } from '@/app/actions/users';

export function CreateUserForm() {
  return (
    <form action={createUser}>
      <input name="email" type="email" required />
      <input name="name" type="text" required />
      <button type="submit">Create</button>
    </form>
  );
}

// With useFormStatus
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Creating...' : 'Create'}
    </button>
  );
}

// With useFormState
import { useFormState } from 'react-dom';

export function UpdateForm({ id }: { id: string }) {
  const [state, dispatch] = useFormState(
    updateUser.bind(null, id),
    { error: null, success: false }
  );
  
  return (
    <form action={dispatch}>
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Updated!</p>}
      <input name="email" type="email" />
      <input name="name" type="text" />
      <button type="submit">Update</button>
    </form>
  );
}
```

---

## ⏳ Loading & Streaming

```typescript
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="loading-spinner">
      <Spinner />
      <p>Loading...</p>
    </div>
  );
}

// app/dashboard/error.tsx
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

// app/dashboard/not-found.tsx
export default function NotFound() {
  return (
    <div>
      <h2>Not Found</h2>
      <p>Could not find requested resource</p>
    </div>
  );
}
```

### Suspense Boundaries

```typescript
// app/products/page.tsx
import { Suspense } from 'react';
import ProductList from './product-list';
import ProductSkeleton from './product-skeleton';

export default function ProductsPage() {
  return (
    <div>
      <h1>Products</h1>
      <Suspense fallback={<ProductSkeleton />}>
        <ProductList />
      </Suspense>
    </div>
  );
}

// app/products/product-list.tsx
export default async function ProductList() {
  const products = await db.product.findMany();
  
  return (
    <ul>
      {products.map(p => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

---

## 📡 API Routes (Route Handlers)

```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function GET() {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  
  const user = await db.user.create({
    data: body,
  });
  
  return NextResponse.json(user, { status: 201 });
}

export async function PUT(request: Request) {
  const { id, ...data } = await request.json();
  
  const user = await db.user.update({
    where: { id },
    data,
  });
  
  return NextResponse.json(user);
}

export async function DELETE(request: Request) {
  const { searchParams } = new URL(request.url);
  const id = searchParams.get('id');
  
  await db.user.delete({ where: { id } });
  
  return NextResponse.json({ deleted: true });
}
```

### Dynamic Routes

```typescript
// app/api/users/[id]/route.ts
import { NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await db.user.findUnique({
    where: { id: params.id },
  });
  
  if (!user) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }
  
  return NextResponse.json(user);
}
```

---

## 🎣 Data Fetching

```typescript
// Direct in Server Component
export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache',  // Default, cache indefinitely
    // cache: 'no-store',   // No cache, fetch every request
    // next: { revalidate: 3600 },  // ISR, revalidate every hour
  });
  
  return <div>...</div>;
}

// With unstable_cache
import { unstable_cache } from 'next/cache';

const getCachedUsers = unstable_cache(
  async () => db.user.findMany(),
  ['users'],
  { revalidate: 3600 }
);

export default async function Page() {
  const users = await getCachedUsers();
  return <div>...</div>;
}
```

---

## 🔗 Связанные заметки

- [[NextJS-Cheatsheet]] — Next.js (Pages Router)
- [[React-MOC]] — React
- [[React-Query]] — TanStack Query
