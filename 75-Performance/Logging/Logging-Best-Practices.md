---
created: 2026-02-16
tags:
  - logging
  - observability
  - monitoring
aliases:
  - Logging Best Practices
  - Structured Logging
related:
  - Monitoring-Cheatsheet
  - Security-Hardening
  - Docker-Cheatsheet
---

# Logging Best Practices

> [!SUMMARY] Обзор
> Структурированное логирование, уровни логов, correlation ID, агрегация логов (ELK, Loki).

---

## 📚 Log Levels

```
┌─────────────────────────────────────────────────────────────────┐
│  Log Levels (by severity)                                        │
│                                                                  │
│  ERROR    →  Something failed, needs attention                  │
│  WARN     →  Something unexpected, but handled                  │
│  INFO     →  Normal operation, business events                  │
│  DEBUG    →  Detailed technical information                     │
│  TRACE    →  Very detailed, for debugging                       │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Each Level

```typescript
// ERROR - Application error
logger.error('Database connection failed', {
  error: err,
  host: config.db.host,
});

// WARN - Unexpected but handled
logger.warn('Rate limit exceeded', {
  userId: user.id,
  limit: 100,
});

// INFO - Business events
logger.info('User created', {
  userId: user.id,
  email: user.email,
});

logger.info('Order completed', {
  orderId: order.id,
  total: order.total,
});

// DEBUG - Technical details
logger.debug('Query executed', {
  query: sql,
  params,
  duration: 42,
});

// TRACE - Very detailed
logger.trace('Request headers', {
  headers: req.headers,
});
```

---

## 🔧 Structured Logging

### JSON Format

```typescript
// ❌ Unstructured
console.log(`User ${userId} logged in from ${ip}`);

// ✅ Structured
logger.info('user_login', {
  userId,
  ip,
  timestamp: new Date().toISOString(),
  userAgent: req.headers['user-agent'],
});
```

### Winston Setup (Node.js)

```typescript
// logger.ts
import winston from 'winston';
import 'winston-daily-rotate-file';

const { combine, timestamp, printf, errors, json } = winston.format;

const logFormat = printf(({ level, message, timestamp, ...meta }) => {
  return JSON.stringify({
    level,
    message,
    timestamp,
    ...meta,
  });
});

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(
    errors({ stack: true }),
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    json(),
  ),
  defaultMeta: {
    service: 'user-service',
    version: process.env.APP_VERSION || '1.0.0',
  },
  transports: [
    // Console
    new winston.transports.Console(),
    
    // Error logs
    new winston.transports.DailyRotateFile({
      filename: 'logs/error-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      level: 'error',
      maxSize: '20m',
      maxFiles: '14d',
    }),
    
    // All logs
    new winston.transports.DailyRotateFile({
      filename: 'logs/combined-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxSize: '20m',
      maxFiles: '14d',
    }),
  ],
});

// NestJS integration
// main.ts
app.useLogger(app.get(Logger));
```

### Correlation ID

```typescript
// middleware/correlation-id.middleware.ts
import { v4 as uuidv4 } from 'uuid';

export function correlationIdMiddleware(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  const correlationId = req.headers['x-correlation-id'] as string || uuidv4();
  
  // Add to response headers
  res.setHeader('x-correlation-id', correlationId);
  
  // Add to request for logging
  (req as any).correlationId = correlationId;
  
  next();
}

// logger.ts with correlation ID
export const logger = winston.createLogger({
  format: combine(
    timestamp(),
    printf(({ correlationId, ...meta }) => {
      return JSON.stringify({
        correlation_id: correlationId,
        ...meta,
      });
    }),
  ),
});

// Usage in controller
@Controller('users')
export class UsersController {
  constructor(private logger: LoggerService) {}

  @Get(':id')
  async getUser(@Param('id') id: string, @Req() req: Request) {
    this.logger.info('Fetching user', {
      userId: id,
      correlationId: (req as any).correlationId,
    });
    
    return this.usersService.findOne(id);
  }
}
```

---

## 📊 Log Aggregation

### ELK Stack (Elasticsearch, Logstash, Kibana)

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

```ruby
# logstash/pipeline/logstash.conf
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  if [level] == "error" {
    mutate {
      add_tag => ["error"]
    }
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

### Grafana Loki (Lighter Alternative)

```yaml
# docker-compose.yml
services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

volumes:
  loki_data:
```

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: container-logs
          __path__: /var/lib/docker/containers/*/*log
```

---

## 🎯 Best Practices

### ✅ Делать

```typescript
// 1. Structured logging
logger.info('order_created', {
  orderId: order.id,
  userId: order.userId,
  total: order.total,
});

// 2. Include context
logger.error('Payment failed', {
  userId: user.id,
  amount: order.total,
  errorCode: paymentError.code,
  correlationId: req.correlationId,
});

// 3. Log at right level
if (retryCount >= maxRetries) {
  logger.error('Max retries exceeded', { retryCount });
} else if (retryCount > 0) {
  logger.warn('Retrying operation', { retryCount });
} else {
  logger.info('Operation started');
}

// 4. Redact sensitive data
logger.info('User login', {
  userId: user.id,
  email: maskEmail(user.email),  // j***@example.com
  // Never log passwords!
});

// 5. Add timing
const start = Date.now();
await operation();
logger.debug('Operation completed', {
  duration: Date.now() - start,
});
```

### ❌ Не делать

```typescript
// 1. Don't log sensitive data
logger.info('User login', {
  password: password,  // ❌
});

// 2. Don't log huge objects
logger.debug('Request', {
  body: req.body,  // ❌ May be MBs
});

// 3. Don't log in loops
for (const item of items) {
  logger.info('Processing', { item });  // ❌
}
logger.info('Processing batch', { count: items.length });  // ✅

// 4. Don't use string concatenation
logger.info('User ' + userId + ' logged in');  // ❌
logger.info('User login', { userId });  // ✅

// 5. Don't ignore errors
try {
  await operation();
} catch (error) {
  // Empty catch  ❌
}

try {
  await operation();
} catch (error) {
  logger.error('Operation failed', { error: error.message, stack: error.stack });
  throw error;  // ✅
}
```

---

## 📈 Log Analysis

### Common Queries (Kibana/Loki)

```
# Find errors
level: error

# Find errors for specific service
level: error AND service: user-service

# Find slow queries
duration:>1000

# Find by correlation ID
correlation_id: abc-123-def

# Find user actions
userId: 12345 AND message: "user_*"

# Error rate
level: error | stats count() by service

# Response time percentiles
duration | stats percentile(duration, 50, 95, 99)
```

### Alerting

```yaml
# Alert on error rate
alert: HighErrorRate
expr: |
  sum(rate(logs_total{level="error"}[5m])) 
  / sum(rate(logs_total[5m])) > 0.05
for: 5m
labels:
  severity: critical
annotations:
  summary: "High error rate detected"
```

---

## 🔗 Связанные заметки

- [[Monitoring-Cheatsheet]] — Мониторинг и метрики
- [[Security-Hardening]] — Security logging
- [[Docker-Cheatsheet]] — Docker логи

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Structured logging** — JSON format
> 2. **Correlation ID** — trace requests
> 3. **Right level** — ERROR, WARN, INFO, DEBUG
> 4. **Redact sensitive** — never log passwords
> 5. **Centralized** — ELK or Loki

> [!INFO] 12-Factor Logs
> 
> - Logs are streams
> - Don't manage files in app
> - Ship to aggregator
> - Let infra handle storage
