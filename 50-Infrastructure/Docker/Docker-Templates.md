---
created: 2026-02-16
tags:
  - docker
  - templates
  - devops
  - infrastructure
aliases:
  - Docker Templates
  - Docker Compose Templates
related:
  - Docker-Cheatsheet
  - Kubernetes-Cheatsheet
  - Database-Design
---

# Docker — Шаблоны для сервисов

> [!SUMMARY] Обзор
> Готовые Docker Compose шаблоны для популярных сервисов: PostgreSQL, Redis, Kafka, MongoDB, Elasticsearch и другие.

---

## 📁 Структура

```
docker/
├── development/       # Local development
│   └── docker-compose.yml
├── production/        # Production ready
│   └── docker-compose.yml
└── services/          # Отдельные сервисы
    ├── postgres/
    ├── redis/
    ├── kafka/
    └── ...
```

---

## 🐘 PostgreSQL

### Basic (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Production Ready

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "127.0.0.1:5432:5432"  # Только localhost
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
    command: >
      postgres
      -c max_connections=200
      -c shared_buffers=256MB
      -c effective_cache_size=768MB
      -c work_mem=16MB
      -c maintenance_work_mem=128MB
      -c wal_level=replica
      -c max_wal_senders=3
      -c max_replication_slots=3
      -c hot_standby=on
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
      PGADMIN_CONFIG_SERVER_MODE: 'False'
    ports:
      - "5050:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      - postgres

volumes:
  postgres_data:
  pgadmin_data:
```

### С репликацией

```yaml
version: '3.8'

services:
  postgres-master:
    image: postgres:15-alpine
    container_name: postgres-master
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
      REPLICATION_MODE: master
      REPLICATION_USER: replicator
      REPLICATION_PASSWORD: replicator
    volumes:
      - master_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  postgres-slave:
    image: postgres:15-alpine
    container_name: postgres-slave
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      REPLICATION_MODE: slave
      REPLICATION_USER: replicator
      REPLICATION_PASSWORD: replicator
      REPLICATION_HOST: postgres-master
    volumes:
      - slave_data:/var/lib/postgresql/data
    depends_on:
      - postgres-master
    ports:
      - "5433:5432"

volumes:
  master_data:
  slave_data:
```

---

## 🔴 Redis

### Basic (Development)

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:
```

### Production Ready (с паролем)

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: always
    command: >
      redis-server
      --port 6379
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
      --save 60 10000
    ports:
      - "127.0.0.1:6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      REDIS_HOSTS: local:redis:6379:0:${REDIS_PASSWORD}
      HTTP_USER: admin
      HTTP_PASSWORD: admin
    ports:
      - "8081:8081"
    depends_on:
      - redis

volumes:
  redis_data:
```

### Redis Cluster

```yaml
version: '3.8'

services:
  redis-node-1:
    image: redis:7-alpine
    command: redis-server --port 7000 --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --appendfilename appendonly-7000.aof
    ports:
      - "7000:7000"
    volumes:
      - redis-node-1-data:/data

  redis-node-2:
    image: redis:7-alpine
    command: redis-server --port 7001 --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --appendfilename appendonly-7001.aof
    ports:
      - "7001:7001"
    volumes:
      - redis-node-2-data:/data

  redis-node-3:
    image: redis:7-alpine
    command: redis-server --port 7002 --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --appendfilename appendonly-7002.aof
    ports:
      - "7002:7002"
    volumes:
      - redis-node-3-data:/data

  redis-cluster-creator:
    image: redis:7-alpine
    entrypoint: redis-cli --cluster create redis-node-1:7000 redis-node-2:7001 redis-node-3:7002 --cluster-replicas 0 --cluster-yes
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3

volumes:
  redis-node-1-data:
  redis-node-2-data:
  redis-node-3-data:
```

---

## 🐘 Kafka (без Zookeeper — KRaft)

### Single Node (Development)

```yaml
version: '3.8'

services:
  kafka:
    image: apache/kafka:3.6.0
    container_name: kafka
    restart: unless-stopped
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
    healthcheck:
      test: nc -z kafka 9092 || exit -1
      interval: 10s
      timeout: 5s
      retries: 5

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: ""
    depends_on:
      - kafka

volumes:
  kafka_data:
```

### Production Cluster

```yaml
version: '3.8'

services:
  # Controller nodes (KRaft)
  kafka-controller-1:
    image: apache/kafka:3.6.0
    container_name: kafka-controller-1
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093,3@kafka-controller-3:9093
      KAFKA_LISTENERS: CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - controller-1-data:/var/kafka/data

  kafka-controller-2:
    image: apache/kafka:3.6.0
    container_name: kafka-controller-2
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093,3@kafka-controller-3:9093
      KAFKA_LISTENERS: CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - controller-2-data:/var/kafka/data

  kafka-controller-3:
    image: apache/kafka:3.6.0
    container_name: kafka-controller-3
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093,3@kafka-controller-3:9093
      KAFKA_LISTENERS: CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - controller-3-data:/var/kafka/data

  # Broker nodes
  kafka-broker-1:
    image: apache/kafka:3.6.0
    container_name: kafka-broker-1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 4
      KAFKA_PROCESS_ROLES: broker
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093,3@kafka-controller-3:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT_HOST://localhost:9092,PLAINTEXT://kafka-broker-1:29092
      KAFKA_LISTENERS: PLAINTEXT_HOST://0.0.0.0:9092,PLAINTEXT://0.0.0.0:29092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    depends_on:
      - kafka-controller-1
      - kafka-controller-2
      - kafka-controller-3
    volumes:
      - broker-1-data:/var/kafka/data

volumes:
  controller-1-data:
  controller-2-data:
  controller-3-data:
  broker-1-data:
```

---

## 🍃 MongoDB

### Basic

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    container_name: mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: admin
      MONGO_INITDB_DATABASE: myapp
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: admin
      ME_CONFIG_MONGODB_URL: mongodb://admin:admin@mongodb:27017/
      ME_CONFIG_BASICAUTH: true
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: admin
    ports:
      - "8082:8081"
    depends_on:
      - mongodb

volumes:
  mongo_data:
```

---

## 🔍 Elasticsearch + Kibana

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    healthcheck:
      test: curl -s http://localhost:9200/_cluster/health | grep -E '"status":"(green|yellow)"'
      interval: 30s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    restart: unless-stopped
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

---

## 🐰 RabbitMQ

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  rabbitmq_data:
```

---

## 🗄️ MySQL

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    command: --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: mysqladmin ping -h localhost -u root -prootpassword
      interval: 10s
      timeout: 5s
      retries: 5

  adminer:
    image: adminer:latest
    container_name: adminer
    restart: unless-stopped
    ports:
      - "8083:8080"
    depends_on:
      - mysql

volumes:
  mysql_data:
```

---

## 🌊 Nginx (Reverse Proxy)

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - ./logs:/var/log/nginx
    depends_on:
      - app
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  app:
    build: .
    restart: unless-stopped
    expose:
      - "3000"

volumes:
  nginx_logs:
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://app;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — Docker основы
- [[Kubernetes-Cheatsheet]] — K8s деплой
- [[Database-Design]] — Проектирование БД

---

## 📝 Заметки

> [!TIP] Совет
> 
> 1. **Используйте Alpine образы** — меньше размер
> 2. **Health checks** — обязательны для production
> 3. **Volumes для данных** — контейнеры эфемерны
> 4. **.env для секретов** — не хардкодьте пароли
> 5. **Network isolation** — отдельные сети для БД
