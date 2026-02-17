---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - data-fetching
  - tanstack-query
  - swr
aliases:
  - React Data Fetching
  - TanStack Query SWR
related:
  - React-MOC
  - React-State-Management
---

# React — Data Fetching

> [!SUMMARY] Обзор
> Работа с серверными данными: TanStack Query (React Query), SWR. Кэширование, синхронизация, оптимистичные обновления.

---

## TanStack Query (React Query)

### Установка и настройка

```bash
npm install @tanstack/react-query
```

```tsx
// app/provider.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 минут
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

export function AppProvider({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### Базовое использование

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Query
function UsersPage() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['users'],
    queryFn: () => api.getUsers(),
  });
  
  if (isLoading) return <Loading />;
  if (error) return <Error message={error.message} />;
  
  return (
    <div>
      <button onClick={() => refetch()}>Refresh</button>
      {data?.map(user => <UserCard key={user.id} user={user} />)}
    </div>
  );
}

// Query с параметрами
function UserPage({ userId }: { userId: string }) {
  const { data, isLoading } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => api.getUser(userId),
    enabled: !!userId, // Только если userId есть
  });
  
  if (isLoading) return <Loading />;
  return <div>{data?.name}</div>;
}
```

### Mutation

```tsx
function CreateUserForm() {
  const queryClient = useQueryClient();
  
  const mutation = useMutation({
    mutationFn: (data: CreateUserDto) => api.createUser(data),
    onSuccess: (newUser) => {
      // Инвалидация query
      queryClient.invalidateQueries({ queryKey: ['users'] });
      
      // Или оптимистичное обновление
      queryClient.setQueryData(['users'], (old: User[]) => [
        ...old,
        newUser,
      ]);
    },
  });
  
  const handleSubmit = (data: CreateUserDto) => {
    mutation.mutate(data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create'}
      </button>
      {mutation.isError && <Error message={mutation.error.message} />}
    </form>
  );
}
```

### Optimistic Update

```tsx
function TodosPage() {
  const queryClient = useQueryClient();
  
  const { data: todos } = useQuery({
    queryKey: ['todos'],
    queryFn: () => api.getTodos(),
  });
  
  const addMutation = useMutation({
    mutationFn: (todo: CreateTodoDto) => api.createTodo(todo),
    onMutate: async (newTodo) => {
      // Отменяем текущие refetch
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      
      // Сохраняем предыдущее состояние
      const previousTodos = queryClient.getQueryData(['todos']);
      
      // Оптимистичное обновление
      queryClient.setQueryData(['todos'], (old: Todo[]) => [
        ...old,
        { ...newTodo, id: Date.now(), temp: true },
      ]);
      
      return { previousTodos };
    },
    onError: (err, newTodo, context) => {
      // Rollback при ошибке
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    onSettled: () => {
      // Синхронизируем с сервером
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
  
  const toggleMutation = useMutation({
    mutationFn: ({ id, completed }: { id: number; completed: boolean }) =>
      api.updateTodo(id, { completed }),
    onMutate: async ({ id, completed }) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData(['todos']);
      
      queryClient.setQueryData(['todos'], (old: Todo[]) =>
        old.map(todo => todo.id === id ? { ...todo, completed } : todo)
      );
      
      return { previousTodos };
    },
    onError: (err, vars, context) => {
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
  });
  
  return (
    <ul>
      {todos?.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={(e) => toggleMutation.mutate({
              id: todo.id,
              completed: e.target.checked,
            })}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

### Infinite Query

```tsx
function InfinitePosts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 1 }) => api.getPosts({ page: pageParam, limit: 10 }),
    getNextPageParam: (lastPage, pages) => {
      // Если есть ещё данные, возвращаем следующую страницу
      return lastPage.length === 10 ? pages.length + 1 : undefined;
    },
  });
  
  if (isLoading) return <Loading />;
  
  return (
    <div>
      {data.pages.map((page, i) => (
        <div key={i}>
          {page.map(post => <PostCard key={post.id} post={post} />)}
        </div>
      ))}
      
      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

### Prefetch

```tsx
// Prefetch при наведении
function PostList({ posts }: { posts: Post[] }) {
  const queryClient = useQueryClient();
  
  const prefetchPost = (postId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['posts', postId],
      queryFn: () => api.getPost(postId),
      staleTime: 1000 * 60 * 5,
    });
  };
  
  return (
    <ul>
      {posts.map(post => (
        <li
          key={post.id}
          onMouseEnter={() => prefetchPost(post.id)}
        >
          <Link to={`/posts/${post.id}`}>{post.title}</Link>
        </li>
      ))}
    </ul>
  );
}

// Prefetch в loader (React Router)
export async function loader() {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery({
    queryKey: ['users'],
    queryFn: () => api.getUsers(),
  });
  return null;
}
```

### Dependent Queries

```tsx
// Запрос зависит от данных другого запроса
function UserOrders({ userId }: { userId: string }) {
  // Сначала получаем пользователя
  const { data: user } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => api.getUser(userId),
  });
  
  // Заказы только если пользователь premium
  const { data: orders } = useQuery({
    queryKey: ['users', userId, 'orders'],
    queryFn: () => api.getUserOrders(userId),
    enabled: user?.isPremium, // Только если premium
  });
  
  return <div>{/* ... */}</div>;
}

// Параллельные запросы
function Dashboard() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: () => api.getUsers(),
  });
  
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: () => api.getPosts(),
  });
  
  const { data: stats } = useQuery({
    queryKey: ['stats'],
    queryFn: () => api.getStats(),
  });
  
  // Или useQueries для динамического количества
  const userQueries = useQueries({
    queries: userIds.map(id => ({
      queryKey: ['users', id],
      queryFn: () => api.getUser(id),
    })),
  });
}
```

---

## SWR

### Установка и настройка

```bash
npm install swr
```

```tsx
// app/provider.tsx
import { SWRConfig } from 'swr';

export function AppProvider({ children }: { children: React.ReactNode }) {
  return (
    <SWRConfig
      value={{
        fetcher: (url: string) => fetch(url).then(r => r.json()),
        dedupingInterval: 2000,
        staleWhileRevalidate: 5000,
      }}
    >
      {children}
    </SWRConfig>
  );
}
```

### Базовое использование

```tsx
import useSWR from 'swr';

function UsersPage() {
  const { data, error, isLoading, mutate } = useSWR<User[]>(
    '/api/users',
    fetcher
  );
  
  if (isLoading) return <Loading />;
  if (error) return <Error message={error.message} />;
  
  return (
    <div>
      <button onClick={() => mutate()}>Refresh</button>
      {data?.map(user => <UserCard key={user.id} user={user} />)}
    </div>
  );
}

// С параметрами
function UserPage({ userId }: { userId: string }) {
  const { data, isLoading } = useSWR(
    userId ? `/api/users/${userId}` : null,
    fetcher
  );
  
  if (isLoading) return <Loading />;
  return <div>{data?.name}</div>;
}
```

### Mutation

```tsx
import useSWRMutation from 'swr/mutation';

function CreateUserForm() {
  const { trigger, isExecuting, error } = useSWRMutation(
    '/api/users',
    (url, { arg }: { arg: CreateUserDto }) =>
      fetch(url, {
        method: 'POST',
        body: JSON.stringify(arg),
      }).then(r => r.json())
  );
  
  const handleSubmit = (data: CreateUserDto) => {
    trigger(data, {
      optimisticData: (currentData) => [
        ...(currentData || []),
        { ...data, id: Date.now() },
      ],
      rollbackOnError: true,
      populateCache: true,
      revalidate: true,
    });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" />
      <button type="submit" disabled={isExecuting}>
        {isExecuting ? 'Creating...' : 'Create'}
      </button>
      {error && <Error message={error.message} />}
    </form>
  );
}
```

---

## Сравнение

| Функция | TanStack Query | SWR |
|---------|---------------|-----|
| Размер | ~13kb | ~6kb |
| Devtools | ✅ | ❌ |
| Mutations | useMutation | useSWRMutation |
| Infinite queries | useInfiniteQuery | useSWRInfinite |
| Prefetch | prefetchQuery | preload |
| Caching | Продвинутый | Базовый |
| Focus revalidation | ✅ | ✅ |
| Offline support | ✅ (с persist) | ✅ |

---

## Best Practices

### ✅ Делать

```tsx
// 1. Структурируйте query keys
queryKey: ['users']
queryKey: ['users', userId]
queryKey: ['users', userId, 'posts']
queryKey: ['posts', { page, limit }]

// 2. Используйте staleTime
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  staleTime: 5 * 60 * 1000, // 5 минут
});

// 3. Инвалидируйте после мутаций
const mutation = useMutation({
  mutationFn: createUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});

// 4. Обработайте ошибки
const { error } = useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  retry: (failureCount, error) => {
    if (error.status === 404) return false;
    return failureCount < 3;
  },
});

// 5. Используйте suspense для простых случаев
const { data } = useSuspenseQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
});

// В ErrorBoundary
<Suspense fallback={<Loading />}>
  <ErrorBoundary fallback={<Error />}>
    <UsersPage />
  </ErrorBoundary>
</Suspense>
```

### ❌ Не делать

```tsx
// 1. Не храните server state в локальном state
const [users, setUsers] = useState([]); // ❌
useEffect(() => {
  fetchUsers().then(setUsers);
}, []);

// Используйте useQuery ✅
const { data: users } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });

// 2. Не делайте запросы в useEffect
useEffect(() => {
  fetch('/api/users').then(setData); // ❌
}, []);

// Используйте useQuery ✅

// 3. Не забывайте про query keys
useQuery({
  queryKey: ['users'], // ❌ — не зависит от userId
  queryFn: () => fetchUser(userId),
});

useQuery({
  queryKey: ['users', userId], // ✅
  queryFn: () => fetchUser(userId),
});
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-State-Management]] — Zustand, Redux
- [[React-Routing]] — React Router
- [[React-Hooks]] — кастомные хуки

---

> [!TIP] Совет
>
> 1. **TanStack Query для большинства случаев** — мощный, много фич
> 2. **SWR для простых проектов** — легче, проще
> 3. **Всегда используйте query keys** — для кэширования и инвалидации
> 4. **Optimistic updates для UX** — мгновенный отклик
> 5. **Prefetch для навигации** — предзагрузка при наведении
