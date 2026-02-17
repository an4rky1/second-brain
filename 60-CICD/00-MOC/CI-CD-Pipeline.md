---
created: 2026-02-16
tags:
  - cheat-sheet
  - cicd
  - github-actions
  - devops
aliases:
  - CI/CD Cheatsheet
  - GitHub Actions Reference
related:
  - Docker-Cheatsheet
  - Kubernetes-Cheatsheet
  - Terraform-Cheatsheet
---

# CI/CD — Полная шпаргалка

> [!SUMMARY] Обзор
> Continuous Integration / Continuous Delivery — автоматизация сборки, тестирования и деплоя. GitHub Actions, GitLab CI, ArgoCD. Ускоряет доставку кода и снижает риски.

---

## 📚 Теория

### CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline                                │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   Code   │→ │  Build   │→ │   Test   │→ │  Deploy  │        │
│  │  (Git)   │  │ (Docker) │  │  (CI)    │  │  (CD)    │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│       │            │            │            │                  │
│       ▼            ▼            ▼            ▼                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Lint    │  │   Unit   │  │ Security │  │   Helm   │        │
│  │  Type    │  │  E2E     │  │  Scan    │  │  K8s     │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Rolling Update:    [v1]→[v1,v2]→[v2,v2]→[v2]                   │
│                                                                  │
│  Blue-Green:        [Blue:v1] ─switch─→ [Green:v2]              │
│                                                                  │
│  Canary:            [90% v1] + [10% v2] → gradual increase      │
│                                                                  │
│  Recreate:          [stop v1] → [start v2]                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚡ Быстрый старт

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'
  REGISTRY: ghcr.io

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Type check
        run: npm run type-check

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  build:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ github.repository }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
      
      - name: Deploy to staging
        run: |
          kubectl set image deployment/app app=${{ env.REGISTRY }}/${{ github.repository }}:${{ github.sha }}
          kubectl rollout status deployment/app

  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
      
      - name: Deploy with Helm
        run: |
          helm upgrade --install app ./charts/app \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=3 \
            --wait --timeout 5m
```

### Advanced Workflows

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version from tag
        id: get_version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Build changelog
        id: changelog
        uses: mikepenz/release-changelog-builder-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          body: ${{ steps.changelog.outputs.changelog }}
          draft: false
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

# .github/workflows/nightly.yml
name: Nightly Build

on:
  schedule:
    - cron: '0 2 * * *'  # Every day at 2 AM
  workflow_dispatch:

jobs:
  nightly:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Run security scan
        run: npm audit --audit-level=high
      
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Nightly build failed: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  DOCKER_TLS_CERTDIR: "/certs"

.default-image:
  image: node:18-alpine
  before_script:
    - npm ci

lint:
  extends: .default-image
  stage: lint
  script:
    - npm run lint
    - npm run type-check

test:
  extends: .default-image
  stage: test
  script:
    - npm test -- --coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
  coverage: '/All files\s+\|\s+\d+\.\d+/'

build:
  image: docker:24
  stage: build
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main
    - develop

deploy-staging:
  image: bitnami/kubectl:latest
  stage: deploy
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE
    - kubectl rollout status deployment/app
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy-production:
  image: bitnami/kubectl:latest
  stage: deploy
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE
    - kubectl rollout status deployment/app
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual  # Requires manual approval
```

### ArgoCD (GitOps)

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/infra.git
    targetRevision: HEAD
    path: apps/myapp
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 🔧 Практические примеры

### Docker in CI

```yaml
# Build with buildx (multi-arch)
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: |
      myapp:latest
      myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Caching

```yaml
# npm cache
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# Go cache
- uses: actions/cache@v3
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-

# Docker layer cache
- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```

### Secrets Management

```yaml
# GitHub Secrets
steps:
  - name: Use secret
    run: echo "$MY_SECRET"
    env:
      MY_SECRET: ${{ secrets.MY_SECRET }}

# AWS credentials
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

# Kubernetes secrets
- name: Create secret
  run: |
    kubectl create secret generic app-secret \
      --from-literal=api-key=${{ secrets.API_KEY }} \
      --dry-run=client -o yaml | kubectl apply -f -
```

### Matrix Builds

```yaml
# Test on multiple versions
test:
  runs-on: ubuntu-latest
  strategy:
    matrix:
      node-version: [16, 18, 20]
      os: [ubuntu-latest, macos-latest]
    fail-fast: false
  steps:
    - uses: actions/checkout@v4
    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
    - run: npm test
```

---

## 🎯 Best Practices

### ✅ Делать

```yaml
# 1. Pin action versions
uses: actions/checkout@v4  # ✅
uses: actions/checkout@main  # ❌

# 2. Use environments
environment: production  # С секретами и правилами

# 3. Minimal permissions
permissions:
  contents: read
  packages: write

# 4. Cache dependencies
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}

# 5. Fail fast
strategy:
  fail-fast: true
```

### ❌ Не делать

```yaml
# 1. Хардкод секретов
run: echo "password123"  # ❌

# 2. checkout@main (непинированная версия)
uses: actions/checkout@main  # ❌

# 3. Излишние permissions
permissions: write-all  # ❌

# 4. Игнорирование ошибок
run: npm test || true  # ❌
```

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — Docker в CI
- [[Kubernetes-Cheatsheet]] — K8s деплой
- [[Terraform-Cheatsheet]] — Инфраструктура

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **Fast feedback** — линт и тесты первыми
> 2. **Cache everything** — зависимости, Docker слои
> 3. **Ephemeral environments** — staging на каждый PR
> 4. **GitOps** — ArgoCD для K8s деплоя
> 5. **Security scanning** — SAST, DAST, dependency check

> [!INFO] Полезные Actions
> ```yaml
> # K8s
> - uses: azure/setup-kubectl@v3
> - uses: azure/setup-helm@v3
> ```

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — контейнеризация
- [[Kubernetes-Cheatsheet]] — оркестрация
- [[Terraform-Cheatsheet]] — Infrastructure as Code
- [[MOC-CICD]] — CI/CD навигация
