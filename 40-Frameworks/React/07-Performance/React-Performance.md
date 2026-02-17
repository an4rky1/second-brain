---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - performance
  - optimization
aliases:
  - React Performance
  - React Optimization
related:
  - React-MOC
  - React-Basics
---

# React — Performance и оптимизация

> [!SUMMARY] Обзор
> Оптимизация производительности React приложений: мемоизация, ленивая загрузка, виртуализация, code splitting.

---

## Мемоизация

### React.memo

```tsx
import { memo } from 'react';

// Базовое использование
const ExpensiveComponent = memo(({ data, onClick }) => {
  return (
    <div>
      {data.map(item => <Item key={item.id} item={item} />)}
      <button onClick={onClick}>Click</button>
    </div>
  );
});

// С кастомным сравнением
const MemoComponent = memo(Component, (prevProps, nextProps) => {
  // Возвращаем true если props не изменились
  return prevProps.id === nextProps.id && 
         prevProps.count === nextProps.count;
});

// Использование
<MemoComponent id={1} count={count} />
```

### useMemo

```tsx
import { useMemo } from 'react';

// Дорогие вычисления
function UserList({ users, filter }) {
  const filteredUsers = useMemo(() => {
    console.log('Filtering users...');
    return users.filter(user => 
      user.name.toLowerCase().includes(filter.toLowerCase())
    );
  }, [users, filter]);
  
  return (
    <ul>
      {filteredUsers.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}

// Сортировка
function SortedList({ items }) {
  const sorted = useMemo(() => {
    return [...items].sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);
  
  return <div>{/* ... */}</div>;
}

// Агрегация
function Cart({ items }) {
  const total = useMemo(() => {
    return items.reduce((sum, item) => sum + item.price, 0);
  }, [items]);
  
  return <div>Total: ${total}</div>;
}

// Создание объектов
function Component({ id }) {
  const config = useMemo(() => ({
    userId: id,
    timeout: 5000,
    retry: true,
  }), [id]);
  
  return <Child config={config} />;
}
```

### useCallback

```tsx
import { useCallback } from 'react';

// Для передачи в memoized компоненты
function Parent() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  
  return (
    <div>
      <p>{count}</p>
      <MemoizedButton onClick={handleClick} />
    </div>
  );
}

// С зависимостями
function Form({ onSubmit }) {
  const handleSubmit = useCallback((data) => {
    onSubmit(data);
  }, [onSubmit]);
  
  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
}

// Для useEffect
function Component({ userId }) {
  const fetchUser = useCallback(async () => {
    const user = await api.getUser(userId);
    setUser(user);
  }, [userId]);
  
  useEffect(() => {
    fetchUser();
  }, [fetchUser]);
}
```

### Когда использовать

```tsx
// ✅ Используйте useMemo/useCallback когда:
// 1. Дорогие вычисления
const result = useMemo(() => expensiveCalculation(data), [data]);

// 2. Передача функций в memoized компоненты
const handleClick = useCallback(() => {}, []);
<MemoizedComponent onClick={handleClick} />

// 3. Создание стабильных объектов
const config = useMemo(() => ({ timeout: 5000 }), []);

// ❌ Не используйте когда:
// 1. Дешёвые вычисления
const name = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// Просто: const name = `${firstName} ${lastName}`;

// 2. Примитивные значения
const count = useMemo(() => 0, []);
// Просто: const count = 0;
```

---

## Code Splitting

### React.lazy + Suspense

```tsx
import { lazy, Suspense } from 'react';

// Ленивая загрузка компонентов
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));
const Profile = lazy(() => import('./Profile'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Dashboard />
      <Settings />
      <Profile />
    </Suspense>
  );
}

// С кастомным fallback
<Suspense fallback={<div>Loading component...</div>}>
  <Dashboard />
</Suspense>

// С обработкой ошибок
function ErrorBoundary({ children }: { children: React.ReactNode }) {
  const [hasError, setHasError] = useState(false);
  
  if (hasError) return <ErrorPage />;
  
  return (
    <Suspense fallback={<Loading />}>
      <ErrorCatcher onError={() => setHasError(true)}>
        {children}
      </ErrorCatcher>
    </Suspense>
  );
}
```

### Route-based Splitting

```tsx
import { lazy } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const HomePage = lazy(() => import('./pages/HomePage'));
const UsersPage = lazy(() => import('./pages/UsersPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoading />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/users" element={<UsersPage />} />
          <Route path="/settings" element={<SettingsPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### Preloading

```tsx
// Preload при наведении
function preloadComponent(modulePath: string) {
  import(modulePath);
}

function Navigation() {
  return (
    <nav>
      <Link
        to="/dashboard"
        onMouseEnter={() => preloadComponent('./Dashboard')}
      >
        Dashboard
      </Link>
    </nav>
  );
}

// Preload с React Query
function usePrefetchQuery(queryKey: string[], queryFn: () => Promise<any>) {
  const queryClient = useQueryClient();
  
  const prefetch = useCallback(() => {
    queryClient.prefetchQuery({
      queryKey,
      queryFn,
      staleTime: 1000 * 60 * 5,
    });
  }, [queryKey, queryFn, queryClient]);
  
  return prefetch;
}

function UserList() {
  const prefetchUser = usePrefetchQuery(
    ['users', userId],
    () => api.getUser(userId)
  );
  
  return (
    <div onMouseEnter={prefetch}>
      <Link to={`/users/${userId}`}>{user.name}</Link>
    </div>
  );
}
```

---

## Виртуализация списков

### React Window

```bash
npm install react-window
```

```tsx
import { FixedSizeList, FixedSizeGrid } from 'react-window';
import AutoSizer from 'react-virtualized-auto-sizer';

// Вертикальный список
function VirtualList({ items }: { items: Item[] }) {
  return (
    <AutoSizer>
      {({ height, width }) => (
        <FixedSizeList
          height={height}
          width={width}
          itemCount={items.length}
          itemSize={50} // Высота элемента в пикселях
          itemData={items}
        >
          {({ index, style }) => (
            <div style={style}>
              {items[index].name}
            </div>
          )}
        </FixedSizeList>
      )}
    </AutoSizer>
  );
}

// Горизонтальный список
<FixedSizeList
  height={100}
  width={600}
  itemCount={items.length}
  itemSize={200} // Ширина элемента
  layout="horizontal"
>
  {({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  )}
</FixedSizeList>

// Grid (двумерный)
<FixedSizeGrid
  columnCount={100}
  columnWidth={100}
  height={600}
  rowCount={1000}
  rowHeight={50}
  width={800}
>
  {({ columnIndex, rowIndex, style }) => (
    <div style={style}>
      Cell {rowIndex}, {columnIndex}
    </div>
  )}
</FixedSizeGrid>

// Variable size
<VariableSizeList
  height={600}
  itemCount={items.length}
  itemSize={(index) => items[index].height}
  width={'100%'}
>
  {({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  )}
</VariableSizeList>
```

### TanStack Virtual

```bash
npm install @tanstack/react-virtual
```

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });
  
  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Оптимизация рендеринга

### Избегаем лишних ререндеров

```tsx
// ❌ Плохо — новый объект каждый рендер
function Component() {
  return <Child style={{ color: 'red' }} />;
}

// ✅ Хорошо — стабильный объект
function Component() {
  const style = useMemo(() => ({ color: 'red' }), []);
  return <Child style={style} />;
}

// ❌ Плохо — новая функция каждый рендер
function Component() {
  const handleClick = () => console.log('click');
  return <Button onClick={handleClick} />;
}

// ✅ Хорошо — стабильная функция
function Component() {
  const handleClick = useCallback(() => console.log('click'), []);
  return <Button onClick={handleClick} />;
}

// ❌ Плохо — inline arrow функция
{items.map(item => (
  <Item key={item.id} onClick={() => handleItemClick(item.id)} />
))}

// ✅ Хорошо — вынесенная функция
const handleItemClick = useCallback((id: number) => {
  // ...
}, []);

{items.map(item => (
  <Item key={item.id} onClick={() => handleItemClick(item.id)} />
))}
```

### Key Prop

```tsx
// ❌ Плохо — индекс как key
{items.map((item, index) => (
  <Item key={index} item={item} />
))}

// ✅ Хорошо — уникальный ID
{items.map(item => (
  <Item key={item.id} item={item} />
))}

// ✅ Хорошо — составной ключ
{items.map(item => (
  <Item key={`${item.category}-${item.id}`} item={item} />
))}
```

### Компоненты высшего порядка

```tsx
// Оптимизация с loading состоянием
function withLoading<P extends object>(
  Component: React.ComponentType<P>
) {
  return function WithLoadingComponent(
    props: P & { isLoading: boolean; loadingComponent?: React.ReactNode }
  ) {
    const { isLoading, loadingComponent, ...rest } = props;
    
    if (isLoading) {
      return loadingComponent || <Loading />;
    }
    
    return <Component {...(rest as P)} />;
  };
}

// Использование
const UserListWithLoading = withLoading(UserList);
<UserListWithLoading isLoading={loading} users={users} />;
```

---

## Profiling

### React DevTools Profiler

```tsx
import { Profiler } from 'react';

function onRenderCallback(
  id: string,
  phase: 'mount' | 'update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number
) {
  console.log(`${id} took ${actualDuration}ms to render`);
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Dashboard />
    </Profiler>
  );
}
```

### why-did-you-render

```bash
npm install @welldone-software/why-did-you-render
```

```tsx
// main.tsx
import whyDidYouRender from '@welldone-software/why-did-you-render';
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}
```

### Performance Monitor

```tsx
// hooks/usePerformance.ts
import { useEffect } from 'react';

export function usePerformance(componentName: string) {
  useEffect(() => {
    const startTime = performance.now();
    
    return () => {
      const endTime = performance.now();
      console.log(`${componentName} rendered in ${endTime - startTime}ms`);
    };
  }, [componentName]);
}

// Использование
function ExpensiveComponent() {
  usePerformance('ExpensiveComponent');
  return <div>{/* ... */}</div>;
}
```

---

## Best Practices

### ✅ Делать

```tsx
// 1. Используйте React.memo для тяжёлых компонентов
const ExpensiveList = memo(({ items }) => {
  return items.map(item => <Item key={item.id} item={item} />);
});

// 2. useMemo для дорогих вычислений
const sorted = useMemo(() => {
  return [...items].sort((a, b) => a.name.localeCompare(b.name));
}, [items]);

// 3. useCallback для функций
const handleClick = useCallback((id: number) => {
  // ...
}, []);

// 4. Code splitting для больших компонентов
const Dashboard = lazy(() => import('./Dashboard'));

// 5. Виртуализация для длинных списков
<FixedSizeList itemCount={10000} itemSize={50}>
  {({ index, style }) => <Item style={style} />}
</FixedSizeList>

// 6. Избегайте inline объектов
const config = useMemo(() => ({ timeout: 5000 }), []);
<Child config={config} />
```

### ❌ Не делать

```tsx
// 1. Не мемоизируйте всё подряд
const name = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// Просто: const name = `${firstName} ${lastName}`;

// 2. Не используйте индекс как key для изменяемых списков
items.map((item, i) => <Item key={i} />); // ❌

// 3. Не создавайте объекты в render
<Child style={{ color: 'red' }} /> // ❌

// 4. Не забывайте зависимости useCallback
const handleClick = useCallback(() => {
  console.log(id); // id может устареть!
}, []); // ❌

const handleClick = useCallback(() => {
  console.log(id);
}, [id]); // ✅
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-Basics]] — основы React
- [[React-Data-Fetching]] — TanStack Query
- [[React-Ecosystem]] — библиотеки

---

> [!TIP] Совет
>
> 1. **Профилируйте перед оптимизацией** — найдите узкие места
> 2. **memo/useMemo для дорогих операций** — не для всего
> 3. **Виртуализация для списков > 100 элементов** — react-window
> 4. **Code splitting для больших страниц** — lazy + Suspense
> 5. **Избегайте inline объектов/функций** — стабильные ссылки
