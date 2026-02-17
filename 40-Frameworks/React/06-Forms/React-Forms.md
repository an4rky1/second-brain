---
created: 2026-02-17
tags:
  - cheat-sheet
  - react
  - forms
  - react-hook-form
  - validation
aliases:
  - React Forms
  - React Hook Form Zod
related:
  - React-MOC
  - React-Hooks
---

# React — Формы и валидация

> [!SUMMARY] Обзор
> Работа с формами в React: react-hook-form, валидация с Zod, паттерны обработки форм.

---

## React Hook Form

### Установка

```bash
npm install react-hook-form
npm install @hookform/resolvers zod  # Для валидации
```

### Базовое использование

```tsx
import { useForm } from 'react-hook-form';

interface FormData {
  email: string;
  password: string;
}

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isDirty },
    reset,
    watch,
  } = useForm<FormData>();
  
  const onSubmit = async (data: FormData) => {
    await api.login(data);
    reset();
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('email', {
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email',
          },
        })}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input
        {...register('password', {
          required: 'Password is required',
          minLength: {
            value: 8,
            message: 'Password must be at least 8 characters',
          },
        })}
        type="password"
        placeholder="Password"
      />
      {errors.password && <span>{errors.password.message}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

### Валидация с Zod

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Схема валидации
const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain an uppercase letter')
    .regex(/[a-z]/, 'Password must contain a lowercase letter')
    .regex(/[0-9]/, 'Password must contain a number'),
});

type LoginForm = z.infer<typeof loginSchema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
  });
  
  const onSubmit = async (data: LoginForm) => {
    await api.login(data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} placeholder="Email" />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input
        {...register('password')}
        type="password"
        placeholder="Password"
      />
      {errors.password && <span>{errors.password.message}</span>}
      
      <button type="submit" disabled={isSubmitting}>Login</button>
    </form>
  );
}
```

### Complex Forms

```tsx
const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  confirmPassword: z.string(),
  name: z.string().min(2),
  age: z.number().min(18),
  country: z.string().optional(),
  interests: z.array(z.string()).min(1, 'Select at least one interest'),
  acceptTerms: z.boolean().refine(v => v === true, 'You must accept the terms'),
}).refine(data => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
});

type RegisterForm = z.infer<typeof registerSchema>;

function RegisterForm() {
  const {
    register,
    handleSubmit,
    watch,
    setValue,
    formState: { errors },
  } = useForm<RegisterForm>({
    resolver: zodResolver(registerSchema),
  });
  
  const password = watch('password');
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email', { valueAsString: true })} />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input {...register('password')} type="password" />
      {errors.password && <span>{errors.password.message}</span>}
      
      <input {...register('confirmPassword')} type="password" />
      {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}
      
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}
      
      <input {...register('age', { valueAsNumber: true })} type="number" />
      {errors.age && <span>{errors.age.message}</span>}
      
      <select {...register('country')}>
        <option value="">Select country</option>
        <option value="us">USA</option>
        <option value="uk">UK</option>
      </select>
      
      {/* Checkboxes */}
      <div>
        <label>
          <input
            type="checkbox"
            {...register('interests')}
            value="coding"
          />
          Coding
        </label>
        <label>
          <input
            type="checkbox"
            {...register('interests')}
            value="design"
          />
          Design
        </label>
      </div>
      {errors.interests && <span>{errors.interests.message}</span>}
      
      {/* Terms */}
      <label>
        <input type="checkbox" {...register('acceptTerms')} />
        I accept the terms
      </label>
      {errors.acceptTerms && <span>{errors.acceptTerms.message}</span>}
      
      <button type="submit">Register</button>
    </form>
  );
}
```

### useFieldArray

```tsx
import { useForm, useFieldArray } from 'react-hook-form';

interface Friend {
  id: string;
  name: string;
}

interface FormData {
  name: string;
  friends: Friend[];
}

function DynamicForm() {
  const { register, control, handleSubmit } = useForm<FormData>();
  
  const { fields, append, remove } = useFieldArray({
    control,
    name: 'friends',
  });
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} placeholder="Your name" />
      
      <div>
        {fields.map((field, index) => (
          <div key={field.id}>
            <input
              {...register(`friends.${index}.name`)}
              placeholder="Friend name"
            />
            <button type="button" onClick={() => remove(index)}>
              Remove
            </button>
          </div>
        ))}
      </div>
      
      <button
        type="button"
        onClick={() => append({ id: Date.now().toString(), name: '' })}
      >
        Add Friend
      </button>
      
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Валидация с Zod

### Базовая валидация

```tsx
import { z } from 'zod';

// Строки
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();
const minMaxSchema = z.string().min(2).max(100);
const regexSchema = z.string().regex(/^[A-Z]+$/);

// Числа
const positiveSchema = z.number().positive();
const intSchema = z.number().int();
const rangeSchema = z.number().min(0).max(100);

// Массивы
const arraySchema = z.array(z.string()).min(1);
const exactLengthSchema = z.array(z.number()).length(3);

// Объекты
const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().positive(),
});

// Использование
try {
  userSchema.parse({ name: 'John', email: 'john@example.com', age: 30 });
} catch (error) {
  if (error instanceof z.ZodError) {
    console.log(error.errors);
  }
}
```

### Refine и Transform

```tsx
// Refine для кастомной валидации
const passwordSchema = z.string().refine(
  (val) => /[A-Z]/.test(val) && /[a-z]/.test(val) && /[0-9]/.test(val),
  { message: 'Password must contain uppercase, lowercase, and number' }
);

// Refine с несколькими условиями
const userSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  { message: "Passwords don't match", path: ['confirmPassword'] }
);

// Transform для преобразования
const numberSchema = z.string().transform((val) => Number(val));

const userSchema = z.object({
  email: z.string().email().transform((val) => val.toLowerCase()),
  age: z.string().transform((val) => parseInt(val, 10)),
});

// Preprocess
const schema = z.object({
  name: z.string().preprocess((val) => String(val).trim(), z.string().min(1)),
});
```

### Schema Composition

```tsx
// Extend
const baseUserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
});

const adminUserSchema = baseUserSchema.extend({
  role: z.literal('admin'),
  permissions: z.array(z.string()),
});

// Merge
const personalInfoSchema = z.object({
  name: z.string(),
  age: z.number(),
});

const contactInfoSchema = z.object({
  email: z.string().email(),
  phone: z.string().optional(),
});

const userSchema = personalInfoSchema.merge(contactInfoSchema);

// Partial / Pick / Omit
const partialUser = userSchema.partial(); // Все поля опциональные
const userId = userSchema.pick({ id: true }); // Только id
const userNoId = userSchema.omit({ id: true }); // Без id
```

---

## Формы без библиотек

### Controlled Form

```tsx
function ControlledForm() {
  const [values, setValues] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState<Record<string, string>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setValues(prev => ({ ...prev, [name]: value }));
    // Clear error on change
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: '' }));
    }
  };
  
  const validate = () => {
    const newErrors: Record<string, string> = {};
    if (!values.email) newErrors.email = 'Email is required';
    if (!values.password) newErrors.password = 'Password is required';
    return newErrors;
  };
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const newErrors = validate();
    
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    
    setIsSubmitting(true);
    try {
      await api.login(values);
    } catch (error) {
      setErrors({ form: 'Login failed' });
    } finally {
      setIsSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      {errors.form && <div className="error">{errors.form}</div>}
      
      <input
        name="email"
        value={values.email}
        onChange={handleChange}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email}</span>}
      
      <input
        name="password"
        type="password"
        value={values.password}
        onChange={handleChange}
        placeholder="Password"
      />
      {errors.password && <span>{errors.password}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

### Uncontrolled Form

```tsx
function UncontrolledForm() {
  const formRef = useRef<HTMLFormElement>(null);
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);
    
    const formData = new FormData(e.currentTarget);
    const data = {
      email: formData.get('email') as string,
      password: formData.get('password') as string,
    };
    
    try {
      await api.login(data);
      formRef.current?.reset();
    } finally {
      setIsSubmitting(false);
    }
  };
  
  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      <input name="email" type="email" placeholder="Email" required />
      <input name="password" type="password" placeholder="Password" required />
      <button type="submit" disabled={isSubmitting}>Login</button>
    </form>
  );
}
```

---

## Паттерны

### Multi-step Form

```tsx
const step1Schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

const step2Schema = z.object({
  name: z.string().min(2),
  age: z.number().positive(),
});

type Step1Data = z.infer<typeof step1Schema>;
type Step2Data = z.infer<typeof step2Schema>;

function MultiStepForm() {
  const [step, setStep] = useState(1);
  const [data, setData] = useState<Step1Data & Step2Data | null>(null);
  
  const {
    register: registerStep1,
    handleSubmit: handleSubmitStep1,
    formState: { errors: errorsStep1 },
  } = useForm<Step1Data>({
    resolver: zodResolver(step1Schema),
  });
  
  const {
    register: registerStep2,
    handleSubmit: handleSubmitStep2,
    formState: { errors: errorsStep2 },
  } = useForm<Step2Data>({
    resolver: zodResolver(step2Schema),
  });
  
  const onStep1Submit = (step1Data: Step1Data) => {
    setData(prev => ({ ...prev, ...step1Data } as Step1Data & Step2Data));
    setStep(2);
  };
  
  const onStep2Submit = (step2Data: Step2Data) => {
    const formData = { ...data, ...step2Data };
    api.register(formData);
  };
  
  return (
    <form>
      {step === 1 && (
        <>
          <input {...registerStep1('email')} />
          {errorsStep1.email && <span>{errorsStep1.email.message}</span>}
          
          <input {...registerStep1('password')} type="password" />
          {errorsStep1.password && <span>{errorsStep1.password.message}</span>}
          
          <button type="button" onClick={handleSubmitStep1(onStep1Submit)}>
            Next
          </button>
        </>
      )}
      
      {step === 2 && (
        <>
          <input {...registerStep2('name')} />
          {errorsStep2.name && <span>{errorsStep2.name.message}</span>}
          
          <input {...registerStep2('age')} type="number" />
          {errorsStep2.age && <span>{errorsStep2.age.message}</span>}
          
          <button type="button" onClick={() => setStep(1)}>Back</button>
          <button type="button" onClick={handleSubmitStep2(onStep2Submit)}>
            Submit
          </button>
        </>
      )}
    </form>
  );
}
```

### Dependent Fields

```tsx
function DependentFields() {
  const { register, watch, setValue } = useForm();
  
  const country = watch('country');
  
  // Обновляем города при изменении страны
  useEffect(() => {
    setValue('city', ''); // Сбрасываем город
  }, [country, setValue]);
  
  const cities = country === 'us' 
    ? ['New York', 'Los Angeles'] 
    : country === 'uk' 
    ? ['London', 'Manchester'] 
    : [];
  
  return (
    <>
      <select {...register('country')}>
        <option value="">Select country</option>
        <option value="us">USA</option>
        <option value="uk">UK</option>
      </select>
      
      <select {...register('city')} disabled={!country}>
        <option value="">Select city</option>
        {cities.map(city => (
          <option key={city} value={city}>{city}</option>
        ))}
      </select>
    </>
  );
}
```

---

## Best Practices

### ✅ Делать

```tsx
// 1. Используйте react-hook-form для сложных форм
const { register, handleSubmit } = useForm();

// 2. Валидация с Zod
const schema = z.object({ email: z.string().email() });
useForm({ resolver: zodResolver(schema) });

// 3. Типизируйте формы
type FormData = z.infer<typeof schema>;
const { register } = useForm<FormData>();

// 4. Обрабатывайте isSubmitting
<button disabled={isSubmitting}>{isSubmitting ? 'Loading...' : 'Submit'}</button>

// 5. Сбрасывайте форму после успешной отправки
const onSubmit = async (data) => {
  await api.submit(data);
  reset();
};
```

### ❌ Не делать

```tsx
// 1. Не храните состояние формы в useState
const [email, setEmail] = useState(''); // ❌
const { register } = useForm(); // ✅

// 2. Не валидируйте вручную без нужды
// Используйте resolver с Zod

// 3. Не забывайте про valueAsNumber
<input {...register('age', { valueAsNumber: true })} type="number" />

// 4. Не игнорируйте errors
{errors.email && <span>{errors.email.message}</span>}
```

---

## 🔗 Связанные заметки

- [[React-MOC]] — индекс раздела
- [[React-Hooks]] — кастомные хуки
- [[React-State-Management]] — Zustand
- [[TypeScript-DataFetching]] — API запросы

---

> [!TIP] Совет
>
> 1. **react-hook-form для большинства форм** — меньше ререндеров
> 2. **Zod для валидации** — типобезопасность
> 3. **Uncontrolled для простых форм** — меньше кода
> 4. **useFieldArray для динамических полей** — добавление/удаление
> 5. **Multi-step для длинных форм** — разбивайте на шаги
