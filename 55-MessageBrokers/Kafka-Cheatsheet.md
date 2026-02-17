---
created: 2026-02-17
tags:
  - message-broker
  - kafka
  - streaming
  - event-sourcing
aliases:
  - Kafka Cheatsheet
  - Apache Kafka Streaming
related:
  - Message-Brokers
  - RabbitMQ-Cheatsheet
  - NestJS-Microservices
  - Event-Sourcing
---

# Apache Kafka — Полная шпаргалка

> [!SUMMARY] Обзор
> Kafka — распределённая streaming платформа. Topics, partitions, producers, consumers, consumer groups, stream processing.

---

## 📦 Установка

```bash
# Docker Compose
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

---

## 🏗️ Архитектура

```
┌─────────────┐      ┌──────────────────────┐      ┌─────────────┐
│  Producer   │ ───► │      Topic           │ ───► │  Consumer   │
│             │      │  ┌────┬────┬────┐   │      │             │
│             │      │  │ P0 │ P1 │ P2 │   │      │             │
│             │      │  └────┴────┴────┘   │      │             │
│             │      │  (Partitions)       │      │             │
└─────────────┘      └──────────────────────┘      └─────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Consumer    │
                    │    Group      │
                    └───────────────┘
```

---

## 📝 Основные концепции

| Термин | Описание |
|--------|----------|
| **Topic** | Логический канал для сообщений |
| **Partition** | Физическое разделение topic (параллелизм) |
| **Offset** | Уникальный ID сообщения в partition |
| **Producer** | Отправляет сообщения в topic |
| **Consumer** | Читает сообщения из topic |
| **Consumer Group** | Группа consumers для параллельной обработки |
| **Broker** | Kafka сервер |
| **Zookeeper** | Координация кластера (в Kafka 3.x опционально) |

---

## 🎯 CLI Commands

```bash
# List topics
kafka-topics --bootstrap-server localhost:9092 --list

# Create topic
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic my-topic \
  --partitions 3 \
  --replication-factor 1

# Describe topic
kafka-topics --bootstrap-server localhost:9092 \
  --describe \
  --topic my-topic

# Delete topic
kafka-topics --bootstrap-server localhost:9092 \
  --delete \
  --topic my-topic

# Produce messages
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic my-topic

# Consume messages
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning

# Consume with key
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic my-topic \
  --property print.key=true \
  --property key.separator=":"

# Consumer groups
kafka-consumer-groups --bootstrap-server localhost:9092 --list
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group my-group \
  --describe

# Reset offsets
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group my-group \
  --topic my-topic \
  --reset-offsets \
  --to-beginning \
  --execute
```

---

## 📬 Producer API (Node.js)

```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092'],
  retry: {
    initialRetryTime: 100,
    retries: 8,
  },
});

const producer = kafka.producer({
  transactionTimeout: 60000,
});

await producer.connect();

// Send message
await producer.send({
  topic: 'my-topic',
  messages: [
    {
      key: 'user-123',
      value: JSON.stringify({ id: 123, name: 'John' }),
      headers: {
        correlationId: 'abc-123',
        timestamp: new Date().toISOString(),
      },
      partition: 0,  // Optional
    },
  ],
});

// Send to multiple topics
await producer.sendBatch({
  topicMessages: [
    {
      topic: 'topic-1',
      messages: [{ value: 'message 1' }],
    },
    {
      topic: 'topic-2',
      messages: [{ value: 'message 2' }],
    },
  ],
});

// Transactions
const transaction = await producer.transaction();
try {
  await transaction.send({
    topic: 'topic-1',
    messages: [{ value: 'message 1' }],
  });
  await transaction.send({
    topic: 'topic-2',
    messages: [{ value: 'message 2' }],
  });
  await transaction.commit();
} catch (error) {
  await transaction.abort();
}

await producer.disconnect();
```

---

## 📥 Consumer API (Node.js)

```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092'],
});

const consumer = kafka.consumer({
  groupId: 'my-group',
  maxWaitTimeInMs: 5000,
  sessionTimeout: 30000,
  heartbeatInterval: 10000,
});

await consumer.connect();

await consumer.subscribe({
  topic: 'my-topic',
  fromBeginning: false,  // Start from latest
});

await consumer.run({
  autoCommit: true,
  autoCommitInterval: 5000,
  autoCommitThreshold: 100,
  
  eachMessage: async ({ topic, partition, message }) => {
    console.log({
      topic,
      partition,
      offset: message.offset,
      key: message.key?.toString(),
      value: JSON.parse(message.value?.toString()),
      headers: message.headers,
    });
    
    // Manual commit if autoCommit: false
    // await consumer.commitOffsets([{ topic, partition, offset: message.offset }]);
  },
  
  // Or eachBatch for batch processing
  eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary }) => {
    for (const message of batch.messages) {
      await processMessage(message);
      resolveOffset(message.offset);
      await heartbeat();
    }
    commitOffsetsIfNecessary();
  },
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await consumer.disconnect();
});
```

---

## 🔄 Consumer Groups

```typescript
// Multiple consumers in same group = parallel processing
// Partition assigned to only ONE consumer per group

// Consumer 1: handles partitions 0, 1
// Consumer 2: handles partition 2

const consumer1 = kafka.consumer({ groupId: 'my-group' });
const consumer2 = kafka.consumer({ groupId: 'my-group' });

await consumer1.subscribe({ topic: 'my-topic' });
await consumer2.subscribe({ topic: 'my-topic' });

// Both consumers will receive messages
// Kafka automatically balances partitions
```

---

## 📊 Topic Configuration

```typescript
await admin.createTopics({
  topics: [
    {
      topic: 'my-topic',
      numPartitions: 6,
      replicationFactor: 3,
      configEntries: [
        { key: 'retention.ms', value: '604800000' },  // 7 days
        { key: 'segment.bytes', value: '1073741824' }, // 1GB
        { key: 'cleanup.policy', value: 'delete' },
        { key: 'min.insync.replicas', value: '2' },
        { key: 'max.message.bytes', value: '1048576' }, // 1MB
      ],
    },
    {
      topic: 'compact-topic',
      numPartitions: 3,
      configEntries: [
        { key: 'cleanup.policy', value: 'compact' },
        { key: 'min.compaction.lag.ms', value: '3600000' }, // 1 hour
      ],
    },
  ],
});
```

---

## 🏛️ Patterns

### Event Sourcing

```typescript
// Events topic (immutable)
await producer.send({
  topic: 'user-events',
  messages: [
    {
      key: 'user-123',
      value: JSON.stringify({
        type: 'USER_CREATED',
        aggregateId: 'user-123',
        timestamp: new Date().toISOString(),
        data: { email: 'john@example.com' },
      }),
    },
    {
      key: 'user-123',
      value: JSON.stringify({
        type: 'USER_EMAIL_UPDATED',
        aggregateId: 'user-123',
        timestamp: new Date().toISOString(),
        data: { email: 'new@example.com' },
      }),
    },
  ],
});

// Consumer rebuilds state from events
await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    await applyEvent(event);  // Apply to state
  },
});
```

### CQRS

```typescript
// Commands topic
await producer.send({
  topic: 'user-commands',
  messages: [{
    key: 'user-123',
    value: JSON.stringify({
      type: 'CREATE_USER',
      data: { email: 'john@example.com' },
    }),
  }],
});

// Events topic (for queries)
await producer.send({
  topic: 'user-events',
  messages: [{
    key: 'user-123',
    value: JSON.stringify({
      type: 'USER_CREATED',
      data: { id: 'user-123', email: 'john@example.com' },
    }),
  }],
});

// Query database (updated from events)
const user = await db.users.findOne({ id: 'user-123' });
```

### Saga Pattern

```typescript
// Order saga
await producer.send({
  topic: 'order-created',
  messages: [{
    key: orderId,
    value: JSON.stringify({ orderId, userId, amount }),
  }],
});

// Payment service consumes and publishes
await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());
    
    try {
      await processPayment(order);
      await producer.send({
        topic: 'payment-completed',
        messages: [{
          key: order.orderId,
          value: JSON.stringify({ orderId: order.orderId }),
        }],
      });
    } catch (error) {
      await producer.send({
        topic: 'payment-failed',
        messages: [{
          key: order.orderId,
          value: JSON.stringify({ orderId: order.orderId, reason: error.message }),
        }],
      });
    }
  },
});
```

---

## 📋 Environment

```bash
# .env
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=my-app

# Topics
TOPIC_USER_EVENTS=user-events
TOPIC_ORDER_EVENTS=order-events
TOPIC_NOTIFICATIONS=notifications

# Consumer
KAFKA_CONSUMER_GROUP=my-group
KAFKA_FROM_BEGINNING=false

# Producer
KAFKA_COMPRESSION_TYPE=gzip
KAFKA_IDEMPOTENT=true
```

---

## 🔗 Связанные заметки

- [[Message-Brokers]] — Message brokers overview
- [[RabbitMQ-Cheatsheet]] — RabbitMQ
- [[NestJS-Microservices]] — NestJS microservices
- [[Event-Sourcing]] — Event sourcing pattern
