---
created: 2026-02-17
tags:
  - cheat-sheet
  - typescript
  - config
  - tsconfig
aliases:
  - TSConfig
  - TypeScript Config
related:
  - TypeScript-MOC
  - TypeScript-Basics
  - TypeScript-DataFetching
---

# TypeScript — tsconfig.json

> [!SUMMARY] Обзор
> Полная расшифровка всех полей `tsconfig.json` — конфигурация компилятора TypeScript.

---

## Пример конфигурации

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020", "DOM"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "moduleResolution": "node",
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"],
      "@utils/*": ["utils/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## Project Layout

| Поле | Описание | Значения | Пример |
|------|----------|----------|--------|
| `target` | Целевая версия JS | `ES3`, `ES5`, `ES6/ES2015`, `ES2016`–`ES2024`, `ESNext` | `"ES2020"` |
| `module` | Система модулей | `commonjs`, `amd`, `umd`, `es6`, `esnext`, `node16`, `nodenext` | `"commonjs"` |
| `lib` | Библиотеки типов | `DOM`, `ES5`, `ES2015`, `ES2020`, `WebWorker` | `["ES2020", "DOM"]` |
| `outDir` | Директория вывода | путь | `"./dist"` |
| `rootDir` | Корневая директория | путь | `"./src"` |
| `baseUrl` | Базовый путь для импортов | путь | `"./src"` |
| `paths` | Маппинг импортов | объект | `{ "@/*": ["./*"] }` |
| `rootDirs` | Несколько корневых директорий | массив | `["./src", "./generated"]` |
| `typeRoots` | Директории типов | массив | `["./types", "./node_modules/@types"]` |
| `types` | Включаемые типы | массив | `["node", "jest"]` |

### Примеры

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "outDir": "./dist",
    "rootDir": "./src",
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"],
      "@components/*": ["components/*"],
      "@utils/*": ["utils/*"]
    },
    "typeRoots": ["./node_modules/@types", "./types"],
    "types": ["node", "jest", "react"]
  }
}
```

---

## Type Checking (Строгость)

| Поле | Описание | Default | Рекомендация |
|------|----------|---------|--------------|
| `strict` | Включает все строгие проверки | `false` | ✅ `true` |
| `noImplicitAny` | Ошибка при неявном `any` | `false` | ✅ `true` |
| `strictNullChecks` | null/undefined отдельные типы | `false` | ✅ `true` |
| `strictFunctionTypes` | Строгая проверка функций | `false` | ✅ `true` |
| `strictBindCallApply` | Строгая проверка bind/call/apply | `false` | ✅ `true` |
| `strictPropertyInitialization` | Требует инициализации свойств | `false` | ✅ `true` |
| `noImplicitThis` | Ошибка при неявном `this` | `false` | ✅ `true` |
| `useUnknownInCatchVariables` | catch переменная как `unknown` | `false` | ✅ `true` |
| `alwaysStrict` | Добавляет `"use strict"` | `false` | ✅ `true` |
| `noUnusedLocals` | Ошибка для неиспользуемых локальных | `false` | ✅ `true` |
| `noUnusedParameters` | Ошибка для неиспользуемых параметров | `false` | ✅ `true` |
| `exactOptionalPropertyTypes` | Строгие опциональные свойства | `false` | ✅ `true` |
| `noImplicitReturns` | Все пути возвращают значение | `false` | ✅ `true` |
| `noFallthroughCasesInSwitch` | Ошибка при fallthrough в switch | `false` | ✅ `true` |
| `noUncheckedIndexedAccess` | Добавляет `undefined` к индексам | `false` | ✅ `true` |
| `noImplicitOverride` | Требует `override` для переопределения | `false` | ✅ `true` |
| `noPropertyAccessFromIndexSignature` | Требует `[]` для dynamic keys | `false` | ⚠️ По ситуации |

### Примеры

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

> [!TIP] strict mode
>
> `"strict": true` включает сразу все проверки. Лучше включить сразу, чем добавлять постепенно.

---

## Module Resolution

| Поле | Описание | Значения | Рекомендация |
|------|----------|----------|--------------|
| `moduleResolution` | Алгоритм разрешения модулей | `node`, `node16`, `nodenext`, `bundler` | `"node"` или `"bundler"` |
| `resolveJsonModule` | Импорт `.json` файлов | `boolean` | ✅ `true` |
| `allowSyntheticDefaultImports` | Default imports из modules без default | `boolean` | ✅ `true` |
| `esModuleInterop` | Совместимость с CommonJS | `boolean` | ✅ `true` |
| `isolatedModules` | Каждый файл как отдельный модуль | `boolean` | ✅ `true` (нужно для Babel) |
| `allowImportingTsExtensions` | Импорт с `.ts` расширением | `boolean` | ⚠️ Для ESM |
| `verbatimModuleSyntax` | Сохраняет синтаксис импортов | `boolean` | ✅ `true` |

### Примеры

```json
{
  "compilerOptions": {
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  }
}
```

---

## Emit (Генерация кода)

| Поле | Описание | Default | Рекомендация |
|------|----------|---------|--------------|
| `declaration` | Генерировать `.d.ts` файлы | `false` | ✅ `true` для библиотек |
| `declarationMap` | Source maps для `.d.ts` | `false` | ✅ `true` для библиотек |
| `sourceMap` | Генерировать `.map` файлы | `false` | ✅ `true` для отладки |
| `inlineSourceMap` | Inline source maps | `false` | ⚠️ По ситуации |
| `inlineSources` | Включать исходники в map | `false` | ⚠️ По ситуации |
| `emitDecoratorMetadata` | Metadata для декораторов | `false` | ✅ `true` для NestJS/Angular |
| `experimentalDecorators` | Включить декораторы | `false` | ✅ `true` для NestJS/Angular |
| `importHelpers` | Импорт helpers из tslib | `false` | ✅ `true` |
| `removeComments` | Удалять комментарии | `false` | ⚠️ Для production |
| `preserveConstEnums` | Сохранять const enum | `false` | ✅ `true` |
| `outFile` | Один файл для вывода | - | ❌ Не использовать |

### Примеры

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "importHelpers": true,
    "preserveConstEnums": true
  }
}
```

---

## JavaScript Support

| Поле | Описание | Рекомендация |
|------|----------|--------------|
| `allowJs` | Разрешить `.js` файлы | ✅ `true` для миграции |
| `checkJs` | Проверять типы в `.js` | ⚠️ По ситуации |
| `maxNodeModuleJsDepth` | Глубина проверки node_modules | `0` |

### Примеры

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "maxNodeModuleJsDepth": 0
  }
}
```

---

## Interop Constraints

| Поле | Описание | Рекомендация |
|------|----------|--------------|
| `forceConsistentCasingInFileNames` | Регистр имён файлов | ✅ `true` |
| `skipLibCheck` | Пропускать проверку `.d.ts` | ✅ `true` (ускоряет сборку) |
| `skipDefaultLibCheck` | Пропускать проверку built-in libs | ✅ `true` |

---

## Project References

```json
{
  "extends": "./tsconfig.base.json",
  "references": [
    { "path": "./packages/utils" },
    { "path": "./packages/core" }
  ],
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

| Поле | Описание |
|------|----------|
| `extends` | Наследование конфигурации |
| `references` | Ссылки на другие проекты (composite) |
| `include` | Включаемые файлы (glob паттерны) |
| `exclude` | Исключаемые файлы |

---

## Готовые конфигурации

### Node.js (CommonJS)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Node.js (ESM)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### React (Vite)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### Библиотека

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## Полезные команды

```bash
# Инициализация
npx tsc --init

# Инициализация со strict
npx tsc --init --strict

# Проверка типов без компиляции
npx tsc --noEmit

# Показать конфигурацию
npx tsc --showConfig

# Компиляция
npx tsc

# Watch режим
npx tsc --watch

# Компиляция с переопределением
npx tsc --target ES2020 --module commonjs
```

---

## 🔗 Связанные заметки

- [[TypeScript-MOC]] — индекс раздела
- [[TypeScript-Basics]] — основы языка
- [[TypeScript-Utility-Types]] — типы и дженерики
- [[TypeScript-DataFetching]] — fetch, API
