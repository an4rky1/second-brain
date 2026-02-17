---
created: 2026-02-16
tags:
  - api
  - rest
  - http
aliases:
  - REST API Design
  - REST Best Practices
related:
  - API-Design
  - HTTP-Basics
  - NestJS-Cheatsheet
---

# REST API

> [!SUMMARY] Обзор
> REST (Representational State Transfer) — архитектурный стиль для распределённых систем. Основан на HTTP методах, stateless, кэшировании.

---

## 📚 Принципы REST

```
┌─────────────────────────────────────────────────────┐
│  REST Constraints                                    │
│                                                      │
│  1. Client-Server    → Разделение интерфейса        │
│  2. Stateless        → Нет состояния на сервере     │
│  3. Cacheable        → Кэшируемые ответы            │
│  4. Layered System   → Промежуточные серверы        │
│  5. Uniform Interface│  Единый интерфейс            │
│     • Resource identification (URI)                 │
│     • Resource manipulation (Representations)       │
│     • Self-descriptive messages                     │
│     • HATEOAS (опционально)                         │
└─────────────────────────────────────────────────────┘
```

---

## ⚡ HTTP Methods

| Method | Idempotent | Safe | Описание |
|--------|------------|------|----------|
| GET | ✅ | ✅ | Получение данных |
| POST | ❌ | ❌ | Создание ресурса |
| PUT | ✅ | ❌ | Полное обновление |
| PATCH | ❌ | ❌ | Частичное обновление |
| DELETE | ✅ | ❌ | Удаление ресурса |
| OPTIONS | ✅ | ✅ | Доступные методы |
| HEAD | ✅ | ✅ | Только заголовки |

---

## 🔧 Best Practices

### URI Design

```
✅ Правильные URI:
GET    /users              # Список пользователей
GET    /users/123          # Пользователь 123
GET    /users/123/posts    # Посты пользователя 123
POST   /users              # Создать пользователя
PUT    /users/123          # Обновить пользователя 123
PATCH  /users/123          # Частичное обновление
DELETE /users/123          # Удалить пользователя 123

❌ Неправильные URI:
GET    /getUsers           # Глаголы в URI
POST   /createUser
GET    /user?id=123        # ID в query, не в path
```

### Query Parameters

```
# Фильтрация
GET /users?role=admin&status=active

# Сортировка
GET /users?sort=-createdAt&sort=name

# Пагинация
GET /users?page=2&limit=20

# Поля (field selection)
GET /users?fields=id,name,email
```

### Status Codes

```
2xx — Success:
  200 OK              # Успешный запрос
  201 Created         # Ресурс создан
  204 No Content      # Успех без тела ответа

3xx — Redirection:
  301 Moved Permanently
  304 Not Modified    # Для кэширования

4xx — Client Error:
  400 Bad Request     # Неверный запрос
  401 Unauthorized    # Нет аутентификации
  403 Forbidden       # Нет доступа
  404 Not Found       # Ресурс не найден
  409 Conflict        # Конфликт состояния
  422 Unprocessable   # Ошибка валидации
  429 Too Many Requests

5xx — Server Error:
  500 Internal Server Error
  502 Bad Gateway
  503 Service Unavailable
```

### Response Format

```json
// Успешный ответ
{
  "data": {
    "id": "123",
    "name": "John",
    "email": "john@example.com"
  },
  "meta": {
    "timestamp": "2024-01-01T12:00:00Z"
  }
}

// Список с пагинацией
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  }
}

// Ошибка
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with id 123 not found",
    "details": {
      "id": "123"
    },
    "timestamp": "2024-01-01T12:00:00Z"
  }
}
```

### HATEOAS (Hypermedia)

```json
{
  "id": "123",
  "name": "John",
  "email": "john@example.com",
  "_links": {
    "self": {
      "href": "/users/123",
      "method": "GET"
    },
    "update": {
      "href": "/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/users/123",
      "method": "DELETE"
    },
    "posts": {
      "href": "/users/123/posts"
    }
  }
}
```

---

## 🔒 Security

### Headers

```
# Request
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json

# Response
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000
Access-Control-Allow-Origin: https://example.com
```

### Rate Limiting Headers

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1704067200
```

---

## 🎯 Versioning

```
# URI Versioning (наиболее распространённый)
GET /api/v1/users
GET /api/v2/users

# Header Versioning
GET /users
Accept: application/vnd.myapi.v1+json

# Query Versioning
GET /users?version=1
```

---

## 🔗 Связанные заметки

- [[GraphQL-API]] — GraphQL альтернатива
- [[gRPC-API]] — gRPC для микросервисов
- [[HTTP-Basics]] — HTTP основы
- [[NestJS-Cheatsheet]] — NestJS REST контроллеры

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Существительные в URI** — не глаголы
> 2. **Множественное число** — /users не /user
> 3. **Нижний регистр** — /users не /Users
> 4. **Дефис для слов** — /user-profiles не /user_profiles
> 5. **Фильтры в query** — не в path
