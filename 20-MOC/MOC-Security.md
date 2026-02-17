---
created: 2026-02-16
tags:
  - moc
  - security
related:
  - MOC-Backend
  - MOC-Infrastructure
---

# 🔐 MOC — Безопасность

> [!ABSTRACT] Обзор
> Практики и инструменты для защиты приложений.

---

## 🗂️ Навигация

| Тема | Файл | Статус |
|------|------|--------|
| Hardening | [[Hardening/Security-Hardening]] | 🟢 Готов |
| Auth Patterns | [[Auth/Auth-Patterns]] | 🟢 Готов |
| OWASP Top 10 | `OWASP-Top-10.md` | ⚪ Создать |

---

## 🛡️ Security Checklist

### Application Level
- [ ] Валидация всех входных данных
- [ ] Prepared statements для SQL
- [ ] HTTPS только
- [ ] Secure headers (CSP, HSTS)
- [ ] Rate limiting
- [ ] JWT с коротким TTL + refresh tokens

### Infrastructure Level
- [ ] Non-root контейнеры
- [ ] Secrets в vault, не в коде
- [ ] Network policies
- [ ] Regular security updates
- [ ] Audit logs

---

## 🔑 Auth Patterns

```
┌─────────────────────────────────────────────────────┐
│                  Authentication                      │
├──────────────┬──────────────┬───────────────────────┤
│   Session    │     JWT      │      OAuth 2.0        │
│   (cookie)   │   (token)    │  (Google, GitHub...)  │
└──────────────┴──────────────┴───────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                  Authorization                       │
├──────────────┬──────────────┬───────────────────────┤
│    RBAC      │    ABAC      │       ACL             │
│  (roles)     │ (attributes) │  (access lists)       │
└──────────────┴──────────────┴───────────────────────┘
```

---

## 🚨 OWASP Top 10 (2021)

| # | Уязвимость | Защита |
|---|------------|--------|
| A01 | Broken Access Control | RBAC, проверка прав |
| A02 | Cryptographic Failures | TLS, шифрование данных |
| A03 | Injection | Prepared statements, валидация |
| A04 | Insecure Design | Threat modeling |
| A05 | Security Misconfiguration | Hardening guides |
| A06 | Vulnerable Components | SCA сканирование |
| A07 | Auth Failures | MFA, rate limiting |
| A08 | Data Integrity | Подписи, checksums |
| A09 | Logging Failures | Centralized logging |
| A10 | SSRF | Whitelist, network policies |

---

## 🔧 Security Tools

| Категория | Инструменты |
|-----------|-------------|
| SAST | SonarQube, Semgrep |
| DAST | OWASP ZAP, Burp Suite |
| SCA | Snyk, Dependabot |
| Secrets | HashiCorp Vault, AWS Secrets Manager |
| Container | Trivy, Docker Scan |

---

## 🔗 Связанные заметки

- [[MOC-Backend]] — Backend разработка
- [[MOC-Infrastructure]] — Инфраструктура
- [[00-MOC/00-MOC-Security]] — Security индекс
- [[Docker-Cheatsheet]] — Docker
- [[Kubernetes-Cheatsheet]] — Kubernetes
