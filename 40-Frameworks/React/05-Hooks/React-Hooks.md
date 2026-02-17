---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - hooks
  - custom-hooks
aliases:
  - React Custom Hooks
  - React Hooks Library
related:
  - React-MOC
  - React-Basics
---

# React — Кастомные хуки

> [!SUMMARY] Обзор
> Библиотека кастомных хуков для React: работа с состоянием, API, формами, персистентность, утилиты.

---

## State Hooks

### useLocalStorage

```tsx
import { useState, useEffect } from 'react';

function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  // Получаем значение из localStorage или используем initial
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  // Сохраняем в localStorage при изменении
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue];
}

// Использование
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');
  
  return (
    <button onClick={() => setTheme(prev => prev === 'light' ? 'dark' : 'light')}>
      {theme}
    </button>
  );
}
```

### useSessionStorage

```tsx
function useSessionStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.sessionStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.sessionStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue];
}
```

### useToggle

```tsx
function useToggle(initialValue = false): [boolean, () => void] {
  const [value, setValue] = useState(initialValue);
  
  const toggle = useCallback(() => setValue(prev => !prev), []);
  
  return [value, toggle];
}

// Расширенная версия
function useToggleWithCallback(
  initialValue = false,
  onToggle?: (value: boolean) => void
): [boolean, (value?: boolean) => void] {
  const [value, setValue] = useState(initialValue);
  
  const setToggle = useCallback((newValue?: boolean) => {
    const nextValue = newValue ?? !value;
    setValue(nextValue);
    onToggle?.(nextValue);
  }, [value, onToggle]);
  
  return [value, setToggle];
}

// Использование
const [isOpen, toggleOpen] = useToggle(false);
const [isChecked, setChecked] = useToggleWithCallback(false, (v) => console.log(v));
```

### useCounter

```tsx
interface UseCounterOptions {
  min?: number;
  max?: number;
  step?: number;
}

function useCounter(
  initialValue = 0,
  { min, max, step = 1 }: UseCounterOptions = {}
) {
  const [count, setCount] = useState(initialValue);
  
  const increment = useCallback(() => {
    setCount(prev => {
      const next = prev + step;
      return max !== undefined ? Math.min(next, max) : next;
    });
  }, [step, max]);
  
  const decrement = useCallback(() => {
    setCount(prev => {
      const next = prev - step;
      return min !== undefined ? Math.max(next, min) : next;
    });
  }, [step, min]);
  
  const reset = useCallback(() => setCount(initialValue), [initialValue]);
  
  const set = useCallback((value: number) => {
    setCount(prev => {
      if (min !== undefined && value < min) return min;
      if (max !== undefined && value > max) return max;
      return value;
    });
  }, [min, max]);
  
  return { count, increment, decrement, reset, set };
}

// Использование
const { count, increment, decrement, reset } = useCounter(0, { min: 0, max: 10 });
```

### useMap

```tsx
function useMap<K, V>(initialValue?: Iterable<readonly [K, V]>) {
  const [map, setMap] = useState(() => new Map(initialValue));
  
  const set = useCallback((key: K, value: V) => {
    setMap(prev => {
      const next = new Map(prev);
      next.set(key, value);
      return next;
    });
  }, []);
  
  const remove = useCallback((key: K) => {
    setMap(prev => {
      const next = new Map(prev);
      next.delete(key);
      return next;
    });
  }, []);
  
  const clear = useCallback(() => {
    setMap(new Map());
  }, []);
  
  return { map, set, remove, clear };
}

// Использование
const { map, set, remove, clear } = useMap<string, number>([['a', 1]]);
```

### useSet

```tsx
function useSet<T>(initialValue?: Iterable<T>) {
  const [set, setSet] = useState(() => new Set(initialValue));
  
  const add = useCallback((value: T) => {
    setSet(prev => {
      const next = new Set(prev);
      next.add(value);
      return next;
    });
  }, []);
  
  const remove = useCallback((value: T) => {
    setSet(prev => {
      const next = new Set(prev);
      next.delete(value);
      return next;
    });
  }, []);
  
  const clear = useCallback(() => {
    setSet(new Set());
  }, []);
  
  return { set, add, remove, clear };
}
```

---

## Event Hooks

### useEventListener

```tsx
import { useEffect, useRef } from 'react';

function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: Window = window
): void;

function useEventListener<K extends keyof HTMLElementEventMap>(
  eventName: K,
  handler: (event: HTMLElementEventMap[K]) => void,
  element: HTMLElement | null
): void;

function useEventListener(
  eventName: string,
  handler: (event: Event) => void,
  element: EventTarget = window
) {
  const savedHandler = useRef(handler);
  
  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);
  
  useEffect(() => {
    const target = element;
    if (!target?.addEventListener) return;
    
    const listener = (event: Event) => savedHandler.current(event);
    target.addEventListener(eventName, listener);
    
    return () => {
      target.removeEventListener(eventName, listener);
    };
  }, [eventName, element]);
}

// Использование
function WindowTracker() {
  const [windowSize, setWindowSize] = useState({ width: 0, height: 0 });
  
  useEventListener('resize', () => {
    setWindowSize({ width: window.innerWidth, height: window.innerHeight });
  });
  
  return <div>{windowSize.width}x{windowSize.height}</div>;
}
```

### useClickOutside

```tsx
function useClickOutside<T extends HTMLElement>(
  handler: (event: MouseEvent) => void
) {
  const ref = useRef<T>(null);
  
  useEventListener('mousedown', (event) => {
    if (ref.current && !ref.current.contains(event.target as Node)) {
      handler(event);
    }
  });
  
  return ref;
}

// Использование
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const ref = useClickOutside<HTMLDivElement>(() => setIsOpen(false));
  
  return (
    <div ref={ref}>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <div className="dropdown">Content</div>}
    </div>
  );
}
```

### useKeyPress

```tsx
function useKeyPress(targetKey: string) {
  const [keyPressed, setKeyPressed] = useState(false);
  
  const downHandler = useCallback(
    ({ key }: KeyboardEvent) => {
      if (key === targetKey) setKeyPressed(true);
    },
    [targetKey]
  );
  
  const upHandler = useCallback(
    ({ key }: KeyboardEvent) => {
      if (key === targetKey) setKeyPressed(false);
    },
    [targetKey]
  );
  
  useEventListener('keydown', downHandler);
  useEventListener('keyup', upHandler);
  
  return keyPressed;
}

// Использование
const isEnterPressed = useKeyPress('Enter');
```

---

## API Hooks

### useFetch

```tsx
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string, options?: RequestInit): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  const fetchFn = useCallback(async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(url, options);
      if (!response.ok) throw new Error(response.statusText);
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [url, options]);
  
  useEffect(() => {
    fetchFn();
  }, [fetchFn]);
  
  return { data, loading, error, refetch: fetchFn };
}

// Использование
const { data, loading, error, refetch } = useFetch<User[]>('/api/users');
```

### useDebounce

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}

// Использование
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);
  
  const { data } = useSearch(debouncedQuery);
  
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

### useThrottle

```tsx
function useThrottle<T>(value: T, interval: number): T {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastUpdated = useRef<number | null>(null);
  
  useEffect(() => {
    const now = Date.now();
    
    if (!lastUpdated.current || now >= lastUpdated.current + interval) {
      lastUpdated.current = now;
      setThrottledValue(value);
    } else {
      const id = setTimeout(() => {
        lastUpdated.current = Date.now();
        setThrottledValue(value);
      }, interval - (now - lastUpdated.current));
      
      return () => clearTimeout(id);
    }
  }, [value, interval]);
  
  return throttledValue;
}
```

---

## UI Hooks

### useModal

```tsx
function useModal(initialState = false) {
  const [isOpen, setIsOpen] = useState(initialState);
  
  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen(prev => !prev), []);
  
  return { isOpen, open, close, toggle };
}

// Использование
function Page() {
  const modal = useModal();
  
  return (
    <>
      <button onClick={modal.open}>Open Modal</button>
      {modal.isOpen && <Modal onClose={modal.close}>Content</Modal>}
    </>
  );
}
```

### useTabs

```tsx
function useTabs<T extends string>(initialTab: T, tabs: T[]) {
  const [activeTab, setActiveTab] = useState<T>(initialTab);
  
  const activate = useCallback((tab: T) => {
    if (tabs.includes(tab)) {
      setActiveTab(tab);
    }
  }, [tabs]);
  
  const next = useCallback(() => {
    const currentIndex = tabs.indexOf(activeTab);
    const nextIndex = (currentIndex + 1) % tabs.length;
    setActiveTab(tabs[nextIndex]);
  }, [activeTab, tabs]);
  
  const prev = useCallback(() => {
    const currentIndex = tabs.indexOf(activeTab);
    const prevIndex = (currentIndex - 1 + tabs.length) % tabs.length;
    setActiveTab(tabs[prevIndex]);
  }, [activeTab, tabs]);
  
  return { activeTab, activate, next, prev };
}

// Использование
type Tab = 'overview' | 'settings' | 'billing';
const { activeTab, activate, next, prev } = useTabs<Tab>('overview', ['overview', 'settings', 'billing']);
```

### usePagination

```tsx
interface UsePaginationOptions {
  total: number;
  pageSize?: number;
  initialPage?: number;
}

function usePagination({ total, pageSize = 10, initialPage = 1 }: UsePaginationOptions) {
  const [page, setPage] = useState(initialPage);
  const totalPages = Math.ceil(total / pageSize);
  
  const goToPage = useCallback((newPage: number) => {
    setPage(Math.max(1, Math.min(newPage, totalPages)));
  }, [totalPages]);
  
  const nextPage = useCallback(() => {
    setPage(prev => Math.min(prev + 1, totalPages));
  }, [totalPages]);
  
  const prevPage = useCallback(() => {
    setPage(prev => Math.max(prev - 1, 1));
  }, []);
  
  const reset = useCallback(() => setPage(1), []);
  
  return {
    page,
    totalPages,
    hasNext: page < totalPages,
    hasPrev: page > 1,
    goToPage,
    nextPage,
    prevPage,
    reset,
  };
}

// Использование
const { page, totalPages, hasNext, nextPage, prevPage } = usePagination({ total: 100 });
```

---

## Form Hooks

### useForm

```tsx
interface UseFormOptions<T> {
  initialValues: T;
  onSubmit?: (values: T) => void | Promise<void>;
  validate?: (values: T) => Partial<Record<keyof T, string>>;
}

function useForm<T extends Record<string, unknown>>({
  initialValues,
  onSubmit,
  validate,
}: UseFormOptions<T>) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [touched, setTouched] = useState<Record<keyof T, boolean>>({} as Record<keyof T, boolean>);
  
  const handleChange = useCallback((
    e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>
  ) => {
    const { name, value } = e.target;
    setValues(prev => ({ ...prev, [name]: value }));
    
    // Очищаем ошибку при изменении
    if (errors[name as keyof T]) {
      setErrors(prev => ({ ...prev, [name]: undefined }));
    }
  }, [errors]);
  
  const handleBlur = useCallback((
    e: React.FocusEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>
  ) => {
    const { name } = e.target;
    setTouched(prev => ({ ...prev, [name]: true }));
    
    // Валидация при blur
    if (validate) {
      const validationErrors = validate(values);
      setErrors(validationErrors);
    }
  }, [validate, values]);
  
  const handleSubmit = useCallback(async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);
    
    if (validate) {
      const validationErrors = validate(values);
      setErrors(validationErrors);
      
      if (Object.keys(validationErrors).length > 0) {
        setIsSubmitting(false);
        return;
      }
    }
    
    try {
      await onSubmit?.(values);
    } finally {
      setIsSubmitting(false);
    }
  }, [values, validate, onSubmit]);
  
  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({} as Record<keyof T, boolean>);
    setIsSubmitting(false);
  }, [initialValues]);
  
  return {
    values,
    errors,
    isSubmitting,
    touched,
    handleChange,
    handleBlur,
    handleSubmit,
    reset,
    setValues,
  };
}

// Использование
function LoginForm() {
  const { values, errors, handleChange, handleSubmit, isSubmitting } = useForm({
    initialValues: { email: '', password: '' },
    validate: (values) => {
      const errors: Partial<Record<'email' | 'password', string>> = {};
      if (!values.email) errors.email = 'Required';
      if (!values.password) errors.password = 'Required';
      return errors;
    },
    onSubmit: async (values) => {
      await api.login(values);
    },
  });
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        value={values.email}
        onChange={handleChange}
        onBlur={handleBlur}
      />
      {errors.email && <span>{errors.email}</span>}
      
      <input
        name="password"
        type="password"
        value={values.password}
        onChange={handleChange}
        onBlur={handleBlur}
      />
      {errors.password && <span>{errors.password}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-Basics]] — основы React
- [[React-State-Management]] — Zustand, Redux
- [[React-Forms]] — react-hook-form

---

> [!TIP] Совет
>
> 1. **Выносите логику в хуки** — переиспользование кода
> 2. **Типизируйте хуки** — дженерики для универсальности
> 3. **useCallback для функций** — избегайте лишних ререндеров
> 4. **useRef для предыдущих значений** — отслеживание изменений
> 5. **Дokumentруйте хуки** — что возвращает, какие параметры
