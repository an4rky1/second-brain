---
created: 2026-02-17
tags:
  - php
  - laravel
  - moc
  - index
aliases:
  - PHP MOC
  - PHP Index
related:
  - MOC-Languages
  - Laravel-Cheatsheet
---

# 🐘 PHP — Индекс

> [!SUMMARY] Обзор
> PHP — серверный язык для веб-разработки. Современный PHP (8.x) имеет строгую типизацию, JIT, атрибуты.

---

## 🗂️ Навигация

### 01 — Basics

| Тема | Файл | Описание |
|------|------|----------|
| 📖 Getting Started | [[01-Basics/PHP-Getting-Started]] | Установка, первый проект |
| 📝 Syntax | [[01-Basics/PHP-Syntax]] | Переменные, типы, функции |
| 🎯 OOP | [[01-Basics/PHP-OOP]] | Классы, наследование, трейты |

### 02 — Advanced

| Тема | Файл | Описание |
|------|------|----------|
| 🔧 Attributes | [[02-Advanced/PHP-Attributes]] | PHP 8 атрибуты |
| ⚡ JIT | [[02-Advanced/PHP-JIT]] | JIT компиляция |
| 📦 Composer | [[02-Advanced/PHP-Composer]] | Composer, autoloading |

### 03 — Laravel

| Тема | Файл | Описание |
|------|------|----------|
| 🚀 Getting Started | [[03-Laravel/Laravel-Getting-Started]] | Установка Laravel |
| 📦 Eloquent | [[03-Laravel/Laravel-Eloquent]] | Eloquent ORM |
| 🔐 Auth | [[03-Laravel/Laravel-Auth]] | Laravel Auth, Sanctum |

---

## 📚 Быстрый старт

```bash
# Установка PHP 8.x
# macOS: brew install php
# Linux: sudo apt install php php-cli php-mbstring
# Windows: choco install php

# Проверка версии
php -v

# Composer
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer

# Laravel
composer global require laravel/installer
laravel new myapp
cd myapp
php artisan serve
```

---

## 🔗 Связанные заметки

- [[MOC-Languages]] — Languages MOC
- [[Laravel-Cheatsheet]] — Laravel фреймворк
- [[Backend-TypeScript-Cheatsheet]] — Backend patterns
