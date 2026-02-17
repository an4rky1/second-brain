---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - ecosystem
  - libraries
aliases:
  - React Ecosystem
  - React Libraries
related:
  - React-MOC
  - React-Performance
---

# React — Экосистема и библиотеки

> [!SUMMARY] Обзор
> Популярные библиотеки и инструменты для React: UI компоненты, иконки, анимации, HTTP клиенты, утилиты.

---

## UI Библиотеки

### Radix UI

```bash
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu
```

```tsx
// Dialog
import * as Dialog from '@radix-ui/react-dialog';

function Modal({ open, onOpenChange, children }) {
  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay className="overlay" />
        <Dialog.Content className="content">
          <Dialog.Title>Title</Dialog.Title>
          {children}
          <Dialog.Close>✕</Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}

// Dropdown Menu
import * as DropdownMenu from '@radix-ui/react-dropdown-menu';

function Menu() {
  return (
    <DropdownMenu.Root>
      <DropdownMenu.Trigger>
        <button>⋮</button>
      </DropdownMenu.Trigger>
      
      <DropdownMenu.Portal>
        <DropdownMenu.Content>
          <DropdownMenu.Item>Edit</DropdownMenu.Item>
          <DropdownMenu.Item>Delete</DropdownMenu.Item>
          <DropdownMenu.Separator />
          <DropdownMenu.Item>Settings</DropdownMenu.Item>
        </DropdownMenu.Content>
      </DropdownMenu.Portal>
    </DropdownMenu.Root>
  );
}
```

### Headless UI

```bash
npm install @headlessui/react
```

```tsx
import { Menu, Transition } from '@headlessui/react';
import { Fragment } from 'react';

function MyMenu() {
  return (
    <Menu as="div" className="relative">
      <Menu.Button>Options</Menu.Button>
      
      <Transition
        as={Fragment}
        enter="transition ease-out duration-100"
        enterFrom="transform opacity-0 scale-95"
        enterTo="transform opacity-100 scale-100"
        leave="transition ease-in duration-75"
        leaveFrom="transform opacity-100 scale-100"
        leaveTo="transform opacity-0 scale-95"
      >
        <Menu.Items className="absolute right-0">
          <Menu.Item>
            {({ active }) => (
              <button className={active ? 'bg-blue-500' : ''}>
                Edit
              </button>
            )}
          </Menu.Item>
        </Menu.Items>
      </Transition>
    </Menu>
  );
}
```

### Chakra UI

```bash
npm install @chakra-ui/react @emotion/react @emotion/styled framer-motion
```

```tsx
import { ChakraProvider, Box, Button, VStack } from '@chakra-ui/react';

function App() {
  return (
    <ChakraProvider>
      <VStack spacing={4}>
        <Box p={4} bg="blue.500" color="white">
          Hello World
        </Box>
        <Button colorScheme="teal">Click me</Button>
      </VStack>
    </ChakraProvider>
  );
}
```

---

## Иконки

### React Icons

```bash
npm install react-icons
```

```tsx
import { FaHome, FaUser, FaSettings } from 'react-icons/fa';
import { AiOutlineLoading } from 'react-icons/ai';
import { BsCheckCircle } from 'react-icons/bs';

function Navigation() {
  return (
    <nav>
      <FaHome size={24} />
      <FaUser size={24} />
      <FaSettings size={24} />
    </nav>
  );
}

function Loading() {
  return <AiOutlineLoading className="animate-spin" size={24} />;
}
```

### Lucide React

```bash
npm install lucide-react
```

```tsx
import { 
  Home, 
  User, 
  Settings, 
  CheckCircle, 
  XCircle,
  AlertCircle,
  Loader2,
  ChevronDown,
  Search,
  Menu,
  X,
} from 'lucide-react';

function Icons() {
  return (
    <>
      <Home size={24} strokeWidth={2} />
      <User className="text-blue-500" size={24} />
      <Loader2 className="animate-spin" size={24} />
    </>
  );
}
```

---

## Анимации

### Framer Motion

```bash
npm install framer-motion
```

```tsx
import { motion, AnimatePresence } from 'framer-motion';

// Базовая анимация
function FadeIn({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
      transition={{ duration: 0.3 }}
    >
      {children}
    </motion.div>
  );
}

// Анимация при скролле
function ScrollReveal({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 50 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true }}
      transition={{ duration: 0.5 }}
    >
      {children}
    </motion.div>
  );
}

// Анимация появления списка
function AnimatedList({ items }) {
  return (
    <motion.ul>
      <AnimatePresence>
        {items.map((item, index) => (
          <motion.li
            key={item.id}
            initial={{ opacity: 0, x: -20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: 20 }}
            transition={{ delay: index * 0.1 }}
          >
            {item.name}
          </motion.li>
        ))}
      </AnimatePresence>
    </motion.ul>
  );
}

// Drag
function DraggableBox() {
  return (
    <motion.div
      drag
      dragConstraints={{ left: 0, right: 300, top: 0, bottom: 300 }}
      whileHover={{ scale: 1.1 }}
      whileTap={{ scale: 0.9 }}
    >
      Drag me!
    </motion.div>
  );
}

// Layout анимации
function LayoutAnimation() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <motion.div layout>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      <AnimatePresence>
        {isOpen && (
          <motion.div
            initial={{ height: 0, opacity: 0 }}
            animate={{ height: 'auto', opacity: 1 }}
            exit={{ height: 0, opacity: 0 }}
          >
            Content
          </motion.div>
        )}
      </AnimatePresence>
    </motion.div>
  );
}
```

---

## HTTP Клиенты

### Axios

```bash
npm install axios
```

```tsx
import axios from 'axios';

// Создание инстанса
const api = axios.create({
  baseURL: '/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptors
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
    }
    return Promise.reject(error);
  }
);

// Использование
const response = await api.get('/users');
const user = await api.post('/users', { name: 'John' });
```

### React Query (TanStack Query)

```bash
npm install @tanstack/react-query
```

Смотрите [[React-Data-Fetching]]

---

## Даты и время

### date-fns

```bash
npm install date-fns
```

```tsx
import { 
  format, 
  formatDistance, 
  parseISO, 
  addDays, 
  isBefore,
  startOfDay,
  endOfDay,
} from 'date-fns';
import { ru } from 'date-fns/locale';

// Форматирование
format(new Date(), 'dd.MM.yyyy'); // "17.02.2026"
format(new Date(), 'HH:mm'); // "14:30"
format(new Date(), 'PPP p', { locale: ru }); // "17 февраля 2026 г. 14:30"

// Относительное время
formatDistance(new Date(), addDays(new Date(), -5), { addSuffix: true });
// "5 дней назад"

// Парсинг
parseISO('2024-01-15T10:30:00Z');

// Манипуляции
addDays(new Date(), 7); // +7 дней
startOfDay(new Date()); // Начало дня
endOfDay(new Date());   // Конец дня

// Сравнение
isBefore(new Date(), addDays(new Date(), 1)); // true
```

### Day.js

```bash
npm install dayjs
```

```tsx
import dayjs from 'dayjs';
import 'dayjs/locale/ru';
import relativeTime from 'dayjs/plugin/relativeTime';

dayjs.extend(relativeTime);
dayjs.locale('ru');

// Форматирование
dayjs().format('DD.MM.YYYY'); // "17.02.2026"
dayjs().format('HH:mm'); // "14:30"

// Относительное время
dayjs().subtract(5, 'day').fromNow(); // "5 дней назад"

// Манипуляции
dayjs().add(7, 'day');
dayjs().startOf('day');
dayjs().endOf('day');
```

---

## Утилиты

### Classnames

```bash
npm install classnames
```

```tsx
import classNames from 'classnames';

// Базовое использование
classNames('foo', 'bar'); // 'foo bar'
classNames('foo', { bar: true }); // 'foo bar'
classNames({ 'foo-bar': true }); // 'foo-bar'

// Массивы
classNames(['foo', 'bar']); // 'foo bar'

// Объекты
classNames({ foo: true, bar: false }); // 'foo'

// Использование в компонентах
function Button({ className, variant = 'primary', size = 'md' }) {
  return (
    <button
      className={classNames(
        'btn',
        `btn-${variant}`,
        `btn-${size}`,
        className
      )}
    >
      Click
    </button>
  );
}
```

### clsx + tailwind-merge

```bash
npm install clsx tailwind-merge
```

```tsx
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

// Утилита для Tailwind
function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Использование
function Button({ className, variant = 'primary' }) {
  return (
    <button
      className={cn(
        'px-4 py-2 rounded',
        variant === 'primary' && 'bg-blue-500 text-white',
        variant === 'secondary' && 'bg-gray-500 text-white',
        className
      )}
    >
      Click
    </button>
  );
}
```

### UUID

```bash
npm install uuid
npm install -D @types/uuid
```

```tsx
import { v4 as uuidv4 } from 'uuid';

const id = uuidv4(); // "550e8400-e29b-41d4-a716-446655440000"

// Для React key
{items.map(item => (
  <Item key={uuidv4()} item={item} />
))}
```

---

## Тестирование

### React Testing Library

```bash
npm install -D @testing-library/react @testing-library/jest-dom vitest
```

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';

describe('LoginForm', () => {
  it('renders form fields', () => {
    render(<LoginForm />);
    
    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
  });
  
  it('submits form with valid data', async () => {
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);
    
    await userEvent.type(screen.getByLabelText(/email/i), 'test@example.com');
    await userEvent.type(screen.getByLabelText(/password/i), 'password123');
    await userEvent.click(screen.getByRole('button', { name: /login/i }));
    
    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123',
      });
    });
  });
  
  it('shows error for invalid email', async () => {
    render(<LoginForm />);
    
    await userEvent.type(screen.getByLabelText(/email/i), 'invalid');
    await userEvent.click(screen.getByRole('button', { name: /login/i }));
    
    expect(await screen.findByText(/invalid email/i)).toBeInTheDocument();
  });
});
```

### MSW (Mock Service Worker)

```bash
npm install -D msw
```

```tsx
// mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' },
    ]);
  }),
  
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 3, ...body }, { status: 201 });
  }),
];

// test setup
import { setupServer } from 'msw/node';
import { handlers } from '../mocks/handlers';

const server = setupServer(...handlers);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-Data-Fetching]] — TanStack Query
- [[React-Forms]] — react-hook-form
- [[React-Performance]] — оптимизация

---

> [!TIP] Совет
>
> 1. **Radix UI для доступности** — headless компоненты
> 2. **Framer Motion для анимаций** — просто и мощно
> 3. **date-fns для дат** — легковесная альтернатива moment
> 4. **clsx + tailwind-merge для классов** — удобно с Tailwind
> 5. **Testing Library для тестов** — тестируйте как пользователь
