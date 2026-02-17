---
created: 2026-02-17
tags:
  - kubernetes
  - k8s
  - deployments
  - pods
aliases:
  - Kubernetes Deployments
  - K8s Pods Deployments
related:
  - Kubernetes-Cheatsheet
  - Docker-Cheatsheet
  - MOC-Infrastructure
---

# Kubernetes — Deployments & Pods

> [!SUMMARY] Обзор
> Kubernetes Deployments: управление подами, репликация, rolling updates, health checks.

---

## 🏗️ Pod (Минимальная единица)

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
    version: v1
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      ports:
        - containerPort: 3000
      env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db_host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db_password
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
          cpu: "500m"
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
      volumeMounts:
        - name: data
          mountPath: /app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-pvc
```

---

## 📦 Deployment (Production)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: my-app
                topologyKey: kubernetes.io/hostname
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Exists"
          effect: "NoSchedule"
```

---

## 🔄 Rolling Update

```bash
# Update image
kubectl set image deployment/my-app app=myapp:2.0.0

# Or edit deployment
kubectl edit deployment/my-app

# Watch rollout status
kubectl rollout status deployment/my-app

# View rollout history
kubectl rollout history deployment/my-app

# Rollback to previous
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# Pause rollout
kubectl rollout pause deployment/my-app

# Resume rollout
kubectl rollout resume deployment/my-app
```

---

## 📊 Scaling

```bash
# Manual scaling
kubectl scale deployment/my-app --replicas=5

# Auto-scaling (HPA)
kubectl autoscale deployment/my-app \
  --min=2 \
  --max=10 \
  --cpu-percent=80

# View HPA
kubectl get hpa

# Vertical scaling (VPA)
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vpa-v0.13.0.yaml
```

### HPA Manifest

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
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

## 🔍 Health Checks

```yaml
# Probes configuration
livenessProbe:
  httpGet:
    path: /health
    port: 3000
    httpHeaders:
      - name: Authorization
        value: "Bearer token"
  initialDelaySeconds: 30   # Wait before first probe
  periodSeconds: 10         # How often to probe
  timeoutSeconds: 5         # Timeout for probe
  successThreshold: 1       # Min consecutive successes
  failureThreshold: 3       # Min consecutive fails before restart

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3       # Remove from service after 3 fails

startupProbe:  # For slow-starting containers
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30      # 30 * 10s = 5 minutes
  periodSeconds: 10
```

---

## 🎯 Common Commands

```bash
# List deployments
kubectl get deployments
kubectl get deploy

# Describe deployment
kubectl describe deployment/my-app

# Get pods
kubectl get pods
kubectl get pods -l app=my-app

# Describe pod
kubectl describe pod/my-pod

# View logs
kubectl logs my-pod
kubectl logs -f my-pod              # Follow
kubectl logs -f my-pod -c app       # Specific container
kubectl logs -f my-pod --since=1h   # Last hour

# Execute command in pod
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -- npm run db:migrate

# Copy files
kubectl cp my-pod:/app/logs ./logs
kubectl cp ./config my-pod:/app/config

# Delete pod (auto-recreated by deployment)
kubectl delete pod/my-pod
kubectl delete pod -l app=my-app

# Delete deployment
kubectl delete deployment/my-app
```

---

## 🔗 Связанные заметки

- [[Kubernetes-Cheatsheet]] — Kubernetes overview
- [[Docker-Cheatsheet]] — Docker
- [[MOC-Infrastructure]] — Infrastructure MOC
