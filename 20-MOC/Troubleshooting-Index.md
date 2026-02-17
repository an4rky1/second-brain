---
created: 2026-02-16
tags:
  - troubleshooting
  - index
---

# Troubleshooting Index

> [!ABSTRACT] Обзор
> Индекс решений распространённых проблем.

---

## 🔍 По технологиям

### TypeScript / JavaScript

| Проблема | Решение |
|----------|---------|
| `Cannot find module` | Проверьте paths в tsconfig, запустите `npm install` |
| `Property does not exist on type` | Добавьте тип или type guard |
| `Maximum call stack size exceeded` | Бесконечная рекурсия, проверьте base case |
| `Cannot read property of undefined` | Добавьте проверку или optional chaining |

### React / Next.js

| Проблема | Решение |
|----------|---------|
| `Hydration failed` | Проверьте условный рендеринг на сервере/клиенте |
| `Too many re-renders` | setState не в useEffect |
| `Module not found` | Проверьте путь, очистите .next |
| `Image optimization error` | Добавьте domain в next.config.js |

### Node.js / NestJS

| Проблема | Решение |
|----------|---------|
| `Cannot find module` | Проверьте paths, запустите `npm install` |
| `CORS error` | Добавьте CORS middleware |
| `Port already in use` | Найдите процесс: `lsof -i :3000` |
| `Memory leak` | Проверьте утечки через heap snapshot |

### Docker

| Проблема | Решение |
|----------|---------|
| `Cannot connect to Docker daemon` | `sudo systemctl start docker` |
| `Permission denied` | `sudo usermod -aG docker $USER` |
| `Container exits immediately` | Проверьте логи: `docker logs container_id` |
| `No space left on device` | `docker system prune -a` |

### Kubernetes

| Проблема | Решение |
|----------|---------|
| `CrashLoopBackOff` | Проверьте логи: `kubectl logs pod-name` |
| `ImagePullBackOff` | Проверьте image name и registry credentials |
| `Pending pod` | Проверьте ресурсы: `kubectl describe pod` |
| `Service not accessible` | Проверьте selector и endpoints |

### Database

| Проблема | Решение |
|----------|---------|
| `Connection refused` | Проверьте host, port, firewall |
| `Too many connections` | Увеличьте max_connections или используйте pool |
| `Deadlock detected` | Проверьте порядок блокировок |
| `Slow query` | EXPLAIN ANALYZE, добавьте индексы |

### Linux

| Проблема | Решение |
|----------|---------|
| `Permission denied` | Проверьте права: `ls -la`, используйте sudo |
| `Command not found` | Проверьте PATH, установите пакет |
| `No space left on device` | `df -h`, очистите логи и кэш |
| `High CPU usage` | `top`, найдите процесс, профилируйте |

---

## 🔧 Общие команды диагностики

```bash
# Логи
journalctl -xe              # Systemd логи
tail -f /var/log/syslog     # Системный лог
docker logs container_id    # Docker логи
kubectl logs pod-name       # K8s логи

# Сеть
curl -v http://localhost    # HTTP запрос
ss -tulpn                   # Порты
dig domain.com              # DNS
traceroute host             # Трассировка

# Процессы
ps aux | grep process       # Найти процесс
top                         # Мониторинг
strace -p PID               # Системные вызовы

# Диски
df -h                       # Место
du -sh *                    # Размер директорий
iotop                     # Disk I/O

# Память
free -h                     # RAM
vmstat 1                    # Виртуальная память
```

---

## 🔗 Связанные заметки

- [[Linux-Commands]] — Linux команды
- [[Docker-Cheatsheet]] — Docker
- [[Kubernetes-Cheatsheet]] — K8s

---

## 📝 Заметки

> [!TIP] Процесс troubleshooting
> 1. Воспроизведите проблему
> 2. Соберите логи и метрики
> 3. Изолируйте компонент
> 4. Проверьте гипотезы
> 5. Примените фикс
> 6. Документируйте решение
