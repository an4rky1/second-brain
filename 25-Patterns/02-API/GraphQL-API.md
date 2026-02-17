---
created: 2026-02-16
tags:
  - api
  - graphql
  - query
aliases:
  - GraphQL API Design
  - GraphQL Best Practices
related:
  - REST-API
  - tRPC-API
  - NestJS-Cheatsheet
---

# GraphQL API

> [!SUMMARY] Обзор
> GraphQL — язык запросов для API. Позволяет клиенту получать точно нужные данные. Единый endpoint, сильная типизация, introspection.

---

## 📚 Основы

```
┌─────────────────────────────────────────────────────┐
│  GraphQL vs REST                                     │
│                                                      │
│  REST                    │ GraphQL                   │
│  ├─ Множественные endpoints│ ├─ Единый endpoint     │
│  ├─ Over/under fetching  │ ├─ Точные данные        │
│  ├─Versioning (/v1/, /v2/)│ ├─ Schema evolution    │
│  └─ Простой кэшинг       │ └─ Сложный кэшинг       │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Schema Definition

```graphql
# Query — получение данных
type Query {
  user(id: ID!): User
  users(page: Int, limit: Int): UserConnection!
  post(id: ID!): Post
  search(query: String!, type: SearchType): SearchResult!
}

# Mutation — изменение данных
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User
  deleteUser(id: ID!): Boolean!
  createPost(input: CreatePostInput!): Post!
}

# Subscription — real-time обновления
type Subscription {
  userCreated: User!
  postAdded(authorId: ID): Post!
  orderStatusChanged(orderId: ID!): OrderStatus!
}

# Types
type User {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
  posts: [Post!]!
  profile: UserProfile
  createdAt: DateTime!
  updatedAt: DateTime!
}

type UserProfile {
  avatar: String
  bio: String
  age: Int
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  published: Boolean!
  tags: [String!]
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
  createdAt: DateTime!
}

# Enums
enum UserRole {
  ADMIN
  USER
  GUEST
}

enum SearchType {
  USER
  POST
  COMMENT
}

# Input types
input CreateUserInput {
  name: String!
  email: String!
  role: UserRole = USER
  profile: UserProfileInput
}

input UserProfileInput {
  avatar: String
  bio: String
  age: Int
}

input UpdateUserInput {
  name: String
  email: String
  role: UserRole
  profile: UserProfileInput
}

# Connection для пагинации
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Scalars
scalar DateTime
scalar JSON
scalar Email
```

---

## ⚡ Queries

```graphql
# Простой запрос
query GetUser {
  user(id: "123") {
    id
    name
    email
  }
}

# С вложенностями
query GetUserWithPosts {
  user(id: "123") {
    id
    name
    posts {
      id
      title
      comments {
        id
        text
        author {
          name
        }
      }
    }
  }
}

# С переменными
query GetUser($id: ID!, $includePosts: Boolean!) {
  user(id: $id) {
    id
    name
    posts @include(if: $includePosts) {
      id
      title
    }
  }
}

# Variables
{
  "id": "123",
  "includePosts": true
}

# Fragments
fragment UserFields on User {
  id
  name
  email
  role
  createdAt
}

query GetUsers {
  users {
    ...UserFields
    posts {
      id
      title
    }
  }
}

# Aliases
query GetTwoUsers {
  john: user(id: "1") {
    name
  }
  jane: user(id: "2") {
    name
  }
}
```

---

## 🔄 Mutations

```graphql
# Создание
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
  }
}

# Variables
{
  "input": {
    "name": "John",
    "email": "john@example.com",
    "role": "USER",
    "profile": {
      "bio": "Developer",
      "age": 30
    }
  }
}

# Обновление
mutation UpdateUser($id: ID!, $input: UpdateUserInput!) {
  updateUser(id: $id, input: $input) {
    id
    name
    email
    updatedAt
  }
}

# Удаление
mutation DeleteUser($id: ID!) {
  deleteUser(id: $id)
}
```

---

## 📡 Subscriptions

```graphql
# Простая подписка
subscription OnUserCreated {
  userCreated {
    id
    name
    email
  }
}

# Подписка с фильтрами
subscription OnOrderStatusChanged($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    orderId
    status
    updatedAt
  }
}

# Frontend usage (React Apollo)
import { useSubscription } from '@apollo/client';

function OrderTracker({ orderId }) {
  const { data, loading } = useSubscription(
    ON_ORDER_STATUS_CHANGED,
    { variables: { orderId } }
  );
  
  if (loading) return <Loading />;
  return <Status status={data.orderStatusChanged.status} />;
}
```

---

## 🎯 Best Practices

### ✅ Делать

```graphql
# 1. Обязательные поля с !
type User {
  id: ID!
  name: String!
  email: String!
}

# 2. Input types для мутаций
input CreateUserInput {
  name: String!
  email: String!
}

# 3. Connection для списков
type Query {
  users(first: Int, after: String): UserConnection!
}

# 4. Описания
"""
Represents a user in the system.
"""
type User {
  "Unique identifier"
  id: ID!
  
  "User's full name"
  name: String!
}

# 5. Error handling в mutation
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
}

type CreateUserPayload {
  user: User
  errors: [Error!]
}

type Error {
  field: String
  message: String!
}
```

### ❌ Не делать

```graphql
# 1. Избегать глубокой вложенности
type Query {
  user {
    posts {
      comments {
        author {
          posts {  # ❌ Бесконечная вложенность
            ...
          }
        }
      }
    }
  }
}

# 2. Не возвращать敏感 данные
type User {
  id: ID!
  name: String!
  # password: String  # ❌ Никогда!
}

# 3. Избегать больших списков без пагинации
type Query {
  users: [User!]!  # ❌ Без пагинации
  users(first: Int!, after: String): UserConnection!  # ✅
}
```

---

## 🔒 Security

### Query Depth Limiting

```typescript
// Apollo Server
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  schema,
  validationRules: [depthLimit(5)],
});
```

### Query Cost Analysis

```typescript
// Ограничение сложности запроса
const costAnalysis = {
  maximumCost: 1000,
  defaultCost: 1,
  variables: {},
  createError: (max, actual) => 
    new Error(`Query cost ${actual} exceeds maximum ${max}`),
};
```

### Rate Limiting

```typescript
// GraphQL Rate Limit
import rateLimit from 'graphql-rate-limit';

const resolvers = {
  Query: {
    users: rateLimit(
      () => 'global',
      { requests: 100, period: '1m' }
    ),
  },
};
```

---

## 🔗 Связанные заметки

- [[REST-API]] — REST альтернатива
- [[gRPC-API]] — gRPC для микросервисов
- [[tRPC-API]] — tRPC для TypeScript
- [[NestJS-Cheatsheet]] — NestJS GraphQL модуль

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Schema-first design** — начните со схемы
> 2. **Resolvers тонкие** — логика в сервисах
> 3. **DataLoader** — избегайте N+1 запросов
> 4. **Query complexity** — ограничивайте сложные запросы
> 5. **Introspection off** — в production
