---
created: 2026-02-17
tags:
  - nestjs
  - websockets
  - gateways
  - real-time
aliases:
  - NestJS WebSockets
  - NestJS Gateways
related:
  - NestJS-MOC
  - NestJS-Microservices
  - NestJS-Events
---

# NestJS — WebSockets

> [!SUMMARY] Обзор
> WebSockets в NestJS: Gateways, real-time communication, rooms, namespaces.

---

## 📦 Установка

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io
npm install -D @types/socket.io
```

---

## 🏛️ Basic Gateway

```typescript
// messages/messages.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({
  cors: {
    origin: ['https://example.com'],
  },
  namespace: 'messages',
})
export class MessagesGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
  }

  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: { message: string },
    @ConnectedSocket() client: Socket,
  ): void {
    // Broadcast to all
    this.server.emit('message', {
      message: data.message,
      clientId: client.id,
      timestamp: new Date().toISOString(),
    });

    // Or send to specific client
    client.emit('message-received', {
      message: 'Your message was received',
    });
  }
}
```

---

## 🏠 Rooms

```typescript
// chat/chat.gateway.ts
import { WebSocketGateway, SubscribeMessage } from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway()
export class ChatGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('join-room')
  handleJoinRoom(
    @MessageBody() data: { room: string },
    @ConnectedSocket() client: Socket,
  ) {
    client.join(data.room);
    console.log(`Client ${client.id} joined room ${data.room}`);
  }

  @SubscribeMessage('leave-room')
  handleLeaveRoom(
    @MessageBody() data: { room: string },
    @ConnectedSocket() client: Socket,
  ) {
    client.leave(data.room);
    console.log(`Client ${client.id} left room ${data.room}`);
  }

  @SubscribeMessage('chat-message')
  handleChatMessage(
    @MessageBody() data: { room: string; message: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Send to room
    this.server.to(data.room).emit('chat-message', {
      message: data.message,
      clientId: client.id,
      timestamp: new Date().toISOString(),
    });
  }

  @SubscribeMessage('broadcast')
  handleBroadcast(
    @MessageBody() data: { message: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Send to all except sender
    client.broadcast.emit('broadcast', {
      message: data.message,
      clientId: client.id,
    });
  }
}
```

---

## 🔐 Authentication Guard

```typescript
// auth/ws-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { WsException } from '@nestjs/websockets';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class WsAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const client = context.switchToWs().getClient();
    const token = client.handshake.auth.token;

    if (!token) {
      throw new WsException('No token provided');
    }

    try {
      const payload = await this.jwtService.verifyAsync(token);
      client.user = payload;
      return true;
    } catch (error) {
      throw new WsException('Invalid token');
    }
  }
}

// Usage
@WebSocketGateway()
export class AuthGateway {
  @UseGuards(WsAuthGuard)
  @SubscribeMessage('authenticated-event')
  handleAuth(@ConnectedSocket() client: Socket) {
    console.log(`Authenticated user: ${client.user.id}`);
  }
}
```

---

## 📡 Multiple Namespaces

```typescript
// notifications/notifications.gateway.ts
@WebSocketGateway({
  namespace: '/notifications',
  cors: {
    origin: ['https://example.com'],
  },
})
export class NotificationsGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('subscribe')
  handleSubscribe(
    @MessageBody() data: { userId: string },
    @ConnectedSocket() client: Socket,
  ) {
    client.join(`user:${data.userId}`);
  }

  sendNotification(userId: string, notification: any) {
    this.server.to(`user:${userId}`).emit('notification', notification);
  }
}

// chat/chat.gateway.ts
@WebSocketGateway({
  namespace: '/chat',
})
export class ChatGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('join-chat')
  handleJoinChat(@MessageBody() data: { chatId: string }) {
    // Join chat room
  }
}
```

---

## 🔄 Sending Events from Services

```typescript
// notifications/notifications.service.ts
import { Injectable } from '@nestjs/common';
import { NotificationsGateway } from './notifications.gateway';

@Injectable()
export class NotificationsService {
  constructor(private gateway: NotificationsGateway) {}

  async sendWelcomeEmail(userId: string, email: string) {
    // Send email...
    
    // Send notification
    this.gateway.sendNotification(userId, {
      type: 'welcome',
      message: 'Welcome to the platform!',
    });
  }

  async sendOrderUpdate(userId: string, orderId: string, status: string) {
    this.gateway.sendNotification(userId, {
      type: 'order-update',
      orderId,
      status,
    });
  }
}

// Gateway method
@WebSocketGateway({ namespace: '/notifications' })
export class NotificationsGateway {
  @WebSocketServer()
  server: Server;

  sendNotification(userId: string, notification: any) {
    this.server.to(`user:${userId}`).emit('notification', notification);
  }
}
```

---

## 🎯 Gateway Configuration

```typescript
@WebSocketGateway({
  // Port
  port: 3001,

  // Path
  path: '/ws',

  // CORS
  cors: {
    origin: ['https://example.com'],
    credentials: true,
  },

  // Namespace
  namespace: 'messages',

  // Transports
  transports: ['websocket', 'polling'],

  // Ping/Pong
  pingInterval: 25000,
  pingTimeout: 60000,

  // Max payload
  maxHttpBufferSize: 1e6,

  // Allow EIO
  allowEIO3: true,
})
export class MessagesGateway {}
```

---

## 📋 Environment

```bash
# .env
WS_PORT=3001
WS_PATH=/ws
WS_CORS_ORIGIN=https://example.com
WS_PING_INTERVAL=25000
WS_PING_TIMEOUT=60000
```

---

## 🔗 Связанные заметки

- [[NestJS-MOC]] — NestJS индекс
- [[NestJS-Microservices]] — Микросервисы
- [[NestJS-Events]] — Event emitter
- [[NestJS-Redis]] — Redis adapter
