---
created: 2026-02-17
tags:
  - moc
  - cicd
  - devops
  - github-actions
aliases:
  - CI/CD MOC
  - DevOps Index
related:
  - MOC-Infrastructure
  - MOC-Security
---

# 🔄 CI/CD — Индекс

> [!SUMMARY] Обзор
> Continuous Integration / Continuous Delivery: GitHub Actions, GitLab CI, ArgoCD.

---

## 🗂️ Навигация

| Область | Файл | Описание |
|---------|------|----------|
| 📖 MOC | [[00-MOC-CICD]] | Главная страница |
| **Pipeline** | | |
| CI/CD Pipeline | [[CI-CD-Pipeline]] | Настройка пайплайнов |
| **GitHub Actions** | | |
| GitHub Actions | `GitHub-Actions.md` | GitHub CI/CD *(создать)* |
| **GitLab CI** | | |
| GitLab CI | `GitLab-CI.md` | GitLab CI/CD *(создать)* |
| **ArgoCD** | | |
| ArgoCD | `ArgoCD.md` | GitOps CD *(создать)* |

---

## 🔄 CI/CD Pipeline

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   Code   │ →  │  Build   │ →  │   Test   │ →  │  Deploy  │ →  │ Monitor  │
│  (Git)   │    │ (Docker) │    │  (CI)    │    │  (K8s)   │    │ (Prom)   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## 🔗 Связанные заметки

- [[MOC-Infrastructure]] — инфраструктура
- [[MOC-Security]] — безопасность
- [[../50-Infrastructure/Docker/Docker-Cheatsheet]] — Docker
- [[../50-Infrastructure/Kubernetes/Kubernetes-Cheatsheet]] — Kubernetes
