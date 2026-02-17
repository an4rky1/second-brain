---
created: 2026-02-16
tags:
  - cheat-sheet
  - docker
  - containers
  - devops
aliases:
  - Docker Cheatsheet
  - Docker Reference
related:
  - Kubernetes-Cheatsheet
  - Docker-Templates
  - MOC-Infrastructure
---

# Docker — Полная шпаргалка

> [!SUMMARY] Обзор
> Платформа для разработки, доставки и запуска приложений в контейнерах. Контейнеризация, оркестрация, CI/CD.

---

## 📚 Теория

### Dockerfile

### Структура Dockerfile

```dockerfile
# Базовый образ
FROM node:18-alpine

# Метаданные
LABEL maintainer="john@example.com"
LABEL version="1.0"
LABEL description="My application"

# Переменные окружения
ENV NODE_ENV=production
ENV PORT=3000

# Аргументы сборки
ARG VERSION=1.0
ARG NPM_REGISTRY=https://registry.npmjs.org

# Рабочая директория
WORKDIR /app

# Копирование файлов
COPY package*.json ./
COPY src/ ./src/
COPY . .

# Добавление файлов (скачивание из URL)
ADD https://example.com/file.tar.gz /tmp/

# Установка зависимостей
RUN npm ci --only=production

# Сборка
RUN npm run build

# Пользователь
USER node

# Проброс порта
EXPOSE 3000

# Том (volume)
VOLUME ["/app/data"]

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1

# Точка входа
ENTRYPOINT ["node"]

# Команда по умолчанию
CMD ["dist/index.js"]
```

---

### Поля Dockerfile — расшифровка

| Инструкция | Описание | Пример |
|------------|----------|--------|
| `FROM` | Базовый образ | `FROM node:18-alpine` |
| `LABEL` | Метаданные образа | `LABEL version="1.0"` |
| `ENV` | Переменные окружения | `ENV NODE_ENV=production` |
| `ARG` | Аргументы сборки | `ARG VERSION=1.0` |
| `WORKDIR` | Рабочая директория | `WORKDIR /app` |
| `COPY` | Копирование файлов | `COPY . .` |
| `ADD` | Копирование + распаковка/URL | `ADD archive.tar.gz /tmp/` |
| `RUN` | Выполнить команду | `RUN npm install` |
| `CMD` | Команда по умолчанию | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | Точка входа | `ENTRYPOINT ["node"]` |
| `EXPOSE` | Проброс порта | `EXPOSE 3000` |
| `VOLUME` | Том для данных | `VOLUME ["/data"]` |
| `USER` | Пользователь для запуска | `USER node` |
| `HEALTHCHECK` | Проверка здоровья | `HEALTHCHECK CMD curl ...` |
| `SHELL` | Оболочка для команд | `SHELL ["/bin/bash", "-c"]` |
| `STOPSIGNAL` | Сигнал остановки | `STOPSIGNAL SIGTERM` |
| `ONBUILD` | Триггеры для дочерних образов | `ONBUILD RUN npm install` |

---

### Инструкции подробно

#### FROM
```dockerfile
FROM ubuntu:22.04                    # Конкретная версия
FROM node:18-alpine                  # Alpine (минимальный)
FROM node:18-slim                    # Slim (облегчённый)
FROM node:latest                     # Последняя версия
FROM scratch                         # Пустой образ
FROM myimage:1.0 AS builder          # Именованный этап (multi-stage)
```

#### ENV vs ARG
```dockerfile
# ARG - только во время сборки
ARG VERSION=1.0
RUN echo $VERSION

# ENV - во время сборки и в контейнере
ENV NODE_ENV=production
RUN echo $NODE_ENV
```

#### COPY vs ADD
```dockerfile
# COPY - простое копирование
COPY package.json ./

# ADD - копирование + распаковка архивов + URL
ADD archive.tar.gz /tmp/             # Распакует архив
ADD https://url.com/file /tmp/       # Скачает файл
```

#### CMD vs ENTRYPOINT
```dockerfile
# CMD - команда по умолчанию (можно переопределить)
CMD ["node", "app.js"]
# Запуск: docker run image → node app.js
# Запуск: docker run image python → python (переопределение)

# ENTRYPOINT - фиксированная команда
ENTRYPOINT ["node"]
CMD ["app.js"]
# Запуск: docker run image → node app.js
# Запуск: docker run image --version → node --version

# Shell форма (запускается через /bin/sh -c)
CMD node app.js
# Exec форма (рекомендуется)
CMD ["node", "app.js"]
```

#### Multi-stage сборка
```dockerfile
# Этап 1: Сборка
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Этап 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

---

## docker-compose.yml — Полное руководство

### Структура docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - NODE_ENV=production
    image: myapp:latest
    container_name: my-app
    restart: unless-stopped
    ports:
      - "3000:3000"
      - "8080:8080"
    expose:
      - "3001"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DB_HOST=db
    env_file:
      - .env
      - .env.production
    volumes:
      - ./src:/app/src
      - /app/node_modules
      - data-volume:/app/data
    networks:
      - app-network
    depends_on:
      - db
      - redis
    command: npm start
    entrypoint: ["node"]
    working_dir: /app
    user: "1000:1000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    secrets:
      - db_password
    configs:
      - app_config

  db:
    image: postgres:15-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    networks:
      - app-network

volumes:
  postgres-data:
  data-volume:

networks:
  app-network:
    driver: bridge

secrets:
  db_password:
    file: ./secrets/db_password.txt

configs:
  app_config:
    file: ./config/app.conf
```

---

### Поля docker-compose.yml — расшифровка

#### Корневые поля

| Поле | Описание | Пример |
|------|----------|--------|
| `version` | Версия формата (устарело в v3+) | `"3.8"` |
| `services` | Список сервисов | `services: { app: ... }` |
| `networks` | Сети | `networks: { app-net: ... }` |
| `volumes` | Тома | `volumes: { data: ... }` |
| `configs` | Конфигурации | `configs: { app_conf: ... }` |
| `secrets` | Секреты | `secrets: { db_pass: ... }` |

#### Поля сервиса

| Поле | Описание | Пример |
|------|----------|--------|
| `build` | Настройки сборки | `build: { context: . }` |
| `image` | Образ для запуска | `image: nginx:latest` |
| `container_name` | Имя контейнера | `container_name: my-app` |
| `restart` | Политика перезапуска | `restart: always` |
| `ports` | Проброс портов | `ports: ["80:80"]` |
| `expose` | Открыть порт (без проброса) | `expose: ["3000"]` |
| `environment` | Переменные окружения | `environment: { NODE_ENV: prod }` |
| `env_file` | Файлы с переменными | `env_file: [.env]` |
| `volumes` | Тома | `volumes: [./src:/app]` |
| `networks` | Сети | `networks: [app-net]` |
| `depends_on` | Зависимости | `depends_on: [db]` |
| `command` | Команда запуска | `command: npm start` |
| `entrypoint` | Точка входа | `entrypoint: ["/bin/sh"]` |
| `working_dir` | Рабочая директория | `working_dir: /app` |
| `user` | Пользователь | `user: "1000:1000"` |
| `healthcheck` | Проверка здоровья | `healthcheck: { test: ... }` |
| `logging` | Настройки логов | `logging: { driver: json-file }` |
| `deploy` | Развёртывание (Swarm) | `deploy: { replicas: 3 }` |
| `secrets` | Секреты | `secrets: [db_password]` |
| `configs` | Конфигурации | `configs: [app_config]` |
| `profiles` | Профили запуска | `profiles: [dev]` |
| `labels` | Метки | `labels: { app: myapp }` |
| `stop_grace_period` | Время остановки | `stop_grace_period: 30s` |
| `stop_signal` | Сигнал остановки | `stop_signal: SIGTERM` |

---

### Поля подробно

#### restart — Политики перезапуска
```yaml
restart: "no"           # Не перезапускать
restart: always         # Всегда перезапускать
restart: on-failure     # Только при ошибке
restart: unless-stopped # Если не остановлен вручную
```

#### ports — Проброс портов
```yaml
ports:
  - "3000:3000"         # host:container
  - "8080:80/tcp"       # С указанием протокола
  - "6000-6005:6000-6005" # Диапазон
  - "127.0.0.1:3000:3000" # Только localhost
```

#### volumes — Тома
```yaml
volumes:
  - ./src:/app/src              # Bind mount (хост → контейнер)
  - /app/node_modules           # Анонимный том
  - data-volume:/app/data       # Именованный том
  - type: bind
    source: ./data
    target: /app/data
    read_only: true             # Только чтение
  - type: volume
    source: data-volume
    target: /app/data
```

#### networks — Сети
```yaml
networks:
  - app-network                 # Подключить к сети
  - frontend
  - backend
  # С алиасами
  app-network:
    aliases:
      - api
      - backend-service
```

#### depends_on — Зависимости
```yaml
# Простая форма
depends_on:
  - db
  - redis

# С условиями (v3.8+)
depends_on:
  db:
    condition: service_healthy   # Ждать healthcheck
  redis:
    condition: service_started   # Ждать запуска
```

#### healthcheck — Проверка здоровья
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  # или: test: ["CMD-SHELL", "curl -f http://localhost/health || exit 1"]
  interval: 30s       # Интервал между проверками
  timeout: 10s        # Таймаут проверки
  retries: 3          # Количество попыток
  start_period: 40s   # Время на старт перед проверками
```

#### deploy — Развёртывание (Docker Swarm)
```yaml
deploy:
  replicas: 3                    # Количество реплик
  update_config:
    parallelism: 2
    delay: 10s
    order: start-first
  rollback_config:
    parallelism: 1
    delay: 10s
  resources:
    limits:
      cpus: "0.5"
      memory: 512M
    reservations:
      cpus: "0.25"
      memory: 256M
  placement:
    constraints:
      - node.role == manager
      - node.labels.type == app
  restart_policy:
    condition: on-failure
    delay: 5s
    max_attempts: 3
```

#### build — Настройки сборки
```yaml
build:
  context: .                     # Путь к Dockerfile
  dockerfile: Dockerfile         # Имя файла
  args:                          # Аргументы
    - NODE_ENV=production
    - VERSION=1.0
  labels:
    - com.example.description
  target: production             # Этап multi-stage
  cache_from:
    - myapp:cache
```

#### logging — Логирование
```yaml
logging:
  driver: json-file              # Драйвер логов
  options:
    max-size: "10m"              # Макс размер файла
    max-file: "3"                # Количество файлов
# Другие драйверы: syslog, journald, fluentd, splunk
```

---

## ⚡ Быстрый старт

### Основные команды

### Управление образами

```bash
docker pull nginx:latest         # Скачать образ
docker push myimage:latest       # Отправить образ
docker build -t myimage .        # Собрать образ
docker build -t myimg --build-arg VERSION=1.0 .
docker images                    # Список образов
docker rmi myimage               # Удалить образ
docker image prune               # Удалить неиспользуемые
docker image prune -a            # Удалить все неиспользуемые
docker tag myimage:latest myregistry.com/myimage:v1
docker save -o image.tar myimage # Сохранить в файл
docker load -i image.tar         # Загрузить из файла
```

### Управление контейнерами

```bash
docker run -d -p 80:80 nginx     # Запустить контейнер
docker run -it ubuntu bash       # Интерактивный запуск
docker run --rm myimage          # Удалить после остановки
docker run -v ./data:/app/data   # С томом
docker run -e NODE_ENV=prod      # С переменной окружения
docker run --name mycontainer    # С именем
docker run --network mynet       # С сетью
docker ps                        # Запущенные контейнеры
docker ps -a                     # Все контейнеры
docker stop mycontainer          # Остановить
docker start mycontainer         # Запустить
docker restart mycontainer       # Перезапустить
docker rm mycontainer            # Удалить
docker rm -f mycontainer         # Принудительно удалить
docker exec -it mycontainer bash # Выполнить команду
docker logs mycontainer          # Показать логи
docker logs -f mycontainer       # Follow логи
docker inspect mycontainer       # Подробная информация
docker cp file.txt container:/path # Копировать файл
docker stats                     # Статистика ресурсов
docker top mycontainer           # Процессы в контейнере
```

### Управление томами

```bash
docker volume create myvolume    # Создать том
docker volume ls                 # Список томов
docker volume inspect myvolume   # Информация о томе
docker volume rm myvolume        # Удалить том
docker volume prune              # Удалить неиспользуемые
```

### Управление сетями

```bash
docker network create mynet      # Создать сеть
docker network ls                # Список сетей
docker network inspect mynet     # Информация о сети
docker network rm mynet          # Удалить сеть
docker network connect mynet container # Подключить контейнер
docker network disconnect mynet container # Отключить
```

### Docker Compose

```bash
docker-compose up                # Запустить сервисы
docker-compose up -d             # Запустить в фоне
docker-compose up --build        # Пересобрать образы
docker-compose down              # Остановить и удалить
docker-compose down -v           # Удалить тома
docker-compose ps                # Статус сервисов
docker-compose logs              # Показать логи
docker-compose logs -f           # Follow логи
docker-compose exec service bash # Выполнить команду
docker-compose build             # Собрать образы
docker-compose restart           # Перезапустить
docker-compose stop              # Остановить
docker-compose start             # Запустить
docker-compose rm                # Удалить контейнеры
docker-compose config            # Проверить конфиг
docker-compose pull              # Скачать образы
docker-compose push              # Отправить образы
```

---

## 🎯 Best Practices

### Dockerfile

### ✅ Делать

```dockerfile
# ✓ Используйте конкретные версии
FROM node:18.16.0-alpine

# ✓ Multi-stage для уменьшения размера
FROM node:18 AS builder
RUN npm run build
FROM node:18-alpine
COPY --from=builder /app/dist /app

# ✓ .dockerignore для исключения лишних файлов
# node_modules
# npm-debug.log
# .git

# ✓ Запуск от не-root пользователя
USER node

# ✓ Минимизация слоёв
RUN apt-get update && apt-get install -y \
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*

# ✓ HEALTHCHECK для мониторинга
HEALTHCHECK --interval=30s CMD curl -f http://localhost/health
```

### docker-compose.yml

```yaml
# ✓ Используйте именованные тома
volumes:
  postgres-data:

# ✓ Сети для изоляции
networks:
  frontend:
  backend:

# ✓ Healthcheck для зависимостей
healthcheck:
  test: ["CMD", "pg_isready"]

# ✓ Secrets для чувствительных данных
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## 🐛 Решение проблем

```bash
# Очистить всё (образы, контейнеры, тома, сети)
docker system prune -a --volumes

# Посмотреть логи контейнера
docker logs --tail 100 container_name

# Проверить использование диска
docker system df

# Войти в запущенный контейнер
docker exec -it container_name bash

# Копировать файл из контейнера
docker cp container_name:/path/file.txt ./

# Проверить сеть контейнера
docker network inspect bridge

# Пересобрать без кэша
docker-compose build --no-cache
```

---

## 🔗 Связанные заметки

- [[Kubernetes-Cheatsheet]] — оркестрация контейнеров
- [[Docker-Templates]] — шаблоны Dockerfile
- [[MOC-Infrastructure]] — инфраструктура
- [[CI-CD-Pipeline]] — CI/CD с Docker
