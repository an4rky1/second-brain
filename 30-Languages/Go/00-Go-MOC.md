---
created: 2026-02-17
tags:
  - go
  - golang
  - moc
  - index
aliases:
  - Go MOC
  - Golang Index
related:
  - MOC-Languages
  - Backend-TypeScript-Cheatsheet
---

# 🐹 Go — Индекс

> [!SUMMARY] Обзор
> Go (Golang) — компилируемый язык от Google. Простота, производительность, встроенная конкурентность.

---

## 🗂️ Навигация

### 01 — Basics

| Тема | Файл | Описание |
|------|------|----------|
| 📖 Getting Started | [[01-Basics/Go-Getting-Started]] | Установка, первый проект |
| 📝 Syntax | [[01-Basics/Go-Syntax]] | Переменные, типы, функции |
| 🔄 Control Flow | [[01-Basics/Go-Control-Flow]] | if, for, switch |

### 02 — Advanced

| Тема | Файл | Описание |
|------|------|----------|
| 🎭 Interfaces | [[02-Advanced/Go-Interfaces]] | Interfaces, type assertions |
| ⚡ Generics | [[02-Advanced/Go-Generics]] | Generics (Go 1.18+) |
| 📦 Modules | [[02-Advanced/Go-Modules]] | Go modules, dependencies |

### 03 — Concurrency

| Тема | Файл | Описание |
|------|------|----------|
| 🧵 Goroutines | [[03-Concurrency/Go-Goroutines]] | Goroutines, goroutine pools |
| 📬 Channels | [[03-Concurrency/Go-Channels]] | Channels, select |
| 🔀 Sync | [[03-Concurrency/Go-Sync]] | WaitGroup, Mutex, atomic |

---

## 📚 Быстрый старт

```bash
# Установка
# macOS: brew install go
# Linux: sudo apt install golang
# Windows: choco install golang

# Проверка версии
go version

# Создание проекта
mkdir myapp && cd myapp
go mod init myapp

# Запуск
go run main.go

# Сборка
go build -o myapp

# Тесты
go test ./...
```

---

## 🔗 Связанные заметки

- [[MOC-Languages]] — Languages MOC
- [[Backend-TypeScript-Cheatsheet]] — Backend patterns
- [[NestJS-MOC]] — NestJS (alternative backend)
