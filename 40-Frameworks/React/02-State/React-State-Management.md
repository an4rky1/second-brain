---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - state
  - zustand
  - redux
aliases:
  - React State Management
  - Zustand Redux Context
related:
  - React-MOC
  - React-Basics
---

# React — Управление состоянием

> [!SUMMARY] Обзор
> Управление состоянием в React: Context API, Zustand, Redux Toolkit. Когда и что использовать.

---

## Context API

### Базовый Context

```tsx
import { createContext, useContext, useState, useCallback } from 'react';

// Типы
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

// Создание контекста
const ThemeContext = createContext<ThemeContextType | null>(null);

// Provider
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  const toggleTheme = useCallback(() => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  }, []);
  
  const value = { theme, toggleTheme };
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// Хук для использования
function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

// Потребление
function ThemedButton() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button onClick={toggleTheme}>
      Current: {theme}
    </button>
  );
}

// Использование в App
function App() {
  return (
    <ThemeProvider>
      <ThemedButton />
    </ThemeProvider>
  );
}
```

### Multiple Contexts

```tsx
// Контексты
const AuthContext = createContext<AuthContextType | null>(null);
const ThemeContext = createContext<ThemeContextType | null>(null);
const ConfigContext = createContext<ConfigContextType | null>(null);

// Композитный Provider
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <ConfigProvider>
          {children}
        </ConfigProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

// Custom hook для всех контекстов
function useAppContext() {
  return {
    auth: useAuth(),
    theme: useTheme(),
    config: useConfig(),
  };
}
```

### Context + useReducer

```tsx
// Типы
interface State {
  user: User | null;
  loading: boolean;
  error: string | null;
}

type Action =
  | { type: 'LOGIN_START' }
  | { type: 'LOGIN_SUCCESS'; payload: User }
  | { type: 'LOGIN_ERROR'; payload: string }
  | { type: 'LOGOUT' };

// Редьюсер
function authReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'LOGIN_START':
      return { ...state, loading: true, error: null };
    case 'LOGIN_SUCCESS':
      return { ...state, loading: false, user: action.payload };
    case 'LOGIN_ERROR':
      return { ...state, loading: false, error: action.payload };
    case 'LOGOUT':
      return { ...state, user: null, loading: false, error: null };
    default:
      return state;
  }
}

// Context
interface AuthContextType {
  state: State;
  dispatch: React.Dispatch<Action>;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

// Provider
function AuthProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(authReducer, {
    user: null,
    loading: false,
    error: null,
  });
  
  const login = useCallback(async (credentials: Credentials) => {
    dispatch({ type: 'LOGIN_START' });
    try {
      const user = await api.login(credentials);
      dispatch({ type: 'LOGIN_SUCCESS', payload: user });
    } catch (error) {
      dispatch({ type: 'LOGIN_ERROR', payload: error.message });
    }
  }, []);
  
  const logout = useCallback(() => {
    dispatch({ type: 'LOGOUT' });
  }, []);
  
  return (
    <AuthContext.Provider value={{ state, dispatch, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Хук
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}

// Использование
function LoginForm() {
  const { login, state } = useAuth();
  
  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    await login({
      email: formData.get('email') as string,
      password: formData.get('password') as string,
    });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit" disabled={state.loading}>
        {state.loading ? 'Loading...' : 'Login'}
      </button>
      {state.error && <p>{state.error}</p>}
    </form>
  );
}
```

---

## Zustand

### Базовое использование

```bash
npm install zustand
```

```tsx
import create from 'zustand';

// Типы
interface Store {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

// Создание store
const useStore = create<Store>(set => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),
  decrement: () => set(state => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Использование в компоненте
function Counter() {
  const { count, increment, decrement } = useStore();
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}
```

### Селекторы (избегаем лишних ререндеров)

```tsx
interface Store {
  users: User[];
  selectedUserId: string | null;
  addUser: (user: User) => void;
  setSelectedUser: (id: string | null) => void;
}

const useStore = create<Store>(set => ({
  users: [],
  selectedUserId: null,
  addUser: (user) => set(state => ({ users: [...state.users, user] })),
  setSelectedUser: (id) => set({ selectedUserId: id }),
}));

// Плохо — ререндер при любом изменении
function UserList() {
  const { users, setSelectedUser } = useStore();
  return users.map(user => (
    <div key={user.id} onClick={() => setSelectedUser(user.id)}>
      {user.name}
    </div>
  ));
}

// Хорошо — ререндер только при изменении users
function UserList() {
  const users = useStore(state => state.users);
  const setSelectedUser = useStore(state => state.setSelectedUser);
  // ...
}

// Селектор с параметрами
function UserName({ userId }: { userId: string }) {
  const user = useStore(state => 
    state.users.find(u => u.id === userId)
  );
  return <span>{user?.name}</span>;
}
```

### Async Actions

```tsx
interface Store {
  users: User[];
  loading: boolean;
  error: string | null;
  fetchUsers: () => Promise<void>;
  createUser: (data: CreateUserDto) => Promise<void>;
}

const useStore = create<Store>((set, get) => ({
  users: [],
  loading: false,
  error: null,
  
  fetchUsers: async () => {
    set({ loading: true, error: null });
    try {
      const users = await api.getUsers();
      set({ users, loading: false });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  },
  
  createUser: async (data) => {
    const newUser = await api.createUser(data);
    set(state => ({ users: [...state.users, newUser] }));
  },
}));

// Использование
function UsersPage() {
  const { users, loading, error, fetchUsers } = useStore();
  
  useEffect(() => {
    fetchUsers();
  }, [fetchUsers]);
  
  if (loading) return <Loading />;
  if (error) return <Error message={error} />;
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

### Persist Middleware

```tsx
import create from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface Store {
  token: string | null;
  setToken: (token: string | null) => void;
  clearAuth: () => void;
}

const useAuthStore = create<Store>()(
  persist(
    set => ({
      token: null,
      setToken: (token) => set({ token }),
      clearAuth: () => set({ token: null }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ token: state.token }), // Только token сохраняется
    }
  )
);
```

### Devtools Middleware

```tsx
import create from 'zustand';
import { devtools } from 'zustand/middleware';

interface Store {
  count: number;
  increment: () => void;
}

const useStore = create<Store>()(
  devtools(
    set => ({
      count: 0,
      increment: () => set({ count: state.count + 1 }),
    }),
    { name: 'CounterStore' }
  )
);
```

### Combine Stores

```tsx
// Auth store
const useAuthStore = create<AuthState>()(/* ... */);

// Users store
const useUsersStore = create<UsersState>()(/* ... */);

// Композитный хук
function useAppStore() {
  return {
    auth: useAuthStore(),
    users: useUsersStore(),
  };
}

// Использование
function Component() {
  const { auth, users } = useAppStore();
  const { user } = auth;
  const { userList } = users;
  // ...
}
```

---

## Redux Toolkit

### Базовая настройка

```bash
npm install @reduxjs/toolkit react-redux
```

```tsx
// store.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';
import usersReducer from './usersSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    users: usersReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// app/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;

// app/provider.tsx
import { Provider } from 'react-redux';
import { store } from './store';

export function AppProvider({ children }: { children: React.ReactNode }) {
  return <Provider store={store}>{children}</Provider>;
}
```

### Slice

```tsx
// features/auth/authSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  loading: boolean;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  token: null,
  loading: false,
  error: null,
};

// Async thunk
export const login = createAsyncThunk(
  'auth/login',
  async (credentials: { email: string; password: string }) => {
    const response = await api.login(credentials);
    return response.user;
  }
);

// Slice
const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    logout(state) {
      state.user = null;
      state.token = null;
    },
    setError(state, action: PayloadAction<string>) {
      state.error = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(login.fulfilled, (state, action) => {
        state.loading = false;
        state.user = action.payload;
      })
      .addCase(login.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message ?? 'Login failed';
      });
  },
});

export const { logout, setError } = authSlice.actions;
export default authSlice.reducer;
```

### Использование в компоненте

```tsx
import { useAppDispatch, useAppSelector } from '@/app/hooks';
import { login, logout } from '@/features/auth/authSlice';

function LoginForm() {
  const dispatch = useAppDispatch();
  const { user, loading, error } = useAppSelector(state => state.auth);
  
  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    await dispatch(login({
      email: formData.get('email') as string,
      password: formData.get('password') as string,
    }));
  };
  
  if (user) {
    return (
      <div>
        Welcome, {user.name}!
        <button onClick={() => dispatch(logout())}>Logout</button>
      </div>
    );
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit" disabled={loading}>
        {loading ? 'Loading...' : 'Login'}
      </button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

---

## Сравнение подходов

| Подход | Когда использовать | Плюсы | Минусы |
|--------|-------------------|-------|--------|
| **useState** | Локальное состояние компонента | Просто, нет бойлерплейта | Не для общего состояния |
| **useReducer** | Сложная логика компонента | Предсказуемость, тестирование | Больше кода |
| **Context** | Глобальное состояние (тема, auth) | Встроенный, нет зависимостей | Ререндеры всех потребителей |
| **Zustand** | Средние/большие приложения | Просто, мало бойлерплейта, нет ререндеров | Дополнительная зависимость |
| **Redux Toolkit** | Корпоративные приложения | Devtools, middleware, экосистема | Много бойлерплейта |

---

## Best Practices

### ✅ Делать

```tsx
// 1. Разделяйте store по доменам
const useAuthStore = create(/* ... */);
const useUsersStore = create(/* ... */);
const useSettingsStore = create(/* ... */);

// 2. Используйте селекторы
const user = useStore(state => state.users.find(u => u.id === id));

// 3. Выносите логику в actions
const useStore = create(set => ({
  // ❌
  updateUser: async (id, data) => {
    const user = await api.updateUser(id, data);
    set(state => ({
      users: state.users.map(u => u.id === id ? user : u)
    }));
  },
  
  // ✅
  updateUser: async (id, data) => {
    const user = await api.updateUser(id, data);
    set(state => ({ users: updateItem(state.users, user) }));
  },
}));

// 4. Используйте persist для важных данных
const useAuthStore = create(persist(/* ... */));

// 5. Типизируйте всё
interface Store { /* ... */ }
const useStore = create<Store>(/* ... */);
```

### ❌ Не делать

```tsx
// 1. Не храните в store всё
const useStore = create(() => ({
  users: [],           // ✅
  formData: {},        // ❌ — локальное состояние
  api: new Api(),      // ❌ — создавайте вне store
}));

// 2. Не мутируйте state напрямую
set(state => {
  state.users.push(newUser); // ❌
  return { users: [...state.users, newUser] }; // ✅
});

// 3. Не создавайте store в компоненте
function Component() {
  const useStore = create(/* ... */); // ❌
  // ...
}

// ✅ Создавайте store вне компонентов
const useStore = create(/* ... */);
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
> 1. **Начинайте с useState** — не усложняйте без нужды
> 2. **Zustand для большинства случаев** — просто и эффективно
> 3. **Redux для enterprise** — когда нужна экосистема
> 4. **Context для редко меняющихся данных** — тема, локаль
> 5. **Селекторы для оптимизации** — избегайте лишних ререндеров
