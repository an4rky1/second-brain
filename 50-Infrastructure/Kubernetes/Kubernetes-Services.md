---
created: 2026-02-17
tags:
  - kubernetes
  - k8s
  - services
  - networking
aliases:
  - Kubernetes Services
  - K8s Networking
related:
  - Kubernetes-Cheatsheet
  - Kubernetes-Deployments
  - MOC-Infrastructure
---

# Kubernetes — Services & Networking

> [!SUMMARY] Обзор
> Kubernetes Services: ClusterIP, NodePort, LoadBalancer, Ingress, Network Policies.

---

## 📡 Service Types

### ClusterIP (Default)

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 3000
      protocol: TCP
  sessionAffinity: ClientIP  # Sticky sessions
```

**Use case:** Internal communication between pods

```bash
# Access from within cluster
kubectl get svc my-app
# ClusterIP: 10.96.100.50

# From another pod
curl http://my-app:80
curl http://my-app.default.svc.cluster.local:80
```

---

### NodePort

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 3000
      nodePort: 30080  # 30000-32767
```

**Use case:** External access for development

```bash
# Access from outside
kubectl get svc my-app
# NODE-PORT: 30080

# Access via any node IP
curl http://<node-ip>:30080
```

---

### LoadBalancer

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 3000
  loadBalancerIP: 1.2.3.4  # Optional: static IP
  loadBalancerSourceRanges:
    - 10.0.0.0/8  # Restrict access
```

**Use case:** Production external access (cloud)

```bash
kubectl get svc my-app
# EXTERNAL-IP: abc123.elb.amazonaws.com
```

---

### Headless Service

```yaml
# service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  clusterIP: None  # Headless
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
```

**Use case:** Direct pod access, stateful sets

```bash
# DNS returns all pod IPs
nslookup my-app
# my-app.default.svc.cluster.local
# addresses: 10.244.1.5 10.244.2.6 10.244.3.7
```

---

## 🌐 Ingress

### Basic Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
```

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Check ingress
kubectl get ingress
kubectl describe ingress my-app
```

---

### Path-Based Routing

```yaml
# ingress-paths.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 8081
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

---

### TLS Termination

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
        - api.myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
    - host: api.myapp.example.com
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

---

## 🔒 Network Policies

### Default Deny All

```yaml
# network-policy-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow Specific Traffic

```yaml
# network-policy-allow.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-traffic
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 8080
```

### Allow Egress

```yaml
# network-policy-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 5432  # PostgreSQL
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 443  # HTTPS external
```

---

## 🔍 Common Commands

```bash
# List services
kubectl get services
kubectl get svc

# Describe service
kubectl describe svc/my-app

# List ingress
kubectl get ingress
kubectl get ing

# Describe ingress
kubectl describe ing/my-app

# List network policies
kubectl get networkpolicy
kubectl get netpol

# Test connectivity
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# Inside debug pod:
curl http://my-app:80
nslookup my-app
```

---

## 🔗 Связанные заметки

- [[Kubernetes-Cheatsheet]] — Kubernetes overview
- [[Kubernetes-Deployments]] — Deployments
- [[Docker-Cheatsheet]] — Docker
