---
created: 2026-02-17
tags:
  - docker
  - best-practices
  - security
  - optimization
aliases:
  - Docker Best Practices
  - Docker Security Optimization
related:
  - Docker-Cheatsheet
  - Kubernetes-Cheatsheet
  - MOC-Infrastructure
---

# Docker Best Practices

> [!SUMMARY] Обзор
> Лучшие практики Docker: безопасность, оптимизация образов, multi-stage builds, health checks.

---

## 🏗️ Dockerfile Best Practices

### Multi-stage Builds

```dockerfile
# ❌ Bad: Large image
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/index.js"]
# Size: ~1GB

# ✅ Good: Multi-stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001
USER nestjs
EXPOSE 3000
CMD ["node", "dist/index.js"]
# Size: ~200MB
```

### Layer Caching

```dockerfile
# ❌ Bad: Cache invalidated on every change
FROM node:20
WORKDIR /app
COPY . .
RUN npm install

# ✅ Good: Install dependencies first
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

# ✅ Better: Separate prod and dev dependencies
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
```

### .dockerignore

```bash
# Node modules
node_modules
npm-debug.log

# Git
.git
.gitignore

# IDE
.vscode
.idea
*.swp
*.swo

# Build
dist
build
*.log

# Tests
coverage
.nyc_output
*.test.ts

# Docker
Dockerfile
.dockerignore
docker-compose*.yml

# Docs
README.md
*.md

# Env
.env
.env.*
!.env.example
```

---

## 🔒 Security

### Non-root User

```dockerfile
# ✅ Create non-root user
FROM node:20-alpine
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001
USER nestjs
WORKDIR /home/nodejs/app
CMD ["node", "dist/index.js"]
```

### Minimal Base Images

```dockerfile
# Size comparison
FROM node:20              # ~1GB
FROM node:20-slim         # ~500MB
FROM node:20-alpine       # ~175MB
FROM gcr.io/distroless/nodejs20  # ~50MB

# ✅ Recommended: Alpine or Distroless
FROM node:20-alpine
# OR
FROM gcr.io/distroless/nodejs20
```

### Scan for Vulnerabilities

```bash
# Docker scan
docker scan myimage

# Trivy
trivy image myimage

# Grype
grype myimage

# In CI/CD
docker build -t myapp .
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp
```

### Secrets Management

```dockerfile
# ❌ Bad: Hardcoded secrets
ENV API_KEY=secret123
ENV DB_PASSWORD=password

# ✅ Good: Build-time secrets
FROM node:20-alpine AS builder
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc \
    npm ci

# ✅ Good: Runtime secrets
docker run -e API_KEY=$(vault read -field=key secret/api) myapp

# ✅ Good: Docker secrets (Swarm)
echo "mysecret" | docker secret create my_secret -
docker service create --secret my_secret myapp
```

---

## ⚡ Optimization

### Reduce Layers

```dockerfile
# ❌ Bad: Multiple RUN commands
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good: Combine commands
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    git && \
    rm -rf /var/lib/apt/lists/*
```

### Copy Only What's Needed

```dockerfile
# ❌ Bad: Copy everything
COPY . .

# ✅ Good: Selective copy
COPY package*.json ./
COPY src/ ./src/
COPY tsconfig.json ./
```

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD wget -q --spider http://localhost:3000/health || exit 1
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s
```

---

## 🎯 Production Checklist

### Dockerfile

```dockerfile
# 1. Use specific version
FROM node:20.10.0-alpine

# 2. Multi-stage build
FROM node:20.10.0-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# 3. Production image
FROM node:20.10.0-alpine AS production

# 4. Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

# 5. Copy artifacts
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# 6. Set ownership
RUN chown -R nestjs:nodejs /app

# 7. Switch to non-root
USER nestjs

# 8. Expose port
EXPOSE 3000

# 9. Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD wget -q --spider http://localhost:3000/health || exit 1

# 10. Start command
CMD ["node", "dist/index.js"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    env_file:
      - .env.production
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
```

---

## 📊 Image Size Optimization

```bash
# Check image size
docker images

# Check layers
docker history myimage

# Check what's using space
docker image inspect myimage

# Remove unused images
docker image prune -a

# Build with no cache
docker build --no-cache -t myimage .

# Use buildkit
DOCKER_BUILDKIT=1 docker build -t myimage .
```

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — Docker шпаргалка
- [[Kubernetes-Cheatsheet]] — Kubernetes
- [[MOC-Infrastructure]] — Infrastructure MOC
