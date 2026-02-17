---
created: 2026-02-16
tags:
  - api
  - grpc
  - microservices
  - protobuf
  - typescript
aliases:
  - gRPC API Design
  - gRPC Best Practices
  - gRPC TypeScript
related:
  - REST-API
  - tRPC-API
  - Microservices-Patterns
  - Docker-Cheatsheet
---

# gRPC API — Полная шпаргалка

> [!SUMMARY] Обзор
> gRPC — высокопроизводительный RPC фреймворк от Google. Использует HTTP/2, Protocol Buffers. Идеален для микросервисов, streaming, polyglot сред.

---

## 📚 Теория

### gRPC vs REST

```
┌─────────────────────────────────────────────────────────┐
│  REST                    │ gRPC                         │
├──────────────────────────┼──────────────────────────────┤
│  JSON/XML                │ Protocol Buffers (binary)    │
│  HTTP/1.1 или HTTP/2     │ HTTP/2 (обязательно)         │
│  Text-based              │ Binary (compact)             │
│  Browser support         │ Нужен gRPC-Web               │
│  100-1000ms latency      │ 10-100ms latency             │
│  Client → Server         │ Bidirectional streaming      │
└──────────────────────────┴──────────────────────────────┘
```

### Use Cases

```
✅ Микросервисы (service-to-service)
✅ Real-time streaming
✅ Mobile backends
✅ Polyglot environments
✅ Low-latency требования

❌ Public API (лучше REST/GraphQL)
❌ Browser clients (gRPC-Web сложнее)
❌ Prototyping (JSON проще)
```

### Архитектура

```
┌──────────────┐    HTTP/2    ┌──────────────┐
│   Client     │◄────────────►│   Server     │
│  (.ts/.go)   │   protobuf   │  (.ts/.go)   │
└──────────────┘              └──────────────┘
       │                            │
       ▼                            ▼
  .proto files              .proto files
       │                            │
       ▼                            ▼
  ts-proto                    ts-proto
  generated                   generated
```

---

## 🔧 Protocol Buffers

### Scalar Types

```protobuf
// Strings & Bytes
string
bytes

// Numbers
int32, int64          // Variable-length encoding
uint32, uint64        // Unsigned
sint32, sint64        // Signed (better for negatives)
fixed32, fixed64      // Fixed 64-bit
sfixed32, sfixed64

// Other
bool
float, double
```

### Message Definition

```protobuf
syntax = "proto3";

package users.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";

// ───────────────────────────────────────────────────────
// ENUMS
// ───────────────────────────────────────────────────────
enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;  // Всегда начинайте с 0
  USER_ROLE_ADMIN = 1;
  USER_ROLE_USER = 2;
  USER_ROLE_GUEST = 3;
}

// ───────────────────────────────────────────────────────
// NESTED MESSAGES
// ───────────────────────────────────────────────────────
message Profile {
  string avatar_url = 1;
  string bio = 2;
  int32 age = 3;
  
  enum Status {
    STATUS_UNSPECIFIED = 0;
    STATUS_ACTIVE = 1;
    STATUS_INACTIVE = 2;
  }
  
  Status status = 4;
}

// ───────────────────────────────────────────────────────
// MAIN MESSAGE
// ───────────────────────────────────────────────────────
message User {
  string id = 1;              // Всегда unique ID
  string name = 2;
  string email = 3;
  UserRole role = 4;
  repeated string tags = 5;   // Array
  Profile profile = 6;
  google.protobuf.Timestamp created_at = 7;
  
  // Oneof (только одно поле активно)
  oneof contact {
    string phone = 8;
    string telegram = 9;
    string discord = 10;
  }
  
  // Map
  map<string, string> metadata = 11;
  
  // Optional (proto3 optional)
  optional string middle_name = 12;
}

// ───────────────────────────────────────────────────────
// REQUEST/RESPONSE
// ───────────────────────────────────────────────────────
message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
  string filter = 3;
  repeated UserRole roles = 4;
  optional string sort_by = 5;
  optional bool descending = 6;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  bool has_more = 3;
  string next_page_token = 4;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  UserRole role = 3;
  optional Profile profile = 4;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;      // Только изменяемые поля
  optional string email = 3;
  optional UserRole role = 4;
  google.protobuf.FieldMask mask = 5;  // Какие поля обновлять
}

message DeleteUserRequest {
  string id = 1;
}

// Empty response
message DeleteUserResponse {}
// или используй google.protobuf.Empty
```

### Service Definition

```protobuf
service UserService {
  // ─────────────────────────────────────────────────────
  // UNARY: один запрос → один ответ
  // ─────────────────────────────────────────────────────
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  
  rpc CreateUser(CreateUserRequest) returns (User);
  
  rpc UpdateUser(UpdateUserRequest) returns (User);
  
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  
  // ─────────────────────────────────────────────────────
  // SERVER STREAMING: один запрос → поток ответов
  // ─────────────────────────────────────────────────────
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  rpc WatchUsers(WatchUsersRequest) returns (stream UserEvent);
  
  // ─────────────────────────────────────────────────────
  // CLIENT STREAMING: поток запросов → один ответ
  // ─────────────────────────────────────────────────────
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkResponse);
  
  // ─────────────────────────────────────────────────────
  // BIDIRECTIONAL: поток ↔ поток
  // ─────────────────────────────────────────────────────
  rpc SyncUsers(stream SyncRequest) returns (stream SyncResponse);
}

// Streaming messages
message WatchUsersRequest {
  repeated string user_ids = 1;
  repeated string events = 2;  // created, updated, deleted
}

message UserEvent {
  string type = 1;      // created, updated, deleted
  User user = 2;
  int64 timestamp = 3;
}

message BulkResponse {
  repeated string ids = 1;
  int32 success_count = 2;
  int32 failed_count = 3;
}

message SyncRequest {
  string user_id = 1;
  string action = 2;
  User user = 3;
}

message SyncResponse {
  string user_id = 1;
  bool success = 2;
  string error = 3;
}
```

### Well-Known Types

```protobuf
import "google/protobuf/timestamp.proto";    // Дата/время
import "google/protobuf/duration.proto";     // Длительность
import "google/protobuf/empty.proto";        // Пустой ответ
import "google/protobuf/any.proto";          // Любой тип
import "google/protobuf/struct.proto";       // JSON-подобный
import "google/protobuf/wrappers.proto";     // Nullable типы
import "google/protobuf/field_mask.proto";   // Partial updates

message Example {
  // Timestamp (ISO 8601)
  google.protobuf.Timestamp created_at = 1;
  google.protobuf.Timestamp updated_at = 2;
  
  // Duration
  google.protobuf.Duration timeout = 3;
  
  // Empty response
  rpc Delete(DeleteRequest) returns (google.protobuf.Empty);
  
  // Any (любой protobuf тип)
  google.protobuf.Any data = 4;
  
  // Struct (JSON объект)
  google.protobuf.Struct metadata = 5;
  
  // Wrappers (nullable)
  google.protobuf.StringValue optional_name = 6;
  google.protobuf.Int32Value optional_age = 7;
  google.protobuf.BoolValue is_active = 8;
}
```

---

## ⚡ TypeScript Интеграция

### Установка

```bash
# Основные пакеты
npm install @grpc/grpc-js @grpc/proto-loader

# Для генерации TypeScript типов
npm install -D ts-proto

# Альтернатива: grpc_tools_node_protoc
npm install -D grpc-tools @grpc/proto-loader
```

### Генерация TypeScript типов

```bash
# ───────────────────────────────────────────────────────
# Вариант 1: ts-proto (рекомендуется)
# ───────────────────────────────────────────────────────
# Простой, чистый код, хорошая типизация

npx protoc \
  --plugin=node_modules/.bin/protoc-gen-ts_proto \
  --ts_proto_out=./src/generated \
  --ts_proto_opt=esModuleInterop=true \
  --ts_proto_opt=outputServices=grpc-js \
  --proto_path=./proto \
  ./proto/users/v1/users.proto
```

### Package.json скрипты

```json
{
  "scripts": {
    "proto:generate": "npm run proto:generate:ts",
    
    "proto:generate:ts": "protoc \\\n      --plugin=node_modules/.bin/protoc-gen-ts_proto \\\n      --ts_proto_out=./src/generated \\\n      --ts_proto_opt=esModuleInterop=true,outputServices=grpc-js \\\n      --proto_path=./proto \\\n      ./proto/**/*.proto",
    
    "proto:generate:ts:watch": "nodemon --watch ./proto --ext proto --exec 'npm run proto:generate:ts'",
    
    "proto:lint": "protolint lint ./proto",
    
    "proto:format": "protolint fix ./proto",
    
    "proto:validate": "npm run proto:lint && npm run proto:generate:ts -- --dry-run",
    
    "proto:clean": "rm -rf src/generated && npm run proto:generate:ts"
  },
  "devDependencies": {
    "ts-proto": "^1.161.0",
    "protobufjs": "^7.2.0",
    "protolint": "^0.47.0",
    "nodemon": "^3.0.0"
  }
}
```

### Структура проекта

```
project/
├── proto/
│   ├── users/
│   │   └── v1/
│   │       ├── users.proto
│   │       └── users.service.ts (generated)
│   ├── common/
│   │   └── pagination.proto
│   └── buf.yaml
├── src/
│   ├── generated/              # Сгенерированные файлы
│   │   └── users/
│   │       └── v1/
│   │           ├── users.ts
│   │           └── users.service.ts
│   ├── grpc/
│   │   ├── clients/
│   │   │   └── users.client.ts
│   │   ├── servers/
│   │   │   └── users.server.ts
│   │   └── interceptors/
│   │       ├── auth.interceptor.ts
│   │       └── logging.interceptor.ts
│   └── index.ts
├── package.json
└── tsconfig.json
```

### TypeScript Client

```typescript
// src/grpc/clients/users.client.ts
import { credentials, ChannelCredentials } from '@grpc/grpc-js';
import { UserServiceClient } from '../../generated/users/v1/users.service';
import {
  GetUserRequest,
  ListUsersRequest,
  CreateUserRequest,
} from '../../generated/users/v1/users';

export class UsersGrpcClient {
  private client: UserServiceClient;

  constructor(address: string = 'localhost:50051') {
    this.client = new UserServiceClient(
      address,
      credentials.createInsecure()  // Для production: credentials.createSsl()
    );
  }

  // Unary
  async getUser(id: string): Promise<User> {
    return new Promise((resolve, reject) => {
      this.client.getUser(
        { id },
        (err, response) => {
          if (err) reject(err);
          else resolve(response);
        }
      );
    });
  }

  // Server Streaming
  async *listUsers(page: number, limit: number): AsyncGenerator<User> {
    const request: ListUsersRequest = { page, limit };
    const stream = this.client.listUsers(request);

    for await (const user of stream) {
      yield user;
    }
  }

  // Client Streaming
  async bulkCreate(users: CreateUserRequest[]): Promise<BulkResponse> {
    return new Promise((resolve, reject) => {
      const stream = this.client.bulkCreateUsers((err, response) => {
        if (err) reject(err);
        else resolve(response);
      });

      users.forEach(user => stream.write(user));
      stream.end();
    });
  }

  // Bidirectional Streaming
  async *syncUsers(requests: AsyncIterable<SyncRequest>): AsyncGenerator<SyncResponse> {
    const stream = this.client.syncUsers();

    // Send
    (async () => {
      for await (const request of requests) {
        stream.write(request);
      }
      stream.end();
    })();

    // Receive
    for await (const response of stream) {
      yield response;
    }
  }

  // Cleanup
  close(): void {
    this.client.close();
  }
}
```

### TypeScript Server

```typescript
// src/grpc/servers/users.server.ts
import { sendUnaryData, ServerUnaryCall, ServerStreamingCall } from '@grpc/grpc-js';
import { UserServiceHandlers } from '../../generated/users/v1/users.service';
import { UserService } from '../../services/users.service';

export function createUserServiceHandlers(userService: UserService): UserServiceHandlers {
  return {
    // Unary
    async getUser(
      call: ServerUnaryCall<GetUserRequest, User>,
      callback: sendUnaryData<User>
    ) {
      try {
        const user = await userService.findById(call.request.id);
        callback(null, user);
      } catch (error) {
        callback(error);
      }
    },

    // Server Streaming
    async listUsers(
      call: ServerUnaryCall<ListUsersRequest, User>,
      callback: sendUnaryData<User>
    ) {
      const users = await userService.findAll({
        page: call.request.page,
        limit: call.request.limit,
      });

      for (const user of users) {
        call.write(user);
      }
      call.end();
    },

    // Client Streaming
    async bulkCreateUsers(
      call: ServerStreamingCall<CreateUserRequest, BulkResponse>,
      callback: sendUnaryData<BulkResponse>
    ) {
      const users: CreateUserRequest[] = [];

      call.on('data', (user) => users.push(user));
      call.on('end', async () => {
        try {
          const result = await userService.bulkCreate(users);
          callback(null, result);
        } catch (error) {
          callback(error);
        }
      });
    },
  };
}
```

### Interceptors

```typescript
// src/grpc/interceptors/auth.interceptor.ts
import {
  InterceptingCall,
  Interceptor,
  InterceptorOptions,
  Listener,
} from '@grpc/grpc-js';

export const authInterceptor: Interceptor = (options, nextCall) => {
  return new InterceptingCall(nextCall(options), {
    start: function (metadata, listener, next) {
      const token = getTokenFromContext();
      if (token) {
        metadata.set('authorization', `Bearer ${token}`);
      }
      next(metadata, listener);
    },
  });
};

// Использование в клиенте
const client = new UserServiceClient(address, credentials.createInsecure(), {
  interceptors: [authInterceptor],
});
```

---

## 🎯 Best Practices

### ✅ Делать

```protobuf
// 1. Версионируйте API
package com.example.users.v1;

// 2. Используйте осмысленные имена
message GetUserRequest {}      // ✅
message Request {}             // ❌

// 3. Enum для статусов
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
}

// 4. Oneof для взаимоисключающих полей
message Payment {
  oneof method {
    string credit_card = 1;
    string paypal = 2;
  }
}

// 5. FieldMask для partial updates
message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  google.protobuf.FieldMask mask = 3;
}

// 6. Reserved для удалённых полей
message User {
  reserved 2, 5;
  reserved "old_field";
}

// 7. Deadlines в клиенте
client.getUser(request, { deadline: Date.now() + 5000 }, callback);
```

### ❌ Не делать

```protobuf
// 1. Не меняйте типы полей
int32 age = 1;      // Было
string age = 1;     // ❌ Нельзя менять

// 2. Не удаляйте поля без reserved
message User {
  string old_name = 2;  // ❌ Удалено без reserved
}

// 3. Не используйте negative numbers
int32 id = -1;  // ❌

// 4. Не меняйте номера полей
message User {
  string name = 1;
  string email = 3;  // ❌ Пропущен 2
}

// 5. Избегайте частых breaking changes
// Используйте backward compatible изменения
```

---

## 🔒 Error Handling

### gRPC Status Codes

```typescript
enum Status {
  OK = 0,
  CANCELLED = 1,
  UNKNOWN = 2,
  INVALID_ARGUMENT = 3,      // 400 Bad Request
  DEADLINE_EXCEEDED = 4,     // 408 Timeout
  NOT_FOUND = 5,             // 404 Not Found
  ALREADY_EXISTS = 6,        // 409 Conflict
  PERMISSION_DENIED = 7,     // 403 Forbidden
  UNAUTHENTICATED = 16,      // 401 Unauthorized
  INTERNAL = 13,             // 500 Internal Error
  UNAVAILABLE = 14,          // 503 Service Unavailable
}
```

### Custom Errors

```protobuf
// common/errors.proto
syntax = "proto3";

package common;

enum ErrorCode {
  ERROR_CODE_UNSPECIFIED = 0;
  ERROR_CODE_NOT_FOUND = 1;
  ERROR_CODE_INVALID_ARGUMENT = 2;
  ERROR_CODE_PERMISSION_DENIED = 3;
  ERROR_CODE_ALREADY_EXISTS = 4;
  ERROR_CODE_RATE_LIMIT_EXCEEDED = 5;
}

message ErrorDetail {
  string field = 1;
  string description = 2;
  string error_code = 3;
}

message ErrorResponse {
  ErrorCode code = 1;
  string message = 2;
  repeated ErrorDetail details = 3;
  string request_id = 4;  // Для tracing
}
```

### TypeScript Error Handling

```typescript
import { status, Metadata, ServerError } from '@grpc/grpc-js';

// Server
throw new ServerError(
  status.NOT_FOUND,
  'User not found',
  new Metadata(),
  [{ field: 'id', description: 'User does not exist' }]
);

// Client
try {
  const user = await client.getUser({ id });
} catch (error) {
  if (error.code === status.NOT_FOUND) {
    console.log('User not found');
  } else if (error.code === status.DEADLINE_EXCEEDED) {
    console.log('Request timeout');
  } else if (error.code === status.UNAVAILABLE) {
    console.log('Service unavailable');
  }
}
```

---

## 🐛 Отладка

### gRPCurl (CLI)

```bash
# Установить
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# List services
grpcurl -plaintext localhost:50051 list

# List methods
grpcurl -plaintext localhost:50051 list users.UserService

# Describe method
grpcurl -plaintext localhost:50051 describe users.UserService.GetUser

# Call method
grpcurl -plaintext \
  -d '{"id": "123"}' \
  localhost:50051 users.UserService/GetUser

# With headers
grpcurl -H "authorization: Bearer token" \
  -plaintext \
  -d '{"page": 1}' \
  localhost:50051 users.UserService/ListUsers
```

### BloomRPC (GUI)

```bash
# Скачать
# https://github.com/bloomrpc/bloomrpc

# Features:
# - Автозаполнение из .proto
# - Streaming support
# - Metadata editor
# - History
```

### Логирование

```typescript
// Interceptor для логирования
const loggingInterceptor: Interceptor = (options, nextCall) => {
  return new InterceptingCall(nextCall(options), {
    start: function (metadata, listener, next) {
      console.log(`GRPC: ${options.method_path}`);
      console.log(`Metadata: ${metadata.getMap()}`);
      next(metadata, {
        onMessage: function (message, next) {
          console.log('Response:', message);
          next(message);
        },
      });
    },
  });
};
```

---

## 🔗 Связанные заметки

- [[REST-API]] — REST альтернатива
- [[tRPC-API]] — tRPC для TypeScript
- [[GraphQL-API]] — GraphQL для сложных запросов
- [[Microservices-Patterns]] — Микросервисы
- [[Docker-Cheatsheet]] — Docker для gRPC сервисов

---

## 📝 Заметки

> [!TIP] Генерация TypeScript
>
> 1. **ts-proto** — рекомендуется (простой, чистый код)
> 2. **protobufjs** — альтернатива (больше возможностей)
> 3. **@grpc/proto-loader** — загрузка .proto файлов
>
> ```bash
> # Команда генерации
> npm run proto:generate
>
> # Watch режим
> npm run proto:generate:ts:watch
> ```

> [!INFO] Инструменты
> ```bash
> # Компиляция proto
> protoc --go_out=. --go-grpc_out=. proto/*.proto
>
> # TypeScript генерация
> npx protoc --plugin=node_modules/.bin/protoc-gen-ts_proto \
>   --ts_proto_out=./src/generated \
>   --ts_proto_opt=esModuleInterop=true,outputServices=grpc-js \
>   --proto_path=./proto ./proto/**/*.proto
>
> # gRPCurl (как curl для gRPC)
> grpcurl -plaintext localhost:50051 list
> grpcurl -plaintext localhost:50051 users.UserService/GetUser
>
> # BloomRPC (GUI клиент)
> # https://github.com/bloomrpc/bloomrpc
> ```

> [!WARNING] Breaking Changes
>
> - Не меняйте типы существующих полей
> - Не удаляйте поля без `reserved`
> - Не меняйте номера полей
> - Используйте версионирование: `package users.v1`
