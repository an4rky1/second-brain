---
created: 2026-02-16
tags:
  - message-broker
  - rabbitmq
  - kafka
  - queue
aliases:
  - Message Brokers
  - RabbitMQ & Kafka
related:
  - Microservices-Patterns
  - Docker-Templates
  - NestJS-Integrations
---

# Message Brokers

> [!SUMMARY] Обзор
> Брокеры сообщений для асинхронной коммуникации: RabbitMQ (queue-based), Kafka (event streaming).

---

## 📚 Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│  RabbitMQ vs Kafka                                               │
│                                                                  │
│  RabbitMQ                          Kafka                         │
│  ├─ Smart broker, dumb consumer    ├─ Dumb broker, smart consumer│
│  ├─ Queue-based                    ├─ Log-based (append-only)    │
│  ├─ Message consumed once          ├─ Message consumed multiple  │
│  ├─ Low latency (< ms)             ├─ High throughput            │
│  ├─ Complex routing                ├─ Simple partitioning        │
│  └─ Use: Task queues, RPC          └─ Use: Event sourcing, logs  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🐰 RabbitMQ

### Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│  Producer → Exchange → Queue → Consumer                         │
│                                                                  │
│  Exchange Types:                                                 │
│  ├─ Direct    → Route by exact match                            │
│  ├─ Fanout    → Broadcast to all queues                         │
│  ├─ Topic     → Route by pattern                                │
│  └─ Headers   → Route by headers                                │
└─────────────────────────────────────────────────────────────────┘
```

### Docker Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

### NestJS Implementation

```typescript
// rabbitmq.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'RABBITMQ_CLIENT',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://guest:guest@localhost:5672'],
          queue: 'tasks_queue',
          queueOptions: {
            durable: true,
          },
        },
      },
    ]),
  ],
})
export class RabbitMqModule {}

// tasks.producer.ts
@Injectable()
export class TasksProducer {
  constructor(
    @Inject('RABBITMQ_CLIENT')
    private client: ClientProxy,
  ) {}

  async sendTask(task: TaskDto) {
    return this.client.send('create_task', task);
  }

  async processTask(task: TaskDto) {
    return this.client.emit('task.process', task);
  }
}

// tasks.consumer.ts
@Controller()
export class TasksConsumer {
  @MessagePattern('create_task')
  async createTask(@Payload() task: TaskDto) {
    console.log('Received task:', task);
    // Process task
    return { success: true };
  }

  @EventPattern('task.process')
  async handleTaskProcess(@Payload() task: TaskDto) {
    console.log('Task processed:', task);
  }
}
```

### Patterns

```typescript
// Work Queues (Round Robin)
// Producer sends to single queue, multiple consumers

// Pub/Sub (Fanout Exchange)
// Producer → Fanout Exchange → Multiple Queues → Consumers

// Routing (Direct Exchange)
// Producer → Direct Exchange (routing key) → Queue

// Topic Exchange
channel.publish('', 'logs.*', Buffer.from(message));
// logs.info, logs.error, logs.warning

// RPC Pattern
const response = await firstValueFrom(
  client.send('calculate', { numbers: [1, 2, 3] })
);
```

---

## 🐘 Kafka

### Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│  Producer → Topic (Partitions) → Consumer Group                 │
│                                                                  │
│  Topic:      Category of messages (like DB table)               │
│  Partition:  Ordered, immutable sequence of messages            │
│  Offset:     Unique ID of message in partition                  │
│  Consumer:   Reads messages from partitions                     │
│  Group:      Multiple consumers share load                      │
└─────────────────────────────────────────────────────────────────┘
```

### Docker Setup (KRaft - no Zookeeper)

```yaml
# docker-compose.yml
version: '3.8'

services:
  kafka:
    image: apache/kafka:3.6.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT_HOST://localhost:9092,PLAINTEXT://kafka:29092
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT_HOST://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093,PLAINTEXT://0.0.0.0:29092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka_data:/var/kafka/data

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    depends_on:
      - kafka

volumes:
  kafka_data:
```

### NestJS Implementation

```typescript
// kafka.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'KAFKA_CLIENT',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'my-app',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'my-consumer-group',
          },
        },
      },
    ]),
  ],
})
export class KafkaModule {}

// events.producer.ts
@Injectable()
export class EventsProducer {
  constructor(
    @Inject('KAFKA_CLIENT')
    private client: ClientKafka,
  ) {}

  async emitEvent(event: EventDto) {
    return this.client.emit('user_created', event);
  }

  async sendMessage(topic: string, message: any) {
    return this.client.send(topic, message);
  }
}

// events.consumer.ts
@Controller()
export class EventsConsumer implements OnModuleInit {
  constructor(
    @Inject('KAFKA_CLIENT')
    private client: ClientKafka,
  ) {}

  async onModuleInit() {
    this.client.subscribeToResponseOf('user_created');
  }

  @MessagePattern('user_created')
  handleUserCreated(@Payload() event: EventDto) {
    console.log('User created:', event);
    // Process event
  }
}
```

### KafkaJS (Direct Usage)

```typescript
// kafka.service.ts
import { Kafka, Producer, Consumer, Admin } from 'kafkajs';

@Injectable()
export class KafkaService implements OnModuleInit, OnModuleDestroy {
  private kafka: Kafka;
  private producer: Producer;
  private consumer: Consumer;
  private admin: Admin;

  constructor() {
    this.kafka = new Kafka({
      clientId: 'my-app',
      brokers: ['localhost:9092'],
      retry: {
        initialRetryTime: 100,
        retries: 8,
      },
    });

    this.producer = this.kafka.producer();
    this.consumer = this.kafka.consumer({ groupId: 'my-group' });
    this.admin = this.kafka.admin();
  }

  async onModuleInit() {
    await this.producer.connect();
    await this.consumer.connect();
    
    // Create topic if not exists
    await this.admin.createTopics({
      topics: [{ topic: 'user-events' }],
    });
  }

  async onModuleDestroy() {
    await this.producer.disconnect();
    await this.consumer.disconnect();
  }

  async publishEvent(topic: string, event: any) {
    await this.producer.send({
      topic,
      messages: [{
        key: event.userId,
        value: JSON.stringify(event),
        headers: {
          eventType: event.type,
          timestamp: Date.now().toString(),
        },
      }],
    });
  }

  async subscribe(topic: string, handler: (event: any) => Promise<void>) {
    await this.consumer.subscribe({ topic, fromBeginning: false });

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const event = JSON.parse(message.value.toString());
        console.log(`Received from ${topic}[${partition}]:`, event);
        
        try {
          await handler(event);
          // Auto-commit after successful processing
        } catch (error) {
          console.error('Error processing message:', error);
          throw error;  // Will retry
        }
      },
    });
  }
}
```

---

## 🎯 Patterns

### Event Sourcing

```typescript
// Store events, not state
class OrderService {
  async createOrder(order: OrderDto) {
    // Emit events
    await this.eventBus.publish('OrderCreated', {
      orderId: order.id,
      userId: order.userId,
      total: order.total,
      timestamp: new Date(),
    });

    // Project to read model
    await this.readModel.orders.insert({
      id: order.id,
      status: 'created',
    });
  }

  async shipOrder(orderId: string) {
    await this.eventBus.publish('OrderShipped', {
      orderId,
      timestamp: new Date(),
    });

    await this.readModel.orders.update(orderId, {
      status: 'shipped',
    });
  }
}
```

### Saga Pattern

```typescript
// Distributed transaction
class OrderSaga {
  async processOrder(order: OrderDto) {
    try {
      // Step 1: Reserve inventory
      await this.inventoryService.reserve(order.items);
      
      // Step 2: Charge payment
      await this.paymentService.charge(order.userId, order.total);
      
      // Step 3: Create order
      const order = await this.orderService.create(order);
      
      // Step 4: Ship order
      await this.shippingService.ship(order.id);
      
    } catch (error) {
      // Compensating transactions
      await this.compensate(order);
      throw error;
    }
  }

  async compensate(order: OrderDto) {
    await this.inventoryService.release(order.items);
    await this.paymentService.refund(order.userId, order.total);
  }
}
```

### CQRS with Event Bus

```typescript
// Command side
class CreateUserHandler {
  async execute(command: CreateUserCommand) {
    const user = new User(command.email, command.password);
    await this.userRepository.save(user);
    
    // Publish events
    await this.eventBus.publish(new UserCreatedEvent(user.id));
  }
}

// Query side
class UserQueryHandler {
  @Query(() => User)
  async getUser(@Args('id') id: string) {
    return this.readModel.users.findById(id);
  }
}

// Event projector
class UserProjector {
  @OnEvent('user.created')
  async projectUserCreated(event: UserCreatedEvent) {
    await this.readModel.users.insert({
      id: event.userId,
      createdAt: event.timestamp,
    });
  }
}
```

---

## 🔗 Связанные заметки

- [[Microservices-Patterns]] — Микросервисы
- [[Docker-Templates]] — Docker для брокеров
- [[NestJS-Integrations]] — NestJS интеграции

---

## 📝 Заметки

> [!TIP] Выбор
> 
> - **RabbitMQ** — task queues, RPC, complex routing
> - **Kafka** — event sourcing, logs, high throughput
> - **Redis Streams** — simple queues, low latency

> [!INFO] Monitoring
> 
> - Queue depth
> - Consumer lag
> - Message rate
> - Error rate
