---
created: 2026-02-17
tags:
  - message-broker
  - rabbitmq
  - amqp
  - queues
aliases:
  - RabbitMQ Cheatsheet
  - RabbitMQ AMQP
related:
  - Message-Brokers
  - Kafka-Cheatsheet
  - Redis-Cheatsheet
  - NestJS-Microservices
---

# RabbitMQ — Полная шпаргалка

> [!SUMMARY] Обзор
> RabbitMQ — брокер сообщений с поддержкой AMQP. Очереди, exchange, routing, pub/sub, delayed messages.

---

## 📦 Установка

```bash
# Docker
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management

# Management UI: http://localhost:15672
# Default: guest / guest
```

---

## 🏗️ Архитектура

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│  Producer   │ ───► │  Exchange    │ ───► │   Queue     │
│             │      │  (routing)   │      │             │
└─────────────┘      └──────────────┘      └──────┬──────┘
                                                  │
                                                  ▼
                                         ┌─────────────┐
                                         │  Consumer   │
                                         └─────────────┘
```

---

## 🔄 Exchange Types

### Direct Exchange

```typescript
// Routing by exact match
// Queue 1: routing_key = "error"
// Queue 2: routing_key = "warning"
// Producer: send to "error" → только Queue 1 получит

const amqp = require('amqplib');

async function directExample() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'direct_logs';
  await ch.assertExchange(exchange, 'direct', { durable: true });
  
  // Send
  const severity = process.argv[2] || 'info';
  const msg = process.argv[3] || 'Hello World!';
  
  ch.publish(exchange, severity, Buffer.from(msg));
  console.log(`Sent: ${severity}:${msg}`);
}
```

### Fanout Exchange

```typescript
// Broadcast to all queues
// Все очереди получают сообщение

async function fanoutExample() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'logs';
  await ch.assertExchange(exchange, 'fanout', { durable: true });
  
  // Send (routing key игнорируется)
  ch.publish(exchange, '', Buffer.from('Broadcast message'));
}
```

### Topic Exchange

```typescript
// Routing by pattern
// *.stock.# → queue получит все сообщения о stock
// *.ny.* → только NY сообщения

async function topicExample() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'topic_logs';
  await ch.assertExchange(exchange, 'topic', { durable: true });
  
  // Patterns:
  // * - одно слово
  // # - ноль или больше слов
  
  ch.publish(exchange, 'us.ny.stock', Buffer.from('NY Stock'));
  ch.publish(exchange, 'eu.london.stock', Buffer.from('London Stock'));
}
```

### Headers Exchange

```typescript
// Routing by headers
async function headersExample() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'headers_exchange';
  await ch.assertExchange(exchange, 'headers', { durable: true });
  
  // Send with headers
  ch.publish(exchange, '', Buffer.from('Message'), {
    headers: { type: 'important', priority: 'high' },
  });
}
```

---

## 📬 Patterns

### Work Queues (Task Queue)

```typescript
// Producer
async function workQueuesProducer() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const queue = 'task_queue';
  await ch.assertQueue(queue, { durable: true });
  
  // Send tasks
  for (let i = 0; i < 10; i++) {
    const msg = `Task ${i}`;
    ch.sendToQueue(queue, Buffer.from(msg), {
      persistent: true,  // Save to disk
    });
    console.log(`Sent: ${msg}`);
  }
}

// Consumer
async function workQueuesConsumer() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const queue = 'task_queue';
  await ch.assertQueue(queue, { durable: true });
  
  // Fair dispatch
  ch.prefetch(1);  // One task at a time
  
  ch.consume(queue, async (msg) => {
    console.log(`Received: ${msg.content.toString()}`);
    
    // Simulate work
    await new Promise(r => setTimeout(r, 1000));
    
    ch.ack(msg);  // Acknowledge
  });
}
```

### Pub/Sub

```typescript
// Producer
async function pubsubProducer() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'logs';
  await ch.assertExchange(exchange, 'fanout', { durable: true });
  
  ch.publish(exchange, '', Buffer.from('Broadcast!'));
}

// Consumer
async function pubsubConsumer() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'logs';
  await ch.assertExchange(exchange, 'fanout', { durable: true });
  
  // Temporary queue
  const q = await ch.assertQueue('', { exclusive: true });
  await ch.bindQueue(q.queue, exchange, '');
  
  ch.consume(q.queue, (msg) => {
    console.log(`Received: ${msg.content.toString()}`);
  });
}
```

### Routing

```typescript
// Producer
async function routingProducer(severity: string) {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'direct_logs';
  await ch.assertExchange(exchange, 'direct', { durable: true });
  
  ch.publish(exchange, severity, Buffer.from('Message'));
}

// Consumer
async function routingConsumer(severities: string[]) {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'direct_logs';
  await ch.assertExchange(exchange, 'direct', { durable: true });
  
  const q = await ch.assertQueue('', { exclusive: true });
  
  for (const severity of severities) {
    await ch.bindQueue(q.queue, exchange, severity);
  }
  
  ch.consume(q.queue, (msg) => {
    console.log(`Received: ${msg.content.toString()}`);
  });
}
```

### Topics

```typescript
// Producer
async function topicProducer(routingKey: string) {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'topic_logs';
  await ch.assertExchange(exchange, 'topic', { durable: true });
  
  ch.publish(exchange, routingKey, Buffer.from('Message'));
}

// Patterns:
// topicProducer('us.ny.stock')
// topicProducer('eu.london.stock')

// Consumer
async function topicConsumer(pattern: string) {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const exchange = 'topic_logs';
  await ch.assertExchange(exchange, 'topic', { durable: true });
  
  const q = await ch.assertQueue('', { exclusive: true });
  await ch.bindQueue(q.queue, exchange, pattern);
  
  // Patterns:
  // '*.ny.*' → все NY
  // 'us.*' → все US
  // '#' → все сообщения
  
  ch.consume(q.queue, (msg) => {
    console.log(`Received: ${msg.content.toString()}`);
  });
}
```

---

## ⏱️ Delayed Messages

```typescript
// Method 1: TTL + Dead Letter Exchange
async function delayedMessages() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const dlx = 'delayed_exchange';
  const dlq = 'delayed_queue';
  
  // Dead Letter Queue
  await ch.assertQueue(dlq);
  await ch.assertExchange(dlx, 'direct', { durable: true });
  await ch.bindQueue(dlq, dlx, 'delayed');
  
  // Delay queue with TTL
  const delayQueue = 'delay_queue';
  await ch.assertQueue(delayQueue, {
    messageTtl: 5000,  // 5 seconds
    deadLetterExchange: dlx,
    deadLetterRoutingKey: 'delayed',
  });
  
  // Send delayed message
  ch.sendToQueue(delayQueue, Buffer.from('Delayed message'), {
    persistent: true,
  });
  
  // Consume from DLQ
  ch.consume(dlq, (msg) => {
    console.log(`Received after delay: ${msg.content.toString()}`);
  });
}

// Method 2: RabbitMQ Delayed Message Plugin
// rabbitmq_delayed_message_exchange
```

---

## 🔒 Reliability

### Publisher Confirms

```typescript
async function publisherConfirms() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  await ch.confirmSelect();
  
  const queue = 'confirm_queue';
  await ch.assertQueue(queue);
  
  ch.sendToQueue(queue, Buffer.from('Message'), (err, ok) => {
    if (err) {
      console.error('Message not confirmed!');
    } else {
      console.log('Message confirmed!');
    }
  });
}
```

### Consumer Acknowledgments

```typescript
async function consumerAcks() {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();
  
  const queue = 'ack_queue';
  await ch.assertQueue(queue, { durable: true });
  
  ch.prefetch(1);
  
  ch.consume(queue, async (msg) => {
    try {
      // Process message
      await processMessage(msg);
      ch.ack(msg);  // Acknowledge success
    } catch (error) {
      ch.nack(msg, false, true);  // Requeue on error
      // Or: ch.reject(msg, true);
    }
  });
}
```

### Message Persistence

```typescript
// Durable queue + persistent message
ch.assertQueue('persistent_queue', { durable: true });

ch.sendToQueue('persistent_queue', Buffer.from('Message'), {
  persistent: true,  // Save to disk
});
```

---

## 🎯 NestJS Integration

```bash
npm install @nestjs/microservices amqplib amqp-connection-manager
```

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'RABBITMQ_CLIENT',
        transport: Transport.RMQ,
        options: {
          urls: ['amqp://localhost:5672'],
          queue: 'my_queue',
          queueOptions: {
            durable: true,
          },
        },
      },
    ]),
  ],
})
export class AppModule {}
```

```typescript
// Controller
@Controller('messages')
export class MessagesController {
  @MessagePattern('message.created')
  handleMessageCreated(data: any) {
    console.log('Received:', data);
  }

  @EventPattern('message.deleted')
  handleMessageDeleted(data: any) {
    console.log('Deleted:', data);
  }
}
```

---

## 📋 Environment

```bash
# .env
RABBITMQ_URL=amqp://localhost:5672
RABBITMQ_MANAGEMENT_URL=http://localhost:15672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest

# Queues
TASK_QUEUE=task_queue
EMAIL_QUEUE=email_queue
NOTIFICATION_QUEUE=notification_queue

# Exchanges
DIRECT_EXCHANGE=direct_logs
FANOUT_EXCHANGE=logs
TOPIC_EXCHANGE=topic_logs
```

---

## 🔗 Связанные заметки

- [[Message-Brokers]] — Message brokers overview
- [[Kafka-Cheatsheet]] — Kafka streaming
- [[Redis-Cheatsheet]] — Redis pub/sub
- [[NestJS-Microservices]] — NestJS microservices
