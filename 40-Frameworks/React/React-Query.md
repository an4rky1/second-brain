---
created: 2026-02-17
tags:
  - react
  - react-query
  - tanstack-query
  - data-fetching
aliases:
  - React Query
  - TanStack Query
related:
  - React-MOC
  - React-Data-Fetching
  - TypeScript-DataFetching
---

# React Query (TanStack Query)

> [!SUMMARY] Обзор
> TanStack Query — библиотека для управления серверным состоянием. Кэширование, синхронизация, optimistic updates.

---

## 📦 Установка

```bash
npm install @tanstack/react-query
npm install -D @tanstack/react-query-devtools
```

---

## 🔧 Setup

```typescript
// App.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,  // 5 minutes
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

## 📥 Basic Queries

```typescript
import { useQuery } from '@tanstack/react-query';

// Basic query
function Users() {
  const { data, error, isLoading, isError } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await fetch('/api/users');
      return response.json();
    },
  });

  if (isLoading) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

// With parameters
function User({ userId }) {
  const { data, isLoading } = useQuery({
    queryKey: ['users', userId],  // Array for dynamic keys
    queryFn: async () => {
      const response = await fetch(`/api/users/${userId}`);
      return response.json();
    },
    enabled: !!userId,  // Only fetch if userId exists
    staleTime: 10 * 60 * 1000,  // 10 minutes
  });
}

// With axios
function Posts() {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: () => axios.get('/posts').then(res => res.data),
  });
}
```

---

## 📤 Mutations

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUser() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newUser) => axios.post('/users', newUser),
    
    // Optimistic update
    onMutate: async (newUser) => {
      await queryClient.cancelQueries({ queryKey: ['users'] });
      
      const previousUsers = queryClient.getQueryData(['users']);
      
      queryClient.setQueryData(['users'], (old) => [...old, newUser]);
      
      return { previousUsers };
    },
    
    // On error: rollback
    onError: (err, newUser, context) => {
      queryClient.setQueryData(['users'], context.previousUsers);
    },
    
    // On success: invalidate
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  return (
    <button onClick={() => mutation.mutate({ name: 'John' })}>
      Create User
    </button>
  );
}
```

---

## 🔄 Pagination

```typescript
function Projects({ page }) {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ['projects', page],
    queryFn: () => fetch(`/api/projects?page=${page}`).then(res => res.json()),
    keepPreviousData: true,  // Keep old data while loading
  });

  return (
    <div>
      {isLoading && <div>Loading...</div>}
      {isError && <div>Error: {error.message}</div>}
      {data && (
        <>
          <div>{data.name}</div>
          <span>Current Page: {page}</span>
        </>
      )}
    </div>
  );
}

// With infinite scroll
function InfiniteProjects() {
  const fetchProjects = async ({ pageParam = 0 }) => {
    const res = await fetch(`/api/projects?cursor=${pageParam}`);
    return res.json();
  };

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  return (
    <div>
      {data.pages.map((page) => (
        <div key={page.id}>{page.name}</div>
      ))}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : 'Load More'}
      </button>
    </div>
  );
}
```

---

## ⚡ Advanced Features

### Prefetching

```typescript
// Prefetch on hover
function Project({ id }) {
  const queryClient = useQueryClient();

  return (
    <div
      onMouseEnter={() => {
        queryClient.prefetchQuery({
          queryKey: ['project', id],
          queryFn: () => fetch(`/api/projects/${id}`).then(res => res.json()),
          staleTime: 5 * 60 * 1000,
        });
      }}
    >
      Project {id}
    </div>
  );
}

// Prefetch in loader (React Router)
async function loader({ params }) {
  await queryClient.prefetchQuery({
    queryKey: ['project', params.id],
    queryFn: () => fetch(`/api/projects/${params.id}`).then(res => res.json()),
  });
  return null;
}
```

### Dependent Queries

```typescript
// Fetch user, then fetch user's posts
function UserPosts({ userId }) {
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(res => res.json()),
  });

  const { data: posts } = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetch(`/api/users/${userId}/posts`).then(res => res.json()),
    enabled: !!user,  // Only fetch if user exists
  });
}
```

### Parallel Queries

```typescript
// Automatic parallel
function Dashboard() {
  const { data: users } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  const { data: posts } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
  const { data: comments } = useQuery({ queryKey: ['comments'], queryFn: fetchComments });

  return <div>...</div>;
}

// Manual parallel with Promise.all
function Dashboard() {
  const { data } = useQuery({
    queryKey: ['dashboard'],
    queryFn: async () => {
      const [users, posts, comments] = await Promise.all([
        fetchUsers(),
        fetchPosts(),
        fetchComments(),
      ]);
      return { users, posts, comments };
    },
  });
}
```

### Real-time Updates

```typescript
function Comments({ postId }) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const ws = new WebSocket(`ws://localhost:8080/comments/${postId}`);
    
    ws.onmessage = (event) => {
      const comment = JSON.parse(event.data);
      
      // Optimistic update
      queryClient.setQueryData(['comments', postId], (old) => [
        ...old,
        comment,
      ]);
    };

    return () => ws.close();
  }, [postId, queryClient]);

  const { data: comments } = useQuery({
    queryKey: ['comments', postId],
    queryFn: () => fetch(`/api/posts/${postId}/comments`).then(res => res.json()),
  });

  return <div>...</div>;
}
```

---

## 🎯 Best Practices

### Query Keys

```typescript
// ✅ Good: Array keys
useQuery({ queryKey: ['users'], ... })
useQuery({ queryKey: ['users', userId], ... })
useQuery({ queryKey: ['users', userId, 'posts'], ... })

// ❌ Bad: String keys
useQuery({ queryKey: 'users', ... })
useQuery({ queryKey: `users-${userId}`, ... })

// Factory pattern
const queryKeys = {
  users: {
    all: ['users'],
    lists: () => [...queryKeys.users.all, 'list'],
    list: (filters) => [...queryKeys.users.lists(), filters],
    details: () => [...queryKeys.users.all, 'detail'],
    detail: (id) => [...queryKeys.users.details(), id],
  },
};

useQuery({ queryKey: queryKeys.users.list({ page: 1 }), ... })
useQuery({ queryKey: queryKeys.users.detail(1), ... })
```

### Configuration

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,      // 5 minutes
      retry: 1,                       // Retry once
      refetchOnWindowFocus: false,   // Don't refocus on window focus
      refetchOnReconnect: true,      // Refetch on reconnect
      cacheTime: 10 * 60 * 1000,     // 10 minutes in cache
      suspense: false,               // Don't use Suspense
    },
    mutations: {
      retry: 0,                       // Don't retry mutations
    },
  },
});
```

---

## 📋 Environment

```typescript
// config/query.config.ts
export const queryConfig = {
  staleTime: {
    short: 1 * 60 * 1000,      // 1 minute
    medium: 5 * 60 * 1000,     // 5 minutes
    long: 15 * 60 * 1000,      // 15 minutes
  },
  cacheTime: {
    short: 2 * 60 * 1000,      // 2 minutes
    medium: 10 * 60 * 1000,    // 10 minutes
    long: 30 * 60 * 1000,      // 30 minutes
  },
};
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — React индекс
- [[React-Data-Fetching]] — Data fetching patterns
- [[TypeScript-DataFetching]] — TypeScript data fetching
