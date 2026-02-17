---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - basics
  - hooks
aliases:
  - React Basics
  - React Основы
related:
  - React-MOC
  - React-Hooks
  - TypeScript-MOC
---

# React — Основы и хуки

> [!SUMMARY] Обзор
> Основы React: компоненты, хуки, паттерны, жизненный цикл. Фундамент для разработки на React.

---

## Быстрый старт

```bash
# Vite (рекомендуется)
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev

# Тесты
npm install -D @testing-library/react @testing-library/jest-dom vitest

# Запуск
npm run dev      # Development
npm run build    # Production build
npm run preview  # Preview production
npm run test     # Тесты
```

### Структура проекта

```
src/
├── components/        # Переиспользуемые компоненты
│   ├── Button/
│   │   ├── Button.tsx
│   │   └── index.ts
│   └── common/
├── features/          # Фичи (feature-sliced)
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── api/
│   └── users/
├── pages/             # Страницы
├── layouts/           # Лейауты
├── hooks/             # Общие хуки
├── context/           # Контексты
├── lib/               # Утилиты
├── types/             # TypeScript типы
├── assets/            # Статика
├── App.tsx
└── main.tsx
```

---

## Компоненты

### Functional Component

```tsx
import React from 'react';

interface Props {
  title: string;
  count?: number;
  onClick: () => void;
}

const Card: React.FC<Props> = ({ title, count = 0, onClick }) => {
  return (
    <div onClick={onClick}>
      <h2>{title}</h2>
      <p>Count: {count}</p>
    </div>
  );
};

// Без React.FC (современный стиль)
interface CardProps {
  title: string;
  count?: number;
  onClick: () => void;
}

export function Card({ title, count = 0, onClick }: CardProps) {
  return (
    <div onClick={onClick}>
      <h2>{title}</h2>
      <p>Count: {count}</p>
    </div>
  );
}
```

### Children

```tsx
interface ContainerProps {
  children: React.ReactNode;
  className?: string;
}

export function Container({ children, className = '' }: ContainerProps) {
  return <div className={`container ${className}`}>{children}</div>;
}

// Использование
<Container className="main">
  <h1>Hello</h1>
  <p>Content</p>
</Container>
```

### Render Props

```tsx
interface DataProviderProps {
  render: (data: Data | null) => React.ReactNode;
}

export function DataProvider({ render }: DataProviderProps) {
  const [data, setData] = useState<Data | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <Loading />;
  return <>{render(data)}</>;
}

// Использование
<DataProvider render={(data) => (
  <div>{data?.title}</div>
)} />
```

### Compound Components

```tsx
interface MenuProps {
  children: React.ReactNode;
}

function Menu({ children }: MenuProps) {
  return <ul className="menu">{children}</ul>;
}

interface MenuItemProps {
  children: React.ReactNode;
  onClick?: () => void;
}

function MenuItem({ children, onClick }: MenuItemProps) {
  return <li className="menu-item" onClick={onClick}>{children}</li>;
}

Menu.Item = MenuItem;

// Использование
<Menu>
  <MenuItem onClick={() => console.log('Click 1')}>Item 1</MenuItem>
  <MenuItem onClick={() => console.log('Click 2')}>Item 2</MenuItem>
</Menu>
```

---

## Хуки (Hooks)

### useState

```tsx
// Базовое использование
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);

// Функциональное обновление
setCount(prev => prev + 1);

// Массив
const [items, setItems] = useState<Item[]>([]);
setItems(prev => [...prev, newItem]);
setItems(prev => prev.filter(item => item.id !== id));

// Объект
const [form, setForm] = useState({ email: '', password: '' });
setForm(prev => ({ ...prev, email: 'new@email.com' }));
```

### useEffect

```tsx
// Mount + Update
useEffect(() => {
  console.log('Effect ran');
  return () => {
    console.log('Cleanup');
  };
}, [dependency]);

// Только mount
useEffect(() => {
  console.log('Mount only');
}, []);

// Только unmount
useEffect(() => {
  return () => {
    console.log('Unmount only');
  };
}, []);

// Async effect
useEffect(() => {
  let cancelled = false;
  
  const fetchData = async () => {
    const result = await fetch('/api/data');
    if (!cancelled) setData(result);
  };
  
  fetchData();
  return () => { cancelled = true; };
}, []);
```

### useMemo

```tsx
// Дорогие вычисления
const filtered = useMemo(() => {
  return items.filter(item => item.active);
}, [items]);

// Сортировка
const sorted = useMemo(() => {
  return [...items].sort((a, b) => a.name.localeCompare(b.name));
}, [items]);

// Агрегация
const total = useMemo(() => {
  return items.reduce((sum, item) => sum + item.price, 0);
}, [items]);
```

### useCallback

```tsx
// Для передачи в дочерние компоненты
const handleClick = useCallback((id: number) => {
  console.log('Clicked:', id);
}, []);

// С зависимостями
const handleSubmit = useCallback((data: FormData) => {
  api.submit(data);
}, [api]);

// Для memoized компонентов
const MemoizedList = memo(List);

function Parent() {
  const renderItem = useCallback((item: Item) => (
    <div>{item.name}</div>
  ), []);
  
  return <MemoizedList items={items} renderItem={renderItem} />;
}
```

### useRef

```tsx
// DOM реф
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus();

// Хранение значения (не вызывает ререндер)
const countRef = useRef(0);
countRef.current += 1;

// Интервал
const intervalRef = useRef<number | null>(null);

useEffect(() => {
  intervalRef.current = setInterval(() => {
    console.log('Tick');
  }, 1000);
  
  return () => {
    if (intervalRef.current) clearInterval(intervalRef.current);
  };
}, []);
```

### useContext

```tsx
// Создание контекста
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

// Provider
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  const toggleTheme = useCallback(() => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  }, []);
  
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Потребление
function ThemedButton() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  
  const { theme, toggleTheme } = context;
  
  return (
    <button onClick={toggleTheme}>
      Current theme: {theme}
    </button>
  );
}
```

### useReducer

```tsx
// Типы
interface State {
  user: User | null;
  loading: boolean;
  error: string | null;
}

type Action =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: User }
  | { type: 'FETCH_ERROR'; payload: string };

// Редьюсер
function authReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS':
      return { ...state, loading: false, user: action.payload };
    case 'FETCH_ERROR':
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}

// Использование
const [state, dispatch] = useReducer(authReducer, {
  user: null,
  loading: false,
  error: null,
});

dispatch({ type: 'FETCH_START' });
dispatch({ type: 'FETCH_SUCCESS', payload: user });
```

---

## Паттерны

### Conditional Rendering

```tsx
// Логическое И
{isLoggedIn && <Dashboard />}

// Тернарный оператор
{users.length > 0 ? <UserList /> : <EmptyState />}

// Ранний return
function Component({ user }) {
  if (!user) return null;
  if (loading) return <Loading />;
  return <div>{user.name}</div>;
}
```

### List Rendering

```tsx
// С ключом
{users.map(user => (
  <UserCard key={user.id} user={user} />
))}

// С индексом (только если список статичен)
{items.map((item, index) => (
  <Item key={index} item={item} />
))}

// Фрагменты
{items.map(item => (
  <Fragment key={item.id}>
    <Item {...item} />
    <Divider />
  </Fragment>
))}
```

### Event Handlers

```tsx
// Типизированные события
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  e.preventDefault();
  console.log(e.currentTarget);
};

const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  console.log(e.target.value);
};

const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
};

// useCallback для handlers
const handleClick = useCallback((e: React.MouseEvent) => {
  // Logic
}, [dependency]);
```

### Controlled vs Uncontrolled

```tsx
// Controlled
function ControlledForm() {
  const [value, setValue] = useState('');
  
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// Uncontrolled
function UncontrolledForm() {
  const inputRef = useRef<HTMLInputElement>(null);
  
  const handleSubmit = () => {
    console.log(inputRef.current?.value);
  };
  
  return (
    <>
      <input ref={inputRef} defaultValue="" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

---

## Работа с API

### Базовый fetch хук

```tsx
function useApi<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    const fetchData = async () => {
      try {
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(res.statusText);
        setData(await res.json());
      } catch (err) {
        if (err instanceof Error && err.name !== 'AbortError') {
          setError(err);
        }
      } finally {
        setLoading(false);
      }
    };

    fetchData();
    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Использование
const { data: users, loading, error } = useApi<User[]>('/api/users');
```

### Optimistic Update

```tsx
const [todos, setTodos] = useState<Todo[]>([]);

const addTodo = async (text: string) => {
  const newTodo: Todo = { id: Date.now(), text, completed: false };
  
  // Optimistic update
  setTodos(prev => [...prev, newTodo]);
  
  try {
    await api.post('/todos', newTodo);
  } catch {
    // Rollback
    setTodos(prev => prev.filter(t => t.id !== newTodo.id));
  }
};

const toggleTodo = async (id: number) => {
  const todo = todos.find(t => t.id === id);
  if (!todo) return;
  
  // Optimistic update
  setTodos(prev => prev.map(t => 
    t.id === id ? { ...t, completed: !t.completed } : t
  ));
  
  try {
    await api.patch(`/todos/${id}`, { completed: !todo.completed });
  } catch {
    // Rollback
    setTodos(prev => prev.map(t => 
      t.id === id ? todo : t
    ));
  }
};
```

---

## Performance

### memo

```tsx
// Мемоизация компонента
const ExpensiveComponent = memo(({ data }) => {
  return <div>{/* ... */}</div>;
});

// С кастомным сравнением
const MemoComponent = memo(Component, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id;
});
```

### Code Splitting

```tsx
// Lazy loading
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Dashboard />
      <Settings />
    </Suspense>
  );
}

// Preloading
function preloadComponent(modulePath: string) {
  import(modulePath);
}

<button
  onMouseEnter={() => preloadComponent('./HeavyComponent')}
  onClick={() => setShowHeavy(true)}
>
  Load
</button>
```

### Error Boundary

```tsx
class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Error:', error, info);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// Использование
<ErrorBoundary fallback={<ErrorPage />}>
  <Dashboard />
</ErrorBoundary>
```

### Portals

```tsx
import { createPortal } from 'react-dom';

function Modal({ children, onClose }: { 
  children: React.ReactNode; 
  onClose: () => void;
}) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')!
  );
}
```

---

## Best Practices

### ✅ Делать

```tsx
// 1. Типизируйте пропсы
interface Props {
  title: string;
  onClick?: () => void;
}

// 2. Используйте деструктуризацию
const Component = ({ title, onClick }: Props) => {};

// 3. Ранний return
if (!user) return null;
if (loading) return <Loading />;

// 4. Разделяйте компоненты
const UserPage = () => (
  <Layout>
    <UserHeader />
    <UserContent />
  </Layout>
);

// 5. Кастомные хуки для логики
function useUsers() {
  const [users, setUsers] = useState<User[]>([]);
  // Logic...
  return { users, loading, error };
}
```

### ❌ Не делать

```tsx
// 1. useEffect для вычислений
useEffect(() => {
  setTotal(items.reduce((a, b) => a + b.price, 0)); // ❌
}, [items]);

const total = useMemo(() => 
  items.reduce((a, b) => a + b.price, 0), [items]); // ✅

// 2. Мутация state
state.items.push(newItem); // ❌
setState({ items: [...state.items, newItem] }); // ✅

// 3. Индексы как key
items.map((item, i) => <Item key={i} />); // ❌
items.map(item => <Item key={item.id} />); // ✅

// 4. Объекты в render
<Component style={{ color: 'red' }} /> // ❌
const style = useMemo(() => ({ color: 'red' }), []); // ✅
```

---

## Частые ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Cannot read property of undefined` | Ранний рендер без данных | Проверка или initial state |
| `Too many re-renders` | setState в render | Переместите в useEffect |
| `Stale closure` | Устаревшие значения | Добавьте в зависимости |
| `Memory leak` | Нет cleanup в useEffect | Возвращайте cleanup функцию |
| `Key prop missing` | Нет key в map | Добавьте уникальный key |

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-Hooks]] — кастомные хуки
- [[React-State-Management]] — Context, Zustand
- [[React-Data-Fetching]] — TanStack Query
- [[TypeScript-MOC]] — типизация

---

> [!TIP] Совет
>
> 1. **Хуки > Классы** — используйте функциональные компоненты
> 2. **Поднимайте state** — храните на нужном уровне
> 3. **Композиция > Наследование** — children и render props
> 4. **Custom hooks для логики** — переиспользуйте
> 5. **React DevTools** — ваш друг для отладки
