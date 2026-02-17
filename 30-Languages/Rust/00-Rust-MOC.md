---
created: 2026-02-17
tags:
  - rust
  - systems
  - moc
  - index
aliases:
  - Rust MOC
  - Rust Index
related:
  - MOC-Languages
  - Backend-TypeScript-Cheatsheet
---

# 🦀 Rust — Индекс

> [!SUMMARY] Обзор
> Rust — системный язык с гарантиями безопасности памяти. Без сборщика мусора, идеален для systems programming, WASM.

---

## 🗂️ Навигация

### 01 — Basics

| Тема | Файл | Описание |
|------|------|----------|
| 📖 Getting Started | [[01-Basics/Rust-Getting-Started]] | Установка, cargo, первый проект |
| 📝 Syntax | [[01-Basics/Rust-Syntax]] | Переменные, типы, функции |
| 🎯 Ownership | [[01-Basics/Rust-Ownership]] | Ownership, borrowing, lifetimes |

### 02 — Advanced

| Тема | Файл | Описание |
|------|------|----------|
| 🎭 Traits | [[02-Advanced/Rust-Traits]] | Traits, trait objects |
| ⚡ Generics | [[02-Advanced/Rust-Generics]] | Generics, const generics |
| 📦 Crates | [[02-Advanced/Rust-Crates]] | Cargo, crates.io, workspaces |

### 03 — Systems

| Тема | Файл | Описание |
|------|------|----------|
| 🔀 Concurrency | [[03-Systems/Rust-Concurrency]] | Threads, channels, async |
| 🛡️ Unsafe | [[03-Systems/Rust-Unsafe]] | Unsafe Rust, FFI |
| 🌐 WASM | [[03-Systems/Rust-WASM]] | WebAssembly, wasm-pack |

---

## 📚 Быстрый старт

```bash
# Установка
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Проверка версии
rustc --version
cargo --version

# Создание проекта
cargo new myapp
cd myapp

# Запуск
cargo run

# Сборка
cargo build --release

# Тесты
cargo test

# Форматирование
cargo fmt

# Lint
cargo clippy
```

---

## 🔗 Связанные заметки

- [[MOC-Languages]] — Languages MOC
- [[Backend-TypeScript-Cheatsheet]] — Backend patterns
- [[WebAssembly-Basics]] — WASM
