---
created: 2026-02-17
tags:
  - moc
  - testing
  - qa
aliases:
  - Testing MOC
  - QA Index
related:
  - MOC-Infrastructure
  - MOC-Patterns
---

# 🧪 Testing — Индекс

> [!SUMMARY] Обзор
> Тестирование приложений: unit, integration, E2E тесты. Инструменты, паттерны, лучшие практики.

---

## 🗂️ Навигация

| Тип | Файл | Описание |
|-----|------|----------|
| 📖 MOC | [[00-MOC-Testing]] | Главная страница |
| **Unit** | | |
| Unit Testing | [[Unit/Unit-Testing-Jest]] | Unit тесты с Jest |
| **Integration** | | |
| Integration Testing | [[Integration/Integration-Testing]] | Integration тесты |
| **E2E** | | |
| E2E Testing | [[E2E/E2E-Testing-Playwright]] | E2E тесты с Playwright |
| **Patterns** | | |
| Testing Patterns | [[Patterns/Testing-Patterns]] | Паттерны тестирования |

---

## 📊 Пирамида тестирования

```
        /\
       /  \
      / E2E \      ← Мало (10%)
     /--------\
    /  Integration \  ← Средне (20%)
   /----------------\
  /      Unit        \ ← Много (70%)
 /--------------------\
```

---

## 🔗 Связанные заметки

- [[MOC-Infrastructure]] — инфраструктура
- [[MOC-Patterns]] — паттерны
- [[../75-Performance/00-MOC/Performance-MOC]] — производительность
