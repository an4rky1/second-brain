---
created: 2026-02-16
tags:
  - cheat-sheet
  - yarn
  - package-manager
  - nodejs
aliases:
  - Yarn Cheatsheet
  - Yarn Reference
related:
  - NPM-Cheatsheet
  - PNPM-Cheatsheet
  - NX-Cheatsheet
  - PackageManagers-MOC
---

# Yarn — Полная шпаргалка

> [!SUMMARY] Обзор
> Быстрый и надёжный пакетный менеджер для Node.js от Facebook. Workspaces, оффлайн-режим, Plug'n'Play.

---

## 📚 Теория

### Установка

```bash
# Установить Yarn глобально
npm install -g yarn

# Проверить версию
yarn --version

# Установить версию (для согласованности в команде)
yarn set version stable
yarn set version berry  # Yarn v2+
```

## Версии Yarn

| Версия | Кодовое имя | Ключевые особенности |
|--------|-------------|---------------------|
| v1.x | Classic | Workspaces, оффлайн-режим |
| v2.x | Berry | PnP, zero-installs, плагины |
| v3.x+ | Berry | Улучшенный PnP, совместимость |

## Основные команды

### Установка
```bash
yarn install                    # Установить все зависимости
yarn add <package>              # Добавить в dependencies
yarn add -D <package>           # Добавить в devDependencies
yarn add -P <package>           # Добавить в peerDependencies
yarn add -O <package>           # Добавить в optionalDependencies
yarn add <package> @version     # Добавить конкретную версию
yarn add <package> --exact      # Зафиксировать точную версию
yarn install --frozen-lockfile  # CI: ошибка если lockfile изменился
yarn install --immutable        # Аналогично frozen-lockfile
```

### Управление зависимостями
```bash
yarn remove <package>           # Удалить пакет
yarn upgrade                    # Обновить все пакеты
yarn upgrade <package>          # Обновить конкретный пакет
yarn upgrade-interactive          # Интерактивное обновление
yarn outdated                   # Проверить устаревшие пакеты
yarn list                       # Показать установленные пакеты
yarn list --depth=0             # Только верхний уровень
yarn why <package>              # Почему пакет установлен?
```

### Запуск скриптов
```bash
yarn run <script>               # Запустить скрипт из package.json
yarn <script>                   # Короткая форма
yarn start                      # Запустить скрипт 'start'
yarn test                       # Запустить скрипт 'test'
yarn build -- --flag            # Передать аргументы
```

### Кэш и очистка
```bash
yarn cache clean                # Очистить кэш
yarn cache dir                  # Показать расположение кэша
yarn cache list                 # Показать содержимое кэша
yarn cache clean <package>      # Очистить конкретный пакет
```

## Yarn Workspaces (Монорепозиторий)

### Настройка (package.json)
```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ],
  "workspaces": {
    "packages": ["packages/*", "apps/*"],
    "nohoist": ["**/react-native"]
  }
}
```

### Команды workspace
```bash
yarn workspace <name> add <pkg>    # Добавить в конкретный workspace
yarn workspace <name> run build    # Запустить скрипт в workspace
yarn workspaces info               # Показать информацию о workspace
yarn workspaces run <script>       # Запустить скрипт во всех workspace
yarn workspaces foreach -A <cmd>   # Выполнить во всех workspace (v2+)
```

### Пример
```bash
yarn workspace @my-org/ui add react
yarn workspace @my-org/api run build
yarn workspaces run test
yarn workspaces foreach -A --parallel run build
```

## Yarn Berry (v2+)

### Конфигурация .yarnrc.yml
```yaml
nodeLinker: node-modules        # Использовать node_modules (не PnP)
# nodeLinker: pnp-unplugged     # Использовать Plug'n'Play

yarnPath: .yarn/releases/yarn-3.6.0.cjs  # Зафиксировать версию Yarn

npmRegistryServer: "https://registry.npmjs.org"

plugins:
  - path: .yarn/plugins/@yarnpkg/plugin-workspace-tools.cjs
    spec: "@yarnpkg/plugin-workspace-tools"

enableGlobalCache: true
cacheFolder: .yarn/cache
```

### Zero-Installs
```bash
# Включить zero-installs (коммитить кэш в git)
yarn config set enableImmutableInstalls true

# .gitignore (без zero-installs)
.yarn/cache/*
!.yarn/cache/.gitignore

# .gitignore (с zero-installs)
# Не игнорировать .yarn/cache
```

### Plug'n'Play (PnP)
```yaml
# .yarnrc.yml
nodeLinker: pnp
```

Преимущества:
- Быстрая установка
- Меньше места на диске
- Нет node_modules
- Лучшая безопасность зависимостей

## Полезные команды

```bash
# Информация о пакете
yarn info <package>              # Показать информацию о пакете
yarn info <package> versions     # Показать доступные версии

# Бинарники
yarn bin                         # Показать папку bin
yarn exec <command>              # Выполнить в окружении

# Управление версиями
yarn version                     # Обновить версию
yarn version --major
yarn version --minor
yarn version --patch

# Публикация
yarn publish                     # Опубликовать пакет
yarn npm login                   # Войти в npm
yarn npm whoami                  # Показать текущего пользователя
yarn npm logout                  # Выйти

# Плагины (Berry)
yarn plugin import <name>        # Импортировать плагин
yarn plugin list                 # Показать плагины
```

## Миграция с npm

```bash
# Удалить lock-файл npm
rm package-lock.json

# Установить через Yarn
yarn install

# Обновить скрипты для использования yarn
# (большинство команд совместимы)
```

## Частые проблемы

```bash
# Очистить всё и переустановить
rm -rf node_modules .yarn/cache
yarn cache clean
yarn install

# Исправить проблемы PnP
yarn dlx @yarnpkg/sdks           # Сгенерировать SDK файлы

# Проверить целостность
yarn check --integrity
```

---

## 🔗 Связанные заметки

- [[NPM-Cheatsheet]] — стандартный менеджер
- [[PNPM-Cheatsheet]] — эффективный менеджер
- [[NX-Cheatsheet]] — монорепозиторий
- [[PackageManagers-MOC]] — обзор всех менеджеров
