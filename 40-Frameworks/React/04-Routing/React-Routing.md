---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - routing
  - react-router
aliases:
  - React Routing
  - React Router v6
related:
  - React-MOC
  - React-Basics
  - React-Data-Fetching
---

# React — Routing (React Router)

> [!SUMMARY] Обзор
> Навигация в React приложениях: React Router v6+, route guards, loader, nested routes, lazy loading.

---

## Установка и настройка

```bash
npm install react-router-dom
```

```tsx
// main.tsx
import { BrowserRouter } from 'react-router-dom';
import { App } from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

---

## Базовые маршруты

### Route Configuration

```tsx
// App.tsx
import { Routes, Route, Navigate } from 'react-router-dom';

function App() {
  return (
    <Routes>
      <Route path="/" element={<HomePage />} />
      <Route path="/about" element={<AboutPage />} />
      <Route path="/users" element={<UsersPage />} />
      <Route path="/users/:id" element={<UserPage />} />
      <Route path="/posts/:postId/comments/:commentId" element={<CommentPage />} />
      
      {/* 404 */}
      <Route path="*" element={<NotFoundPage />} />
      
      {/* Redirect */}
      <Route path="/old-path" element={<Navigate to="/new-path" replace />} />
    </Routes>
  );
}
```

### Navigation

```tsx
import { Link, NavLink, useNavigate } from 'react-router-dom';

function Navigation() {
  const navigate = useNavigate();
  
  return (
    <nav>
      {/* Обычные ссылки */}
      <Link to="/about">About</Link>
      
      {/* С активным классом */}
      <NavLink
        to="/users"
        className={({ isActive }) => isActive ? 'active' : ''}
      >
        Users
      </NavLink>
      
      {/* Программная навигация */}
      <button onClick={() => navigate('/users')}>Go to Users</button>
      
      {/* С параметрами */}
      <button onClick={() => navigate('/users/123')}>User 123</button>
      
      {/* С query параметрами */}
      <button onClick={() => navigate('/search?q=react&page=2')}>Search</button>
      
      {/* Назад */}
      <button onClick={() => navigate(-1)}>Back</button>
      
      {/* Заменить текущую запись в истории */}
      <button onClick={() => navigate('/login', { replace: true })}>
        Login
      </button>
    </nav>
  );
}
```

---

## Параметры маршрута

### URL Parameters

```tsx
// Route
<Route path="/users/:id" element={<UserPage />} />

// Компонент
import { useParams } from 'react-router-dom';

function UserPage() {
  const { id } = useParams<'id'>();
  
  return <div>User ID: {id}</div>;
}

// С несколькими параметрами
<Route path="/posts/:postId/comments/:commentId" element={<CommentPage />} />

function CommentPage() {
  const { postId, commentId } = useParams<'postId' | 'commentId'>();
  // postId: string | undefined
  // commentId: string | undefined
}
```

### Query Parameters

```tsx
import { useSearchParams } from 'react-router-dom';

function SearchPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const q = searchParams.get('q');
  const page = searchParams.get('page') || '1';
  
  const handleSearch = (query: string) => {
    setSearchParams({ q: query, page: '1' });
  };
  
  const handlePageChange = (newPage: number) => {
    setSearchParams(prev => ({
      ...prev,
      page: newPage.toString(),
    }));
  };
  
  return (
    <div>
      <p>Query: {q}</p>
      <p>Page: {page}</p>
      <button onClick={() => handleSearch('react')}>Search React</button>
      <button onClick={() => handlePageChange(2)}>Next Page</button>
    </div>
  );
}
```

---

## Вложенные маршруты

### Nested Routes

```tsx
// App.tsx
<Route path="/dashboard" element={<DashboardLayout />}>
  <Route index element={<DashboardHome />} />
  <Route path="users" element={<DashboardUsers />} />
  <Route path="settings" element={<DashboardSettings />} />
  <Route path="users/:id" element={<DashboardUserDetail />} />
</Route>

// DashboardLayout.tsx
import { Outlet, NavLink } from 'react-router-dom';

function DashboardLayout() {
  return (
    <div className="dashboard">
      <nav>
        <NavLink to="/dashboard">Home</NavLink>
        <NavLink to="/dashboard/users">Users</NavLink>
        <NavLink to="/dashboard/settings">Settings</NavLink>
      </nav>
      
      {/* Дочерние маршруты рендерятся здесь */}
      <Outlet />
    </div>
  );
}
```

### Layout Route

```tsx
// App.tsx
<Route element={<MainLayout />}>
  <Route path="/" element={<HomePage />} />
  <Route path="/about" element={<AboutPage />} />
  
  {/* Защищённые маршруты */}
  <Route element={<RequireAuth />}>
    <Route path="/profile" element={<ProfilePage />} />
    <Route path="/settings" element={<SettingsPage />} />
  </Route>
</Route>

// MainLayout.tsx
function MainLayout() {
  return (
    <div>
      <Header />
      <main>
        <Outlet />
      </main>
      <Footer />
    </div>
  );
}
```

---

## Route Guards

### RequireAuth

```tsx
// components/RequireAuth.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';

interface RequireAuthProps {
  children: React.ReactNode;
  redirectTo?: string;
}

export function RequireAuth({ children, redirectTo = '/login' }: RequireAuthProps) {
  const { user, loading } = useAuth();
  const location = useLocation();
  
  if (loading) {
    return <Loading />;
  }
  
  if (!user) {
    // Перенаправить на login с сохранением текущего пути
    return <Navigate to={redirectTo} state={{ from: location }} replace />;
  }
  
  return <>{children}</>;
}

// Использование
<Route
  path="/profile"
  element={
    <RequireAuth>
      <ProfilePage />
    </RequireAuth>
  }
/>
```

### RequireRole

```tsx
interface RequireRoleProps {
  children: React.ReactNode;
  roles: string[];
  redirectTo?: string;
}

export function RequireRole({ children, roles, redirectTo = '/unauthorized' }: RequireRoleProps) {
  const { user } = useAuth();
  
  if (!user || !roles.includes(user.role)) {
    return <Navigate to={redirectTo} replace />;
  }
  
  return <>{children}</>;
}

// Использование
<Route
  path="/admin"
  element={
    <RequireRole roles={['admin', 'moderator']}>
      <AdminPage />
    </RequireRole>
  }
/>
```

---

## Loader и Action (React Router v6.4+)

### Loader

```tsx
// router.tsx
import { createBrowserRouter } from 'react-router-dom';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <HomePage />,
  },
  {
    path: '/users',
    element: <UsersPage />,
    loader: async () => {
      const users = await api.getUsers();
      return { users };
    },
  },
  {
    path: '/users/:id',
    element: <UserPage />,
    loader: async ({ params }) => {
      const user = await api.getUser(params.id!);
      return { user };
    },
  },
]);

// main.tsx
import { RouterProvider } from 'react-router-dom';
import { router } from './router';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <RouterProvider router={router} />
);

// Компонент с loader
import { useLoaderData } from 'react-router-dom';

function UsersPage() {
  const { users } = useLoaderData<typeof loader>();
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

### Action

```tsx
// router.tsx
export const router = createBrowserRouter([
  {
    path: '/users/new',
    element: <CreateUserPage />,
    action: async ({ request }) => {
      const formData = await request.formData();
      const user = await api.createUser({
        name: formData.get('name') as string,
        email: formData.get('email') as string,
      });
      return { user };
    },
  },
]);

// Компонент с action
import { Form, useActionData, useNavigation } from 'react-router-dom';

function CreateUserPage() {
  const actionData = useActionData<typeof action>();
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';
  
  return (
    <Form method="post">
      <input name="name" type="text" />
      <input name="email" type="email" />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create'}
      </button>
      
      {actionData?.error && <p className="error">{actionData.error}</p>}
    </Form>
  );
}
```

---

## Lazy Loading

```tsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const HomePage = lazy(() => import('./pages/HomePage'));
const AboutPage = lazy(() => import('./pages/AboutPage'));
const UsersPage = lazy(() => import('./pages/UsersPage'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/about" element={<AboutPage />} />
        <Route path="/users" element={<UsersPage />} />
      </Routes>
    </Suspense>
  );
}
```

---

## Программная навигация

### useNavigate

```tsx
import { useNavigate } from 'react-router-dom';

function LoginForm() {
  const navigate = useNavigate();
  
  const handleSubmit = async (credentials: Credentials) => {
    try {
      await api.login(credentials);
      navigate('/dashboard'); // Простая навигация
    } catch (error) {
      // Handle error
    }
  };
  
  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
}

// Навигация с состоянием
navigate('/dashboard', { 
  state: { from: 'login', message: 'Welcome!' } 
});

// Чтение состояния
import { useLocation } from 'react-router-dom';

function Dashboard() {
  const location = useLocation();
  const message = location.state?.message;
  
  return <div>{message}</div>;
}
```

---

## Хуки

### useParams

```tsx
import { useParams } from 'react-router-dom';

function UserPage() {
  const { id } = useParams<'id'>();
  // id: string | undefined
  
  return <div>User: {id}</div>;
}
```

### useSearchParams

```tsx
import { useSearchParams } from 'react-router-dom';

function SearchPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const query = searchParams.get('q');
  
  const updateQuery = (newQuery: string) => {
    setSearchParams({ q: newQuery });
  };
  
  return <div>{/* ... */}</div>;
}
```

### useLocation

```tsx
import { useLocation } from 'react-router-dom';

function Page() {
  const location = useLocation();
  
  // Текущий путь
  console.log(location.pathname); // "/users/123"
  
  // Query строка
  console.log(location.search); // "?page=2"
  
  // Состояние
  console.log(location.state); // { from: 'login' }
  
  return <div>{/* ... */}</div>;
}
```

### useNavigation

```tsx
import { useNavigation } from 'react-router-dom';

function SubmitButton() {
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';
  
  return (
    <button type="submit" disabled={isSubmitting}>
      {isSubmitting ? 'Submitting...' : 'Submit'}
    </button>
  );
}
```

### useRouteError

```tsx
// error-page.tsx
import { useRouteError, isRouteErrorResponse } from 'react-router-dom';

export function ErrorPage() {
  const error = useRouteError();
  
  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>{error.status} {error.statusText}</h1>
        <p>{error.data}</p>
      </div>
    );
  }
  
  return (
    <div>
      <h1>Something went wrong</h1>
      <p>{error instanceof Error ? error.message : 'Unknown error'}</p>
    </div>
  );
}

// router.tsx
export const router = createBrowserRouter([
  {
    path: '/',
    element: <HomePage />,
    errorElement: <ErrorPage />,
  },
]);
```

---

## Best Practices

### ✅ Делать

```tsx
// 1. Используйте абсолютные пути
<Link to="/users/123">User</Link>

// 2. Типизируйте параметры
function UserPage() {
  const { id } = useParams<'id'>();
  // ...
}

// 3. Обрабатывайте 404
<Routes>
  {/* ... */}
  <Route path="*" element={<NotFoundPage />} />
</Routes>

// 4. Используйте NavLink для активных ссылок
<NavLink
  to="/users"
  className={({ isActive }) => isActive ? 'nav-link active' : 'nav-link'}
/>

// 5. Loader для данных
{
  path: '/users',
  loader: () => api.getUsers(),
  element: <UsersPage />,
}
```

### ❌ Не делать

```tsx
// 1. Не используйте history.push
history.push('/users'); // ❌
navigate('/users'); // ✅

// 2. Не храните состояние маршрута в state
const [currentPage, setCurrentPage] = useState('/'); // ❌
// Используйте маршрутизатор ✅

// 3. Не забывайте про key для списков
{users.map(user => <Link to={`/users/${user.id}`}>{user.name}</Link>)}
// ✅ Ключ не нужен для Link, но нужен для map
{users.map(user => (
  <li key={user.id}><Link to={`/users/${user.id}`}>{user.name}</Link></li>
))}
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-Basics]] — основы React
- [[React-Data-Fetching]] — TanStack Query
- [[React-Hooks]] — кастомные хуки

---

> [!TIP] Совет
>
> 1. **Используйте React Router v6+** — современный API
> 2. **Loader для данных** — загрузка до рендера
> 3. **Route guards для защиты** — RequireAuth компонент
> 4. **Lazy loading для больших страниц** — Suspense + lazy
> 5. **useNavigate для навигации** — вместо history
