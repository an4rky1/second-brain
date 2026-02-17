# 🧠 Second Brain v11 — База знаний Fullstack-разработчика

> **Персональная коллекция паттернов, шпаргалок и архитектурных решений**

---

## 📁 Структура

```
second-brain-v11/
├── 00-Inbox/              # Входящие — быстрые заметки
├── 00-Style-Guide.md      # Стандарты оформления
├── 10-Templates/          # Шаблоны заметок
├── 20-MOC/                # Карты контента (главная навигация)
│   ├── 00-Home.md
│   ├── MOC-Backend.md
│   ├── MOC-Languages.md
│   ├── MOC-Frameworks.md
│   ├── MOC-Infrastructure.md
│   ├── MOC-CICD.md
│   ├── MOC-Security.md
│   ├── MOC-Databases.md
│   └── MOC-Linux.md
├── 25-Patterns/           # Паттерны и архитектура
│   ├── 00-MOC-Patterns.md
│   ├── 01-Architecture/   # Архитектурные паттерны
│   ├── 02-API/            # API паттерны
│   ├── 03-Design/         # Design patterns
│   ├── 04-Testing/        # Testing patterns
│   └── 05-Performance/    # Performance patterns
├── 30-Languages/          # Языки программирования
│   ├── TypeScript/
│   ├── Go/
│   ├── PHP/
│   ├── Rust/
│   └── Bash/
├── 40-Frameworks/         # Фреймворки
│   ├── React/
│   ├── Next.js/
│   ├── Vue3/
│   ├── Angular/
│   ├── NestJS/
│   └── Laravel/
├── 50-Infrastructure/     # Инфраструктура
│   ├── Docker/
│   ├── Kubernetes/
│   ├── Terraform/
│   ├── Monitoring/
│   └── PackageManagers/
├── 55-MessageBrokers/     # Брокеры сообщений
├── 60-CICD/               # CI/CD
│   ├── 00-MOC/
│   ├── GitHub-Actions/
│   ├── GitLab-CI/
│   └── ArgoCD/
├── 65-Testing/            # Тестирование
│   ├── 00-MOC/
│   ├── Unit/
│   ├── Integration/
│   ├── E2E/
│   └── Patterns/
├── 70-Security/           # Безопасность
│   ├── 00-MOC/
│   ├── Auth/
│   ├── OWASP/
│   └── Hardening/
├── 75-Performance/        # Производительность
│   ├── 00-MOC/
│   ├── Caching/
│   ├── Logging/
│   ├── Error-Handling/
│   └── Optimization/
├── 80-Databases/          # Базы данных
│   ├── SQL/
│   └── NoSQL/
├── 90-Linux/              # Linux
└── 99-Archives/           # Архив
```

---

## 🚀 Быстрый старт

### Для Obsidian

1. Откройте папку `second-brain-v11` в Obsidian как хранилище
2. Установите плагины (рекомендуемые):
   - **Dataview** — запросы к заметкам
   - **Templater** — шаблоны
   - **QuickAdd** — быстрые действия
   - **OmniSearch** — поиск
   - **Various Complements** — автодополнение

### Настройка Templater

```javascript
// Templater настройки
Templates folder: 10-Templates/
Date format: YYYY-MM-DD
Time format: HH:mm
```

---

## 📊 Содержимое

### Языки (30-Languages)

| Язык | Заметки |
|------|---------|
| TypeScript | [[TypeScript-MOC]] |
| Go | [[Go-Cheatsheet]] |
| PHP | [[PHP-Cheatsheet]] |
| Rust | [[Rust-Cheatsheet]] |
| Bash | [[Bash-Cheatsheet]] |

### Фреймворки (40-Frameworks)

| Фреймворк | Заметки |
|-----------|---------|
| React | [[React-MOC]] |
| Next.js | [[NextJS-Cheatsheet]] |
| Vue 3 | [[Vue3-Cheatsheet]] |
| Angular | [[Angular-Cheatsheet]] |
| NestJS | [[NestJS-Cheatsheet]] |
| Laravel | [[Laravel-Cheatsheet]] |

### Инфраструктура (50-Infrastructure)

| Инструмент | Заметки |
|------------|---------|
| Docker | [[Docker-Cheatsheet]] |
| Kubernetes | [[Kubernetes-Cheatsheet]] |
| Terraform | [[Terraform-Cheatsheet]] |
| Monitoring | [[Monitoring-Cheatsheet]] |

### Базы данных (80-Databases)

| Тип | Заметки |
|-----|---------|
| SQL | [[PostgreSQL-Cheatsheet]] |
| NoSQL | [[NoSQL-Cheatsheet]], [[Redis-Cheatsheet]] |

### Паттерны (25-Patterns)

| Область | Заметки |
|---------|---------|
| Architecture | [[25-Patterns/00-MOC-Patterns]], [[25-Patterns/01-Architecture/Backend-Architecture-MOC]] |
| API | [[25-Patterns/02-API/REST-API]], [[25-Patterns/02-API/GraphQL-API]] |
| Testing | [[65-Testing/00-MOC/00-MOC-Testing]] |
| Performance | [[75-Performance/00-MOC/00-MOC-Performance]] |

### Остальное

| Область | Заметки |
|---------|---------|
| CI/CD | [[60-CICD/00-MOC/00-MOC-CICD]], [[60-CICD/00-MOC/CI-CD-Pipeline]] |
| Security | [[70-Security/00-MOC/00-MOC-Security]], [[70-Security/Auth/Auth-Patterns]] |
| Testing | [[65-Testing/00-MOC/00-MOC-Testing]] |
| Performance | [[75-Performance/00-MOC/00-MOC-Performance]] |
| Linux | [[90-Linux/Linux-Commands]] |

---

## 🔖 Система тегов

### По типу контента
- `#cheat-sheet` — шпаргалка
- `#concept` — концепция
- `#pattern` — паттерн
- `#commands` — команды
- `#troubleshooting` — решение проблем
- `#best-practices` — лучшие практики
- `#moc` — карта контента

### По технологиям
- `#typescript` `#go` `#php` `#rust` `#bash`
- `#react` `#nextjs` `#vue` `#angular` `#nestjs` `#laravel`
- `#docker` `#kubernetes` `#terraform`
- `#postgresql` `#mysql` `#mongodb` `#redis`

### По темам
- `#auth` `#performance` `#testing` `#debugging`
- `#architecture` `#design-patterns` `#security`
- `#cicd` `#devops` `#monitoring`

---

## 🔗 Cross-linking

### Примеры связей

```
TypeScript ─┬─→ React ──→ Next.js
            ├─→ NestJS ──→ Docker
            └─→ Angular

Go ─┬─→ Docker ──→ Kubernetes
    └─→ Microservices

PHP ─→ Laravel ──→ Docker

Rust ─┬─→ WebAssembly
      └─→ System Programming

Bash ─┬─→ Linux
      └─→ CI/CD
```

---

## 📝 Как использовать

### 1. Поиск информации

- Используйте **Quick Switcher** (`Ctrl/Cmd + O`)
- Поиск по тегам: `tag:#docker`
- Поиск по тексту: `content:"docker run"`

### 2. Добавление новых заметок

1. Создайте заметку в `00-Inbox/`
2. Используйте шаблон из `10-Templates/`
3. Добавьте теги и связи
4. Переместите в соответствующую папку

### 3. Обновление существующих

- Добавляйте примеры из практики
- Обновляйте секцию troubleshooting
- Связывайте с новыми заметками

---

## 🎯 Best Practices

### Для заметок

- ✅ Один файл = одна тема
- ✅ Используйте шаблоны
- ✅ Добавляйте теги
- ✅ Создавайте cross-links
- ✅ Пишите примеры кода
- ✅ Добавляйте troubleshooting

### Для структуры

- ✅ Следуйте иерархии папок
- ✅ Используйте MOC для навигации
- ✅ Регулярно чистите Inbox
- ✅ Архивируйте устаревшее в `99-Archives/`

---

## 📈 Статистика

| Раздел | Файлов | Статус |
|--------|--------|--------|
| Templates | 5 | ✅ Готово |
| MOC | 9 | ✅ Готово |
| Patterns | 10+ | ✅ Готово |
| Languages | 5 | ✅ Готово |
| Frameworks | 6 | ✅ Готово |
| Infrastructure | 4 | ✅ Готово |
| CI/CD | 1 | ✅ Готово |
| Security | 2 | ✅ Готово |
| Databases | 3 | ✅ Готово |
| Linux | 1 | ✅ Готово |
| **Итого** | **50+** | **🟢 Активно** |

---

## 📚 Рекомендации

### Книги

- «Clean Code» — Robert Martin
- «The Pragmatic Programmer» — Hunt & Thomas
- «Designing Data-Intensive Applications» — Martin Kleppmann
- «Site Reliability Engineering» — Google

### Ресурсы

- [DevDocs](https://devdocs.io/) — документация
- [DevHints](https://devhints.io/) — шпаргалки
- [Roadmap.sh](https://roadmap.sh/) — пути развития

---

*Последнее обновление: 2026-02-17*
