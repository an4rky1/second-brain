---
created: 2026-02-16
tags:
  - websocket
  - real-time
  - socket
aliases:
  - WebSocket Patterns
  - Real-time Communication
related:
  - REST-API
  - Socket.IO
  - NestJS-Cheatsheet
---

# WebSocket & Real-time

> [!SUMMARY] Обзор
> WebSocket — протокол для двусторонней связи в реальном времени. Используется для чатов, уведомлений, collaborative editing, live данных.

---

## 📚 Основы

```
┌─────────────────────────────────────────────────────┐
│  WebSocket vs HTTP                                   │
│                                                      │
│  HTTP:                                               │
│  ├─ Request-Response (клиент инициирует)            │
│  ├─ Stateless                                         │
│  └─ Polling для real-time                           │
│                                                      │
│  WebSocket:                                          │
│  ├─ Full-duplex (двусторонняя связь)                │
│  ├─ Persistent connection                            │
│  └─ Server push                                      │
└─────────────────────────────────────────────────────┘

Use Cases:
├── Чат приложения
├── Real-time уведомления
├── Collaborative editing
├── Live sports/stocks
└── Multiplayer games
```

---

## 🔧 WebSocket API

### Client-side

```typescript
// Basic WebSocket
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  console.log('Connected');
  ws.send(JSON.stringify({ type: 'message', data: 'Hello' }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};

ws.onerror = (error) => {
  console.error('Error:', error);
};

ws.onclose = () => {
  console.log('Disconnected');
};

// With reconnection
class WebSocketClient {
  private ws: WebSocket | null = null;
  private reconnectInterval = 1000;
  private reconnectAttempts = 0;

  constructor(private url: string) {}

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectAttempts = 0;
    };

    this.ws.onclose = () => {
      console.log('Disconnected');
      this.reconnect();
    };

    this.ws.onerror = (error) => {
      console.error('Error:', error);
    };
  }

  private reconnect() {
    if (this.reconnectAttempts < 5) {
      setTimeout(() => {
        this.reconnectAttempts++;
        this.connect();
      }, this.reconnectInterval * this.reconnectAttempts);
    }
  }

  send(data: any) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }

  close() {
    this.ws?.close();
  }
}
```

---

## 🕸️ Socket.IO

### Server (NestJS)

```typescript
// events/events.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
  MessageBody,
  ConnectedSocket,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({
  cors: {
    origin: 'http://localhost:3000',
    credentials: true,
  },
  namespace: 'events',
})
export class EventsGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  handleConnection(client: Socket) {
    console.log('Client connected:', client.id);
  }

  handleDisconnect(client: Socket) {
    console.log('Client disconnected:', client.id);
  }

  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: { room: string; message: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Отправить в комнату
    client.to(data.room).emit('message', {
      user: client.id,
      message: data.message,
    });
  }

  @SubscribeMessage('joinRoom')
  handleJoinRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(room);
    client.emit('joined', { room });
    
    // Уведомить других
    client.to(room).emit('userJoined', {
      userId: client.id,
      room,
    });
  }

  @SubscribeMessage('leaveRoom')
  handleLeaveRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.leave(room);
  }

  // Broadcast всем
  broadcast(event: string, data: any) {
    this.server.emit(event, data);
  }

  // Broadcast в комнату
  broadcastToRoom(room: string, event: string, data: any) {
    this.server.to(room).emit(event, data);
  }

  // Отправить конкретному клиенту
  sendToClient(clientId: string, event: string, data: any) {
    this.server.to(clientId).emit(event, data);
  }
}

// Отправка из сервиса
@Injectable()
export class NotificationsService {
  constructor(@Inject(EventsGateway) private gateway: EventsGateway) {}

  notifyUser(userId: string, message: string) {
    this.gateway.sendToClient(`user:${userId}`, 'notification', {
      message,
      timestamp: new Date(),
    });
  }

  notifyRoom(room: string, event: string, data: any) {
    this.gateway.broadcastToRoom(room, event, data);
  }
}
```

### Client (React)

```typescript
// hooks/useSocket.ts
import { useEffect, useState } from 'react';
import { io, Socket } from 'socket.io-client';

export function useSocket(namespace: string) {
  const [socket, setSocket] = useState<Socket | null>(null);

  useEffect(() => {
    const newSocket = io(`http://localhost:3000${namespace}`, {
      withCredentials: true,
    });
    setSocket(newSocket);

    return () => {
      newSocket.close();
    };
  }, [namespace]);

  return socket;
}

// ChatRoom component
function ChatRoom({ roomId }: { roomId: string }) {
  const socket = useSocket('/events');
  const [messages, setMessages] = useState([]);
  const [users, setUsers] = useState([]);

  useEffect(() => {
    if (!socket) return;

    // Join room
    socket.emit('joinRoom', roomId);

    // Listen for messages
    socket.on('message', (msg) => {
      setMessages(prev => [...prev, msg]);
    });

    // Listen for user events
    socket.on('userJoined', ({ userId }) => {
      setUsers(prev => [...prev, userId]);
    });

    socket.on('userLeft', ({ userId }) => {
      setUsers(prev => prev.filter(id => id !== userId));
    });

    return () => {
      socket.off('message');
      socket.off('userJoined');
      socket.off('userLeft');
    };
  }, [socket, roomId]);

  const sendMessage = (message: string) => {
    socket?.emit('message', { room: roomId, message });
  };

  return (
    <div>
      <div>Users: {users.length}</div>
      {messages.map((msg, i) => (
        <div key={i}>{msg.message}</div>
      ))}
      <button onClick={() => sendMessage('Hello!')}>Send</button>
    </div>
  );
}
```

---

## 📡 Patterns

### Request-Response over WebSocket

```typescript
// Client
class WSClient {
  private callbacks = new Map();
  private messageId = 0;

  sendRequest(type: string, data: any): Promise<any> {
    return new Promise((resolve, reject) => {
      const id = this.messageId++;
      
      this.callbacks.set(id, { resolve, reject });
      
      this.socket.send(JSON.stringify({
        id,
        type: 'request',
        messageType: type,
        data,
      }));

      setTimeout(() => {
        this.callbacks.delete(id);
        reject(new Error('Timeout'));
      }, 5000);
    });
  }

  handleMessage(message: any) {
    if (message.type === 'response') {
      const callback = this.callbacks.get(message.id);
      if (callback) {
        if (message.error) {
          callback.reject(new Error(message.error));
        } else {
          callback.resolve(message.data);
        }
        this.callbacks.delete(message.id);
      }
    }
  }
}

// Server
ws.on('message', (data) => {
  const message = JSON.parse(data);
  
  if (message.type === 'request') {
    handleRequest(message).then(result => {
      ws.send(JSON.stringify({
        id: message.id,
        type: 'response',
        data: result,
      }));
    }).catch(error => {
      ws.send(JSON.stringify({
        id: message.id,
        type: 'response',
        error: error.message,
      }));
    });
  }
});
```

### Presence System

```typescript
// Server
@WebSocketGateway()
export class PresenceGateway implements OnGatewayConnection, OnGatewayDisconnect {
  private onlineUsers = new Map<string, Set<string>>();

  handleConnection(client: Socket) {
    const userId = this.getUserId(client);
    
    if (!this.onlineUsers.has(userId)) {
      this.onlineUsers.set(userId, new Set());
    }
    this.onlineUsers.get(userId)!.add(client.id);

    client.broadcast.emit('userStatus', {
      userId,
      status: 'online',
    });
  }

  handleDisconnect(client: Socket) {
    const userId = this.getUserId(client);
    const sockets = this.onlineUsers.get(userId);
    
    if (sockets) {
      sockets.delete(client.id);
      
      if (sockets.size === 0) {
        this.onlineUsers.delete(userId);
        client.broadcast.emit('userStatus', {
          userId,
          status: 'offline',
        });
      }
    }
  }

  isUserOnline(userId: string): boolean {
    return this.onlineUsers.has(userId);
  }
}
```

### Rate Limiting

```typescript
// Server-side rate limiting
const rateLimits = new Map<string, { count: number; resetTime: number }>();

function checkRateLimit(clientId: string, limit: number, window: number): boolean {
  const now = Date.now();
  const client = rateLimits.get(clientId) || { count: 0, resetTime: now + window };

  if (now > client.resetTime) {
    client.count = 0;
    client.resetTime = now + window;
  }

  client.count++;
  rateLimits.set(clientId, client);

  return client.count <= limit;
}

@SubscribeMessage('message')
@UseGuards(new WsRateLimitGuard(10, 60000))  // 10 messages per minute
handleMessage(@MessageBody() data: any, @ConnectedSocket() client: Socket) {
  // Handle message
}
```

---

## 🔗 Связанные заметки

- [[REST-API]] — REST для сравнения
- [[GraphQL-API]] — GraphQL Subscriptions
- [[NestJS-Cheatsheet]] — NestJS WebSocket gateway

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Reconnection logic** — всегда на клиенте
> 2. **Heartbeat/Ping-Pong** — для keep-alive
> 3. **Rooms** — для групповой коммуникации
> 4. **Rate limiting** — защита от abuse
> 5. **Cleanup** — отписывайтесь от событий
