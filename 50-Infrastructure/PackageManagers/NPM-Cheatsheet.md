---
created: 2026-02-16
tags:
  - cheat-sheet
  - npm
  - package-manager
  - nodejs
aliases:
  - NPM Cheatsheet
  - NPM Reference
related:
  - Yarn-Cheatsheet
  - PNPM-Cheatsheet
  - NX-Cheatsheet
  - PackageManagers-MOC
---

# NPM — Полная шпаргалка

> [!SUMMARY] Обзор
> Стандартный пакетный менеджер для Node.js. Установка зависимостей, управление версиями, публикация пакетов.

---

## 📚 Теория

### Package.json
```json
{ "name": "my-awesome-package" }
```
- Имя проекта/пакета (lowercase, без пробелов)
- Должно быть уникальным для публикации в npm

#### `version`
```json
{ "version": "1.0.0" }
```
- Версия в формате Semantic Versioning (`MAJOR.MINOR.PATCH`)
- Обязательно для публикации

---

### Структура package.json

```json
{
  "name": "@myorg/my-package",
  "version": "1.0.0",
  "description": "Описание проекта",
  "type": "module",
  "private": false,
  
  "main": "dist/index.js",
  "module": "dist/index.mjs",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    }
  },
  "bin": {
    "my-cli": "./bin/cli.js"
  },
  "files": ["dist", "README.md", "LICENSE"],
  
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "test": "jest",
    "lint": "eslint ."
  },
  
  "repository": {
    "type": "git",
    "url": "https://github.com/user/repo.git"
  },
  "homepage": "https://myproject.com",
  "bugs": { "url": "https://github.com/user/repo/issues" },
  "funding": "https://opencollective.com/myproject",
  
  "keywords": ["amazing", "package", "nodejs"],
  "author": "John Doe <john@example.com>",
  "license": "MIT",
  
  "engines": { "node": ">=18.0.0", "npm": ">=9.0.0" },
  
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "~4.17.21"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "jest": "^29.0.0"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  },
  "optionalDependencies": {
    "fsevents": "^2.3.0"
  },
  "overrides": {
    "colors": "1.4.0"
  },
  
  "workspaces": ["packages/*"]
}
```

---

### Поля package.json — расшифровка

| Поле | Описание | Пример |
|------|----------|--------|
| `name` | Имя пакета | `"my-package"` |
| `version` | Версия (SemVer) | `"1.0.0"` |
| `description` | Краткое описание | `"Amazing package"` |
| `keywords` | Ключевые слова для поиска | `["api", "rest"]` |
| `license` | Лицензия | `"MIT"` |
| `author` | Автор проекта | `"John Doe <john@example.com>"` |
| `contributors` | Список участников | `[{ "name": "Jane" }]` |
| `main` | Точка входа (CommonJS) | `"dist/index.js"` |
| `module` | Точка входа (ESM) | `"dist/index.mjs"` |
| `types` | TypeScript определения | `"dist/index.d.ts"` |
| `exports` | Явные экспорты | `{ ".": "./index.js" }` |
| `bin` | CLI команды | `{ "cli": "./bin/cli.js" }` |
| `files` | Файлы для публикации | `["dist", "README.md"]` |
| `scripts` | Команды запуска | `{ "build": "tsc" }` |
| `repository` | Ссылка на репозиторий | `{ "type": "git", "url": "..." }` |
| `homepage` | Домашняя страница | `"https://myproject.com"` |
| `bugs` | Где сообщать о багах | `{ "url": "..." }` |
| `funding` | Поддержка проекта | `"https://patreon.com/..."` |
| `engines` | Требуемые версии | `{ "node": ">=18" }` |
| `os` / `cpu` | Поддерживаемые ОС/архитектуры | `["linux", "x64"]` |
| `type` | Тип модуля | `"module"` или `"commonjs"` |
| `private` | Запрет публикации | `true` |
| `workspaces` | Монорепозиторий | `["packages/*"]` |
| `sideEffects` | Для tree-shaking | `false` |

---

### Зависимости — типы и версии

#### Типы зависимостей

| Тип | Описание | Команда |
|-----|----------|---------|
| `dependencies` | Для работы приложения | `npm install <pkg>` |
| `devDependencies` | Только для разработки | `npm install -D <pkg>` |
| `peerDependencies` | Требуются у потребителя | Вручную в package.json |
| `optionalDependencies` | Не обязательные | `npm install -O <pkg>` |
| `bundledDependencies` | Включаются в пакет | Вручную в package.json |

#### Версионирование (Semantic Versioning)

| Символ | Значение | Пример |
|--------|----------|---------|
| `^` | Совместимая версия (minor/patch) | `^1.2.3` → `>=1.2.3 <2.0.0` |
| `~` | Приблизительно (только patch) | `~1.2.3` → `>=1.2.3 <1.3.0` |
| `>=` | Больше или равно | `>=1.2.3` |
| `<` | Меньше | `<2.0.0` |
| `*` | Любая версия | `*` |
| `latest` | Последняя версия | `latest` |

#### Переопределение версий

```json
{
  "overrides": {
    "colors": "1.4.0"
  }
}
```
- **npm v8.3+**: Принудительная версия даже для транзитивных зависимостей

---

## Скрипты

### Основные скрипты

```json
{
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "build:watch": "tsc --watch",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/ --ext .ts",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "clean": "rm -rf dist/"
  }
}
```

### Lifecycle события

| Событие | Когда выполняется |
|---------|-------------------|
| `preinstall` | Перед установкой |
| `postinstall` | После установки пакета |
| `prepublishOnly` | Перед `npm publish` |
| `prepare` | Перед упаковкой и публикацией |
| `preversion` | Перед изменением версии |
| `version` | После изменения версии |
| `postversion` | После изменения версии + git push |
| `pretest` / `test` / `posttest` | Тесты |
| `prestart` / `start` / `poststart` | Запуск |
| `prebuild` / `build` / `postbuild` | Сборка |

### Pre/Post скрипты

```json
{
  "scripts": {
    "prebuild": "npm run clean",
    "build": "tsc",
    "postbuild": "echo 'Build complete!'"
  }
}
```
- Выполняются автоматически до/после основного скрипта

---

## Основные команды NPM

### Установка

```bash
npm install                    # Установить все зависимости
npm install <package>          # Установить пакет (в dependencies)
npm install -D <package>       # Установить в devDependencies
npm install -g <package>       # Установить глобально
npm install <package>@version  # Конкретную версию
npm ci                         # Чистая установка (для CI/CD)
npm i                          # Короткая форма
```

### Управление зависимостями

```bash
npm uninstall <package>        # Удалить пакет
npm update                     # Обновить все пакеты
npm update <package>           # Обновить конкретный
npm outdated                   # Проверить устаревшие
npm ls                         # Показать установленные
npm ls --depth=0               # Только верхний уровень
```

### Скрипты

```bash
npm run <script>               # Запустить скрипт
npm start                      # Запустить 'start'
npm test                       # Запустить 'test'
npm run build -- --flag        # Передать аргументы
```

### Кэш и очистка

```bash
npm cache clean --force        # Очистить кэш
npm cache verify               # Проверить кэш
npm prune                      # Удалить лишние пакеты
npm rebuild                    # Пересобрать node_modules
```

### Публикация

```bash
npm login                      # Войти в npm registry
npm publish                    # Опубликовать пакет
npm publish --access public    # Опубликовать публично
npm unpublish <pkg>@version    # Удалить версию
npm version <type>             # Обновить версию (major/minor/patch)
```

### Реестр и конфиг

```bash
npm config list                # Показать конфиг
npm config set registry <url>  # Установить реестр
npm whoami                     # Показать пользователя
npm logout                     # Выйти
```

### Работа с package.json

```bash
npm init -y                    # Создать package.json
npm pkg get name               # Получить поле
npm pkg set description="..."  # Установить поле
npm pkg delete keywords        # Удалить поле
```

---

## Полезные библиотеки

### Для разработки

| Библиотека | Описание |
|------------|----------|
| `nodemon` | Автоперезапуск при изменениях |
| `typescript` | Компилятор TypeScript |
| `ts-node` | Выполнение TypeScript напрямую |
| `eslint` | Линтинг кода |
| `prettier` | Форматирование кода |
| `husky` | Git hooks |
| `lint-staged` | Линтинг staged файлов |
| `jest` / `vitest` | Тестирование |
| `@types/node` | TypeScript определения |

### Для продакшена

| Библиотека | Описание |
|------------|----------|
| `express` / `fastify` | Веб-фреймворки |
| `axios` / `node-fetch` | HTTP клиент |
| `lodash` / `ramda` | Утилиты |
| `winston` / `pino` | Логирование |
| `dotenv` | Переменные окружения |
| `zod` / `yup` | Валидация |
| `prisma` / `typeorm` | ORM |
| `redis` | Redis клиент |
| `jsonwebtoken` | JWT |

---

## NPM Workspaces (Монорепозиторий)

### Настройка

```json
{
  "name": "monorepo",
  "private": true,
  "workspaces": ["packages/*"]
}
```

### Команды

```bash
npm install --workspace=<pkg>  # Установить в workspace
npm run build --workspace=<pkg> # Запустить в workspace
npm ls --workspace=<pkg>       # Показать зависимости
```

---

## package-lock.json

- **Назначение**: Фиксирует точные версии зависимостей
- **Коммитить**: Всегда в систему контроля версий
- **Пересоздать**: `rm package-lock.json && npm install`

---

## 🐛 Решение проблем

```bash
# Очистить всё и переустановить
rm -rf node_modules package-lock.json
npm cache clean --force
npm install

# Исправить права (Linux/Mac)
sudo chown -R $(whoami) ~/.npm

# Проверить peer-зависимости
npm ls

# Отладка
npm install --verbose
```

---

## 🔗 Связанные заметки

- [[Yarn-Cheatsheet]] — альтернативный пакетный менеджер
- [[PNPM-Cheatsheet]] — эффективный менеджер
- [[NX-Cheatsheet]] — монорепозиторий
- [[PackageManagers-MOC]] — обзор всех менеджеров
