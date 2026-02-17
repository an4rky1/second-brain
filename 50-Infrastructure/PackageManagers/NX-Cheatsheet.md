---
created: 2026-02-16
tags:
  - cheat-sheet
  - nx
  - monorepo
  - build-system
aliases:
  - NX Cheatsheet
  - Nx Reference
related:
  - NPM-Cheatsheet
  - PackageManagers-MOC
  - MOC-Infrastructure
---

# NX — Полная шпаргалка

> [!SUMMARY] Обзор
> Система сборки с продвинутыми возможностями для монорепозиториев. Кэширование, affected команды, распределённое выполнение.

---

## 📚 Теория

### Установка

```bash
# Создать новое Nx workspace
npx create-nx-workspace@latest my-org
npx create-nx-workspace@latest my-org --preset=ts        # TypeScript
npx create-nx-workspace@latest my-org --preset=react     # React
npx create-nx-workspace@latest my-org --preset=angular   # Angular
npx create-nx-workspace@latest my-org --preset=nest      # NestJS
npx create-nx-workspace@latest my-org --preset=express   # Express

# Добавить Nx в существующий проект
npm install -D nx
npx nx init
```

## Структура проекта

```
my-org/
├── apps/
│   ├── app-name/
│   │   ├── src/
│   │   ├── project.json
│   │   └── package.json
├── libs/
│   ├── lib-name/
│   │   ├── src/
│   │   ├── project.json
│   │   └── package.json
├── tools/
├── nx.json
├── package.json
└── tsconfig.base.json
```

## Основные концепции

### Конфигурация workspace

**nx.json** — глобальная конфигурация Nx:
```json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"]
      }
    }
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    }
  },
  "namedInputs": {
    "production": ["default", "!{projectRoot}/**/*.spec.ts"]
  }
}
```

**project.json** — конфигурация проекта:
```json
{
  "name": "my-app",
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "sourceRoot": "apps/my-app/src",
  "projectType": "application",
  "targets": {
    "build": {
      "executor": "@nx/webpack:webpack",
      "outputs": ["{options.outputPath}"],
      "options": {
        "outputPath": "dist/apps/my-app",
        "main": "apps/my-app/src/main.ts"
      }
    },
    "serve": {
      "executor": "@nx/webpack:dev-server",
      "options": {
        "port": 3000
      }
    },
    "test": {
      "executor": "@nx/jest:jest",
      "outputs": ["{workspaceRoot}/coverage/apps/my-app"]
    },
    "lint": {
      "executor": "@nx/eslint:lint"
    }
  }
}
```

## Команды

### Запуск задач
```bash
nx run <project>:<target>           # Запустить конкретную задачу
nx run <project>:<target>:<config>  # Запустить с конфигурацией
nx run-many --target=build          # Запустить для всех проектов
nx run-many --target=build --all    # То же самое
nx run-many --target=test --projects=app1,app2

# Короткая форма
nx build <project>                  # Запустить build
nx test <project>                   # Запустить test
nx serve <project>                  # Запустить serve
nx lint <project>                   # Запустить lint
```

### Affected команды (оптимизация для CI)
```bash
nx affected --target=build          # Запустить только для затронутых
nx affected --target=test           # Тестировать затронутые
nx affected --target=lint
nx affected:graph                   # Визуализировать затронутые
nx affected --base=main --head=HEAD # Сравнить ветки
```

### Граф зависимостей
```bash
nx graph                            # Открыть интерактивный граф
nx graph --file=graph.html          # Сохранить граф в файл
nx graph --watch                    # Режим наблюдения
```

### Генераторы (каркас)
```bash
# Приложения
nx generate @nx/node:application my-app
nx generate @nx/react:application my-app
nx generate @nx/angular:application my-app
nx generate @nx/nest:application my-app

# Библиотеки
nx generate @nx/node:library my-lib
nx generate @nx/react:library my-lib
nx generate @nx/angular:library my-lib

# Компоненты
nx generate @nx/react:component my-component --project=my-app
nx generate @nx/angular:component my-component

# Другое
nx generate @nx/node:service my-service
nx generate @nx/node:library my-lib --publishable --importPath=@my-org/my-lib
```

### Утилиты workspace
```bash
nx list                             # Показать установленные плагины
nx list @nx/react                   # Показать возможности плагина
nx migrate latest                   # Проверить миграции
nx migrate --run-migrations         # Запустить миграции
nx report                           # Отладочная информация
nx reset                            # Очистить кэш и сбросить
nx daemon --start                   # Запустить демон
```

## Конфигурация пайплайна задач

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],  # Сначала собрать зависимости
      "inputs": ["production", "^production"],
      "cache": true
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["default", "^production"],
      "cache": true
    },
    "lint": {
      "inputs": ["default"],
      "cache": true
    },
    "serve": {
      "dependsOn": ["build"]
    }
  }
}
```

## Кэширование и распределение

### Локальное кэширование
```bash
nx run build --skip-nx-cache    # Пропустить кэш для этого запуска
nx reset                        # Очистить локальный кэш
```

### Удалённое кэширование (Nx Cloud)
```bash
# Включить Nx Cloud
npx nx-cloud init

# Переменные окружения
NX_CLOUD_ACCESS_TOKEN=xxx
NX_CLOUD_DISTRIBUTED_EXECUTION=true
```

### Пользовательские входы кэша
```json
{
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.ts",
      "!{projectRoot}/tsconfig.spec.json"
    ],
    "sharedGlobals": ["{workspaceRoot}/babel.config.json"]
  }
}
```

## Module Federation

```bash
# Сгенерировать host
nx generate @nx/react:host apps/host --remote=remote1

# Сгенерировать remote
nx generate @nx/react:remote apps/remote1 --host=host
```

## Полезные плагины

| Плагин | Пакет | Описание |
|--------|-------|----------|
| React | `@nx/react` | Генераторы React приложений/библиотек |
| Angular | `@nx/angular` | Поддержка Angular |
| NestJS | `@nx/nest` | Генераторы NestJS |
| Node | `@nx/node` | Утилиты Node.js |
| Next.js | `@nx/next` | Поддержка Next.js |
| Vue | `@nx/vue` | Поддержка Vue.js |
| Webpack | `@nx/webpack` | Конфигурация Webpack |
| Jest | `@nx/jest` | Настройка Jest |
| ESLint | `@nx/eslint` | Линтинг |
| Docker | `@nx/docker` | Docker образы |
| Kubernetes | `@nx/kubernetes` | K8s манифесты |

## CI/CD Интеграция

### Пример GitHub Actions
```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci
      - run: npx nx affected --target=build --base=origin/main
      - run: npx nx affected --target=test --base=origin/main
```

## Паттерны монорепозиториев

### Типы библиотек
- **Shared**: Общие утилиты, типы, константы
- **UI**: Переиспользуемые компоненты
- **Data-access**: Управление состоянием, API вызовы
- **Feature**: Полные фичи (UI + логика)
- **Infrastructure**: Конфигурация, инструменты, утилиты

### Ограничения зависимостей
```json
// nx.json
{
  "dependencyConstraints": {
    "apps": {
      "onlyDependOnLibsWithTags": ["type:feature", "type:ui", "type:shared"]
    },
    "feature": {
      "onlyDependOnLibsWithTags": ["type:ui", "type:shared", "type:data-access"]
    }
  }
}
```

## Решение проблем

```bash
# Очистить кэш и пересобрать
nx reset && nx run-many --target=build

# Проверить конфигурацию проекта
nx show project <name>

# Подробный вывод
nx run build --verbose

# Проверить целостность workspace
nx graph

# Показать все задачи
nx show projects --json | jq
```

---

## 🔗 Связанные заметки

- [[NPM-Cheatsheet]] — пакетный менеджер
- [[PackageManagers-MOC]] — обзор менеджеров
- [[MOC-Infrastructure]] — инфраструктура
