---
created: 2026-02-16
tags:
  - cheat-sheet
  - kubernetes
  - k8s
  - orchestration
  - devops
aliases:
  - Kubernetes Cheatsheet
  - K8s Reference
related:
  - Docker-Cheatsheet
  - Terraform-Cheatsheet
  - MOC-Infrastructure
---

# Kubernetes — Полная шпаргалка

> [!SUMMARY] Обзор
> Система оркестрации контейнеров для автоматизации развёртывания, масштабирования и управления.

---

## 📚 Теория

### Архитектура Kubernetes

```
┌─────────────────────────────────────────────────────────┐
│                    Control Plane                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ kube-    │ │ etcd     │ │ kube-    │ │ cloud-   │   │
│  │ apiserver│ │          │ │ scheduler│ │ controller│  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
        ──────────────────┼─────────────────
                          │
┌─────────────────────────┼─────────────────────────┐
│    Worker Nodes         │      Worker Nodes       │
│  ┌──────────┐ ┌──────────┐ │  ┌──────────┐ ┌──────────┐ │
│  │ kubelet  │ │ kube-    │ │  │ kubelet  │ │ kube-    │ │
│  │          │ │ proxy    │ │  │          │ │ proxy    │ │
│  └──────────┘ └──────────┘ │  └──────────┘ └──────────┘ │
│  ┌──────────────────────┐ │  ┌──────────────────────┐ │
│  │   Containers (Pods)  │ │  │   Containers (Pods)  │ │
│  └──────────────────────┘ │  └──────────────────────┘ │
└───────────────────────────┴───────────────────────────┘
```

### Компоненты

| Компонент | Описание |
|-----------|----------|
| **kube-apiserver** | API сервер, фронтенд control plane |
| **etcd** | Распределённое хранилище ключ-значение |
| **kube-scheduler** | Планировщик подов на узлы |
| **kube-controller-manager** | Контроллеры (node, replication, endpoints) |
| **cloud-controller-manager** | Интеграция с облачными провайдерами |
| **kubelet** | Агент на узле, управляет подами |
| **kube-proxy** | Сетевой прокси, балансировка |
| **Container Runtime** | Среда запуска контейнеров (Docker, containerd) |

---

## Основные объекты K8s

| Объект | Описание | API Version |
|--------|----------|-------------|
| **Pod** | Минимальная единица (один или несколько контейнеров) | v1 |
| **Deployment** | Декларативное управление подами | apps/v1 |
| **StatefulSet** | Stateful приложения (базы данных) | apps/v1 |
| **DaemonSet** | Под на каждом узле | apps/v1 |
| **ReplicaSet** | Репликация подов | apps/v1 |
| **Service** | Сетевой сервис (ClusterIP, NodePort, LoadBalancer) | v1 |
| **Ingress** | Внешний HTTP/S доступ | networking.k8s.io/v1 |
| **ConfigMap** | Конфигурационные данные | v1 |
| **Secret** | Чувствительные данные | v1 |
| **PersistentVolume** | Хранилище в кластере | v1 |
| **PersistentVolumeClaim** | Запрос на хранилище | v1 |
| **Namespace** | Логическое разделение ресурсов | v1 |
| **ServiceAccount** | Идентификация для подов | v1 |
| **Role/ClusterRole** | Права доступа | rbac.authorization.k8s.io/v1 |
| **RoleBinding/ClusterRoleBinding** | Привязка ролей | rbac.authorization.k8s.io/v1 |
| **HorizontalPodAutoscaler** | Авто-масштабирование | autoscaling/v2 |
| **Job/CronJob** | Одноразовые/периодические задачи | batch/v1 |

---

## Pod — Полное руководство

### Пример Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: myapp
    version: v1
  annotations:
    description: "My application pod"
spec:
  restartPolicy: Always  # Always, OnFailure, Never
  terminationGracePeriodSeconds: 30
  activeDeadlineSeconds: 3600
  serviceAccountName: default
  automountServiceAccountToken: true
  nodeSelector:
    disktype: ssd
  nodeName: node-1
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myapp
        topologyKey: kubernetes.io/hostname
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  priorityClassName: high-priority
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: IfNotPresent  # Always, IfNotPresent, Never
    command: ["node"]
    args: ["dist/index.js"]
    workingDir: /app
    ports:
    - name: http
      containerPort: 3000
      protocol: TCP
    env:
    - name: NODE_ENV
      value: "production"
    - name: PORT
      value: "3000"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: config_value
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    volumeMounts:
    - name: data-volume
      mountPath: /app/data
    - name: config-volume
      mountPath: /app/config
      readOnly: true
    - name: secret-volume
      mountPath: /app/secrets
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "while true; do echo sidecar; sleep 10; done"]
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: data-pvc
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: app-secrets
  - name: empty-volume
    emptyDir: {}
  - name: host-volume
    hostPath:
      path: /var/log
      type: Directory
  initContainers:
  - name: init-db
    image: busybox
    command: ["sh", "-c", "until pg_isready; do sleep 2; done"]
```

---

### Поля Pod — расшифровка

#### Metadata

| Поле | Описание | Пример |
|------|----------|--------|
| `name` | Имя пода | `my-pod` |
| `namespace` | Пространство имён | `default` |
| `labels` | Метки для селекции | `app: myapp` |
| `annotations` | Метаданные (не для селекции) | `description: "..."` |
| `finalizers` | Очистка перед удалением | `kubernetes.io/pod-protection` |

#### Spec — Основные

| Поле | Описание | Пример |
|------|----------|--------|
| `restartPolicy` | Политика перезапуска | `Always`, `OnFailure`, `Never` |
| `serviceAccountName` | ServiceAccount для пода | `default` |
| `nodeSelector` | Выбор узла по меткам | `disktype: ssd` |
| `nodeName` | Конкретный узел | `node-1` |
| `affinity` | Правила размещения | node/pod affinity |
| `tolerations` | Игнорирование taints | `key: dedicated` |
| `priorityClassName` | Приоритет пода | `high-priority` |
| `terminationGracePeriodSeconds` | Время на остановку | `30` |
| `activeDeadlineSeconds` | Макс время жизни пода | `3600` |

#### Container

| Поле | Описание | Пример |
|------|----------|--------|
| `name` | Имя контейнера | `app` |
| `image` | Docker образ | `nginx:1.21` |
| `imagePullPolicy` | Политика pull | `Always`, `IfNotPresent`, `Never` |
| `command` | ENTRYPOINT | `["node"]` |
| `args` | CMD | `["dist/index.js"]` |
| `workingDir` | Рабочая директория | `/app` |
| `ports` | Порты контейнера | `containerPort: 3000` |
| `env` | Переменные окружения | `value`, `valueFrom` |
| `envFrom` | Загрузить из ConfigMap/Secret | `configMapRef` |
| `resources` | Ресурсы CPU/memory | `requests`, `limits` |
| `volumeMounts` | Подключенные тома | `mountPath` |
| `livenessProbe` | Проверка живости | `httpGet`, `exec`, `tcpSocket` |
| `readinessProbe` | Проверка готовности | `httpGet` |
| `startupProbe` | Проверка запуска | `failureThreshold: 30` |
| `securityContext` | Безопасность контейнера | `runAsUser` |

#### Probes (Проверки)

| Тип | Описание | Когда используется |
|-----|----------|-------------------|
| `livenessProbe` | Под жив — перезапустить если fail | Постоянно |
| `readinessProbe` | Под готов принимать трафик | Перед добавлением в Service |
| `startupProbe` | Приложение запустилось | Только при старте |

```yaml
# Типы проверок
livenessProbe:
  httpGet:           # HTTP проверка
    path: /health
    port: 8080
    httpHeaders:
    - name: Authorization
      value: Bearer token
  
  exec:              # Выполнение команды
    command:
    - cat
    - /tmp/healthy
  
  tcpSocket:         # TCP проверка
    port: 3306
  
  initialDelaySeconds: 10   # Задержка перед первой проверкой
  periodSeconds: 10         # Интервал между проверками
  timeoutSeconds: 5         # Таймаут проверки
  failureThreshold: 3       # Количество неудач до действия
  successThreshold: 1       # Количество успехов для восстановления
```

#### Resources

```yaml
resources:
  requests:          # Гарантированные ресурсы
    cpu: "100m"      # 0.1 CPU ядра
    memory: "128Mi"  # 128 Мебибайт
  limits:            # Максимальные ресурсы
    cpu: "500m"      # 0.5 CPU ядра
    memory: "512Mi"  # 512 Мебибайт
```

**Единицы измерения:**
- CPU: `100m` (милли-ядра), `0.5`, `1`, `2`
- Memory: `128Mi`, `1Gi`, `512M`

#### Volumes

| Тип тома | Описание | Пример |
|----------|----------|--------|
| `emptyDir` | Временный том (жизнь пода) | `emptyDir: {}` |
| `persistentVolumeClaim` | Постоянное хранилище | `claimName: data-pvc` |
| `configMap` | Данные из ConfigMap | `name: app-config` |
| `secret` | Данные из Secret | `secretName: my-secret` |
| `hostPath` | Путь на хосте | `path: /var/log` |
| `nfs` | NFS хранилище | `server: nfs.example.com` |

---

## Deployment

### Пример Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
      annotations:
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
```

### Поля Deployment

| Поле | Описание | Пример |
|------|----------|--------|
| `replicas` | Количество реплик | `3` |
| `selector` | Селектор подов | `matchLabels` |
| `template` | Шаблон пода | `pod template` |
| `strategy` | Стратегия обновления | `RollingUpdate`, `Recreate` |
| `minReadySeconds` | Мин время готовности пода | `10` |
| `revisionHistoryLimit` | История ревизий | `10` |
| `progressDeadlineSeconds` | Таймаут прогресса | `600` |

### Стратегии развёртывания

```yaml
# RollingUpdate (по умолчанию)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Макс новых подов сверх replicas
    maxUnavailable: 0  # Макс недоступных подов

# Recreate (удалить старые, создать новые)
strategy:
  type: Recreate
```

---

## Service

### Пример Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  type: ClusterIP  # ClusterIP, NodePort, LoadBalancer, ExternalName
  clusterIP: 10.96.0.1      # Или None для headless
  clusterIPs:
  - 10.96.0.1
  ports:
  - name: http
    port: 80
    targetPort: 3000
    protocol: TCP
    nodePort: 30080  # Для NodePort
  selector:
    app: myapp
  sessionAffinity: None  # None, ClientIP
  externalIPs:
  - 192.168.1.100
  loadBalancerIP: 192.168.1.50
  loadBalancerSourceRanges:
  - 10.0.0.0/8
```

### Типы Service

| Тип | Описание | Использование |
|-----|----------|---------------|
| `ClusterIP` | Внутренний IP (по умолчанию) | Внутренняя коммуникация |
| `NodePort` | Порт на каждом узле | Внешний доступ без LB |
| `LoadBalancer` | Балансировщик облака | Публичный доступ |
| `ExternalName` | DNS имя внешнего сервиса | Интеграция с внешними |

### Порты Service

```yaml
ports:
- port: 80        # Порт сервиса
  targetPort: 3000 # Порт контейнера
  nodePort: 30080 # Порт на узле (NodePort)
  protocol: TCP   # TCP, UDP, SCTP
  name: http      # Имя порта
```

---

## Ingress

### Пример Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Поля Ingress

| Поле | Описание | Пример |
|------|----------|--------|
| `ingressClassName` | Класс Ingress контроллера | `nginx` |
| `tls` | TLS конфигурация | `secretName: tls-secret` |
| `rules` | Правила маршрутизации | `host`, `paths` |
| `pathType` | Тип пути | `Exact`, `Prefix`, `ImplementationSpecific` |

---

## ConfigMap и Secret

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Простые ключ-значение
  DATABASE_URL: "postgres://localhost:5432/mydb"
  LOG_LEVEL: "info"
  
  # Файлы конфигурации
  app.properties: |
    server.port=8080
    server.host=0.0.0.0
  
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:3000;
      }
    }
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: default
type: Opaque  # Opaque, kubernetes.io/tls, kubernetes.io/dockerconfigjson
data:
  # Base64 encoded
  password: cGFzc3dvcmQxMjM=
stringData:
  # Plain text (K8s закодирует сам)
  api-key: "my-secret-key"
```

### Типы Secret

| Тип | Описание |
|-----|----------|
| `Opaque` | Произвольные данные |
| `kubernetes.io/tls` | TLS сертификат и ключ |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/basic-auth` | Basic auth credentials |
| `kubernetes.io/ssh-auth` | SSH ключи |

---

## PersistentVolume и PersistentVolumeClaim

### PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce  # RWO, ROX, RWX
  persistentVolumeReclaimPolicy: Retain  # Retain, Delete, Recycle
  storageClassName: standard
  claimRef:
    namespace: default
    name: data-pvc
  hostPath:
    path: /mnt/data
  # Или NFS
  # nfs:
  #   server: nfs.example.com
  #   path: /exports/data
  # Или облако
  # awsElasticBlockStore:
  #   volumeId: vol-12345678
  #   fsType: ext4
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
  # volumeName: data-pv  # Для привязки к конкретному PV
```

### AccessModes

| Режим | Описание |
|-------|----------|
| `ReadWriteOnce` (RWO) | Чтение/запись одним узлом |
| `ReadOnlyMany` (ROX) | Только чтение многими узлами |
| `ReadWriteMany` (RWX) | Чтение/запись многими узлами |

---

## StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  podManagementPolicy: OrderedReady  # OrderedReady, Parallel
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

---

## DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          hostPort: 9100
```

---

## Job и CronJob

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  completions: 1      # Количество успешных завершений
  parallelism: 1      # Параллельные поды
  backoffLimit: 3     # Количество перезапусков
  activeDeadlineSeconds: 3600
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["./backup.sh"]
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"           # Cron выражение
  timeZone: "UTC"
  concurrencyPolicy: Forbid       # Allow, Forbid, Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  suspend: false
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
```

### Cron выражения

```
# ┌ минута (0-59)
# │ ┌ час (0-23)
# │ │ ┌ день месяца (1-31)
# │ │ │ ┌ месяц (1-12)
# │ │ │ │ ┌ день недели (0-6, 0=Sunday)
# │ │ │ │ │
# * * * * *
```

---

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

---

## RBAC (Role-Based Access Control)

### Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-custom
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Верб (verbs)

| Верб | Описание |
|------|----------|
| `get` | Получить ресурс |
| `list` | Список ресурсов |
| `watch` | Следить за изменениями |
| `create` | Создать ресурс |
| `update` | Обновить ресурс |
| `patch` | Частичное обновление |
| `delete` | Удалить ресурс |
| `deletecollection` | Удалить коллекцию |

---

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
  annotations:
    description: "Production namespace"
spec:
  finalizers:
  - kubernetes
```

---

## kubectl — Основные команды

### Информация и получение

```bash
kubectl get pods                          # Список подов
kubectl get pods -n namespace             # В namespace
kubectl get pods -o wide                  # Подробно
kubectl get pods -o yaml                  # YAML формат
kubectl get pods -o json                  # JSON формат
kubectl get pods --show-labels            # С метками
kubectl get pods -l app=myapp             # По label selector
kubectl get all                           # Все ресурсы
kubectl get nodes                         # Узлы кластера
kubectl get namespaces                    # Namespace'ы
kubectl get services                      # Сервисы
kubectl get deployments                   # Deployment'ы
kubectl get configmaps                    # ConfigMap'ы
kubectl get secrets                       # Secrets
kubectl get ingress                       # Ingress
kubectl get pv                            # PersistentVolume
kubectl get pvc                           # PersistentVolumeClaim
kubectl get events --sort-by='.lastTimestamp' # События
kubectl api-resources                     # Доступные ресурсы
kubectl explain pod                       # Документация поля
kubectl explain pod.spec.containers
kubectl version                           # Версия kubectl и сервера
kubectl cluster-info                      # Информация о кластере
```

### Создание и применение

```bash
kubectl apply -f manifest.yaml            # Применить манифест
kubectl apply -f ./dir/                   # Применить все в директории
kubectl apply -f https://url/manifest.yaml
kubectl create -f manifest.yaml           # Создать ресурс
kubectl create deployment myapp --image=nginx
kubectl create namespace myns
kubectl create configmap myconfig --from-file=config.txt
kubectl create secret generic mysecret --from-literal=key=value
kubectl run mypod --image=nginx           # Запустить под
kubectl expose deployment myapp --port=80 --type=LoadBalancer
```

### Редактирование

```bash
kubectl edit pod mypod                    # Редактировать ресурс
kubectl edit deployment myapp
kubectl set image deployment/myapp app=myapp:v2
kubectl set env deployment/myapp KEY=value
kubectl set resources deployment/myapp --limits=cpu=500m,memory=512Mi
kubectl scale deployment myapp --replicas=3
kubectl rollout status deployment/myapp   # Статус развёртывания
kubectl rollout history deployment/myapp  # История ревизий
kubectl rollout undo deployment/myapp     # Откатить
kubectl rollout undo deployment/myapp --to-revision=2
kubectl rollout pause deployment/myapp    # Пауза
kubectl rollout resume deployment/myapp   # Возобновить
```

### Логи и выполнение команд

```bash
kubectl logs mypod                        # Логи пода
kubectl logs -f mypod                     # Follow логи
kubectl logs mypod -c container-name      # Логи конкретного контейнера
kubectl logs -l app=myapp --all-containers
kubectl exec -it mypod -- bash            # Выполнить команду
kubectl exec -it mypod -c container -- sh
kubectl exec mypod -- cat /etc/config
kubectl attach -it mypod                  # Подключиться к консоли
kubectl cp mypod:/path/file.txt ./        # Копировать из пода
kubectl cp ./file.txt mypod:/path/        # Копировать в под
```

### Отладка

```bash
kubectl describe pod mypod                # Подробная информация
kubectl describe node node-name
kubectl describe deployment myapp
kubectl top pods                          # Использование ресурсов
kubectl top nodes                         # Использование узлов
kubectl get events --sort-by='.lastTimestamp'
kubectl debug -it mypod --image=busybox   # Debug контейнер
kubectl port-forward mypod 8080:80        # Проброс порта
kubectl port-forward service/myapp 8080:80
kubectl proxy                             # Запустить proxy
```

### Удаление

```bash
kubectl delete pod mypod                  # Удалить под
kubectl delete -f manifest.yaml           # Удалить по манифесту
kubectl delete deployment myapp
kubectl delete service myapp
kubectl delete namespace myns
kubectl delete all --all                  # Удалить всё (осторожно!)
kubectl delete pods --all -n namespace
```

### Контекст и конфигурация

```bash
kubectl config view                       # Показать конфиг
kubectl config get-contexts               # Список контекстов
kubectl config use-context my-context     # Переключить контекст
kubectl config current-context            # Текущий контекст
kubectl config set-context my-context --namespace=myns
kubectl config delete-context my-context
```

---

## Helm (менеджер пакетов для K8s)

### Основные команды

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm search repo nginx
helm install myrelease stable/nginx
helm install myrelease ./chart
helm install myrelease stable/nginx --set replicaCount=3
helm install myrelease stable/nginx -f values.yaml
helm list
helm status myrelease
helm get values myrelease
helm get manifest myrelease
helm upgrade myrelease stable/nginx --version 1.2.3
helm upgrade myrelease stable/nginx -f values.yaml
helm rollback myrelease 1
helm uninstall myrelease
helm create mychart
helm lint ./mychart
helm package ./mychart
helm template myrelease ./mychart
```

---

## Лучшие практики

### Pod Security

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL
```

### Resource Management

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Labels

```yaml
labels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/instance: myapp-1
  app.kubernetes.io/version: "1.0.0"
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: myproject
  app.kubernetes.io/managed-by: helm
```

---

## Решение проблем

```bash
# Проверить статус подов
kubectl get pods --all-namespaces

# Описать проблемный под
kubectl describe pod problematic-pod

# Проверить логи
kubectl logs problematic-pod --previous

# Проверить события
kubectl get events --sort-by='.lastTimestamp'

# Проверить ресурсы узла
kubectl top nodes
kubectl describe node node-name

# Проверить Pending поды
kubectl get pods --field-selector=status.phase=Pending

# Проверить OOMKilled
kubectl get pods -o json | jq '.items[] | select(.status.containerStatuses[].lastState.terminated.reason=="OOMKilled")'

# Debug pod без shell
kubectl debug -it mypod --image=busybox --target=mycontainer
```

---

## 🔗 Связанные заметки

- [[Docker-Cheatsheet]] — контейнеризация
- [[Terraform-Cheatsheet]] — Infrastructure as Code
- [[MOC-Infrastructure]] — инфраструктура
- [[CI-CD-Pipeline]] — CI/CD с Kubernetes
