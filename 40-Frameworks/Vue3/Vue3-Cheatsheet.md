---
created: 2026-02-16
tags:
  - cheat-sheet
  - vue
  - vue3
  - frontend
  - javascript
aliases:
  - Vue 3 Cheatsheet
  - Vue.js Reference
related:
  - TypeScript-Cheatsheet
  - React-Cheatsheet
  - Nuxt-Cheatsheet
---

# Vue 3 — Полная шпаргалка

> [!SUMMARY] Обзор
> Vue 3 — прогрессивный JavaScript фреймворк. Composition API, реактивность через Proxy, Teleport, Suspense. Проще React, гибче Angular.

---

## 📚 Теория

### Options API vs Composition API

```vue
<!-- Options API (Vue 2 стиль) -->
<script>
export default {
  data() {
    return { count: 0 }
  },
  methods: {
    increment() { this.count++ }
  },
  computed: {
    double() { return this.count * 2 }
  }
}
</script>

<!-- Composition API (Vue 3, рекомендуется) -->
<script setup>
import { ref, computed } from 'vue'

const count = ref(0)
const increment = () => count.value++
const double = computed(() => count.value * 2)
</script>
```

### Реактивность

```typescript
import { ref, reactive, computed, watch, watchEffect } from 'vue'

// ref — для примитивов и объектов
const count = ref(0)
count.value++

// reactive — для объектов
const state = reactive({ count: 0, name: 'Vue' })
state.count++

// computed — вычисляемые значения
const double = computed(() => count.value * 2)
const writable = computed({
  get: () => count.value,
  set: (val) => count.value = val
})

// watch — слежение за изменениями
watch(count, (newVal, oldVal) => {
  console.log(`Changed from ${oldVal} to ${newVal}`)
})

watch([count, name], ([newCount, newName], [oldCount, oldName]) => {})

// watchEffect — авто-зависимости
watchEffect(() => {
  console.log(`Count is ${count.value}`)
})
```

---

## ⚡ Быстрый старт

```bash
# Создание проекта
npm create vue@latest my-app
cd my-app
npm install
npm run dev

# Или с Vite
npm create vite@latest my-app -- --template vue-ts

# Структура
src/
├── components/
├── composables/      # Reusable logic
├── views/           # Page components
├── router/
├── stores/          # Pinia stores
├── assets/
└── App.vue

# Запуск
npm run dev      # Development
npm run build    # Production
npm run preview  # Preview
```

---

## 🔧 Практические примеры

### Template Syntax

```vue
<template>
  <!-- Интерполяция -->
  <div>{{ message }}</div>
  <div v-text="message"></div>
  <div v-html="rawHtml"></div>

  <!-- Директивы -->
  <div v-if="show">If</div>
  <div v-else-if="maybe">Else If</div>
  <div v-else>Else</div>

  <div v-show="visible">Show/Hide</div>

  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
    </li>
  </ul>

  <!-- События -->
  <button @click="handleClick">Click</button>
  <button @click.prevent="submit">Submit</button>
  <input @keyup.enter="onEnter" />
  <input @input.debounce="onInput" />

  <!-- Привязки -->
  <input :value="text" @input="text = $event.target.value" />
  <input v-model="text" />

  <input type="checkbox" v-model="checked" />
  <input type="radio" v-model="picked" />
  <select v-model="selected">
    <option value="1">One</option>
  </select>

  <!-- Class binding -->
  <div :class="{ active: isActive, 'text-danger': hasError }"></div>
  <div :class="[activeClass, errorClass]"></div>

  <!-- Style binding -->
  <div :style="{ color: activeColor, fontSize: fontSize + 'px' }"></div>

  <!-- Slots -->
  <MyComponent>
    <template #header>
      <h1>Header</h1>
    </template>
    <template #default="{ data }">
      <p>{{ data }}</p>
    </template>
  </MyComponent>

  <!-- Teleport -->
  <teleport to="body">
    <div class="modal">Modal content</div>
  </teleport>

  <!-- Suspense (экспериментальный) -->
  <Suspense>
    <template #default>
      <AsyncComponent />
    </template>
    <template #fallback>
      <Loading />
    </template>
  </Suspense>
</template>
```

### Components

```vue
<!-- ParentComponent.vue -->
<script setup>
import { ref } from 'vue'
import ChildComponent from './ChildComponent.vue'

const message = ref('Hello from parent')
const handleEvent = (data) => console.log(data)
</script>

<template>
  <ChildComponent 
    :message="message"
    :count="5"
    @custom-event="handleEvent"
  >
    <template #default="{ data }">
      <span>{{ data }}</span>
    </template>
  </ChildComponent>
</template>

<!-- ChildComponent.vue -->
<script setup>
import { ref } from 'vue'

// Props
const props = defineProps({
  message: {
    type: String,
    required: true,
    default: 'Default'
  },
  count: {
    type: Number,
    default: 0
  }
})

// Emits
const emit = defineEmits(['custom-event', 'update:modelValue'])

// Вызов события
const handleClick = () => {
  emit('custom-event', { data: 'from child' })
}

// v-model
const model = defineModel()
</script>

<template>
  <div>
    <p>{{ message }}</p>
    <button @click="handleClick">Send</button>
  </div>
</template>

<!-- Provide/Inject -->
<!-- Parent -->
<script setup>
import { provide, ref } from 'vue'

const theme = ref('dark')
provide('theme', theme)
</script>

<!-- Child (любой уровень вложенности) -->
<script setup>
import { inject } from 'vue'

const theme = inject('theme')
</script>
```

### Composables

```typescript
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  
  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => count.value = initialValue
  
  const double = computed(() => count.value * 2)
  
  return {
    count,
    increment,
    decrement,
    reset,
    double
  }
}

// composables/useFetch.ts
import { ref, watch } from 'vue'

export function useFetch<T>(url: string) {
  const data = ref<T | null>(null)
  const loading = ref(true)
  const error = ref<Error | null>(null)
  
  const fetchData = async () => {
    loading.value = true
    try {
      const response = await fetch(url)
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }
  
  watch(() => url, fetchData, { immediate: true })
  
  return { data, loading, error, refetch: fetchData }
}

// composables/useLocalStorage.ts
import { ref, watch } from 'vue'

export function useLocalStorage<T>(key: string, initialValue: T) {
  const stored = localStorage.getItem(key)
  const value = ref<T>(stored ? JSON.parse(stored) : initialValue)
  
  watch(value, (newValue) => {
    localStorage.setItem(key, JSON.stringify(newValue))
  }, { deep: true })
  
  return value
}

// Использование
<script setup>
import { useCounter } from '@/composables/useCounter'

const { count, increment, double } = useCounter(10)
</script>
```

### Router (Vue Router 4)

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import type { RouteRecordRaw } from 'vue-router'

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/views/HomeView.vue')
  },
  {
    path: '/about',
    name: 'About',
    component: () => import('@/views/AboutView.vue')
  },
  {
    path: '/users/:id',
    name: 'User',
    component: () => import('@/views/UserView.vue'),
    props: true  // Передача params как props
  },
  {
    path: '/dashboard',
    component: () => import('@/layouts/DashboardLayout.vue'),
    children: [
      {
        path: '',
        name: 'Dashboard',
        component: () => import('@/views/DashboardView.vue')
      }
    ]
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

// Navigation Guards
router.beforeEach((to, from, next) => {
  const isAuthenticated = localStorage.getItem('token')
  
  if (to.meta.requiresAuth && !isAuthenticated) {
    next('/login')
  } else {
    next()
  }
})

export default router

// Использование в компоненте
<script setup>
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

// Навигация
router.push('/path')
router.push({ name: 'User', params: { id: 1 } })
router.replace('/path')
router.go(-1)  // Back

// Доступ к параметрам
const userId = route.params.id
const query = route.query.search
</script>
```

### State Management (Pinia)

```typescript
// stores/counter.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  // State
  const count = ref(0)
  const name = ref('Vue')
  
  // Getters
  const doubleCount = computed(() => count.value * 2)
  
  // Actions
  function increment() {
    count.value++
  }
  
  function incrementBy(amount: number) {
    count.value += amount
  }
  
  async function fetchData() {
    const response = await fetch('/api/data')
    count.value = await response.json()
  }
  
  return { count, name, doubleCount, increment, incrementBy, fetchData }
})

// Setup store (с классом)
export const useUserStore = defineStore('user', {
  state: () => ({
    user: null as User | null,
    token: null as string | null
  }),
  getters: {
    isAuthenticated: (state) => !!state.token
  },
  actions: {
    async login(credentials: LoginCredentials) {
      const response = await api.login(credentials)
      this.token = response.token
      this.user = response.user
    },
    logout() {
      this.token = null
      this.user = null
    }
  }
})

// Использование
<script setup>
import { useCounterStore } from '@/stores/counter'
import { storeToRefs } from 'pinia'

const store = useCounterStore()

// Деструктуризация с реактивностью
const { count, doubleCount } = storeToRefs(store)
const { increment } = store

// Изменение
store.count++
store.increment()
</script>
```

---

## 🎯 Best Practices

### ✅ Делать

```vue
<!-- 1. Используйте <script setup> -->
<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

<!-- 2. Типизируйте props -->
<script setup lang="ts">
interface Props {
  id: number
  title?: string
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Default Title'
})
</script>

<!-- 3. Composables для логики -->
// composables/useUsers.ts
export function useUsers() {
  const users = ref([])
  const loading = ref(false)
  
  const fetchUsers = async () => {
    loading.value = true
    users.value = await api.getUsers()
    loading.value = false
  }
  
  return { users, loading, fetchUsers }
}

<!-- 4. v-for с key -->
<li v-for="user in users" :key="user.id">{{ user.name }}</li>

<!-- 5. Lazy loading роутов -->
const routes = [
  {
    path: '/about',
    component: () => import('@/views/AboutView.vue')
  }
]
```

### ❌ Не делать

```vue
<!-- 1. Избегать this в <script setup> -->
<script setup>
this.count  // ❌
count.value  // ✅
</script>

<!-- 2. Не мутировать props напрямую -->
<script setup>
const props = defineProps(['modelValue'])

props.modelValue = 'new'  // ❌

const emit = defineEmits(['update:modelValue'])
emit('update:modelValue', 'new')  // ✅
</script>

<!-- 3. Избегать v-if с v-for на одном элементе -->
<div v-for="item in items" v-if="item.active">  <!-- ❌ -->
  
<template v-for="item in items" :key="item.id">
  <div v-if="item.active">{{ item.name }}</div>
</template>  <!-- ✅ -->

<!-- 4. Не создавать computed в template -->
{{ items.filter(i => i.active).length }}  <!-- ❌ -->

<script setup>
const activeCount = computed(() => items.filter(i => i.active).length)
</script>
{{ activeCount }}  <!-- ✅ -->
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `ref.value is undefined` | Доступ без .value | Используйте `.value` для ref |
| `props are reactive` | Мутация props | Используйте emit для изменений |
| `v-if with v-for` | Приоритет директив | Используйте template wrapper |
| `Key prop missing` | Нет key в v-for | Добавьте уникальный key |
| `Composable not reactive` | Неправильное возвращение | Возвращайте ref/reactive |

---

## 🔗 Связанные заметки

- [[TypeScript-Cheatsheet]] — TypeScript
- [[React-Cheatsheet]] — React для сравнения
- [[Nuxt-Cheatsheet]] — Nuxt.js фреймворк

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Composition API** — используйте для новой логики
> 2. **Composables** — переиспользуйте логику
> 3. **Pinia > Vuex** — проще и типобезопаснее
> 4. **<script setup>** — меньше бойлерплейта
> 5. **Vue DevTools** — обязательно для отладки

> [!INFO] Полезные библиотеки
> ```bash
> # State
> npm install pinia
> 
> # Router
> npm install vue-router
> 
> # UI
> npm install @vueuse/components  # Composition utilities
> npm install element-plus        # UI framework
> npm install primevue            # UI components
> 
> # Forms
> npm install vee-validate yup    # Валидация
> 
> # HTTP
> npm install axios
> npm install @tanstack/vue-query # React Query для Vue
> ```
