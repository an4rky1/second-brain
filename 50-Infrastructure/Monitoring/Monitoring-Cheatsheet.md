---
created: 2026-02-16
tags:
  - cheat-sheet
  - monitoring
  - observability
  - prometheus
  - grafana
aliases:
  - Monitoring Cheatsheet
  - Observability Reference
related:
  - Kubernetes-Cheatsheet
  - Docker-Cheatsheet
  - Logging-Best-Practices
---

# Monitoring & Observability — Полная шпаргалка

> [!SUMMARY] Обзор
> Мониторинг и наблюдаемость систем. Prometheus, Grafana, AlertManager, tracing, logging. Три столпа observability: metrics, logs, traces.

---

## 📚 Теория

### Три столпа Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    Observability                                 │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Metrics    │  │    Logs      │  │   Traces     │          │
│  │  (Что?)      │  │  (Почему?)   │  │  (Где?)      │          │
│  │              │  │              │  │              │          │
│  │  Prometheus  │  │   ELK/Loki   │  │  Jaeger/     │          │
│  │  Grafana     │  │   Splunk     │  │  Tempo       │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### RED и USE методы

```
RED Method (для сервисов):
├── Rate — запросов в секунду
├── Errors — количество ошибок
└── Duration — время ответа

USE Method (для ресурсов):
├── Utilization — % использования
├── Saturation — очередь задач
└── Errors — ошибки оборудования
```

### Уровни мониторинга

```
┌─────────────────────────────────────────────────────┐
│              Business Metrics                        │
│  (конверсии, пользователи, заказы)                  │
└─────────────────────────────────────────────────────┘
                      │
┌─────────────────────▼─────────────────────────────┐
│              Application Metrics                   │
│  (RPS, latency, error rate, throughput)            │
└─────────────────────────────────────────────────────┘
                      │
┌─────────────────────▼─────────────────────────────┐
│              Infrastructure Metrics                │
│  (CPU, memory, disk, network)                      │
└─────────────────────────────────────────────────────┘
```

---

## ⚡ Быстрый старт

```bash
# Prometheus + Grafana stack
docker-compose up -d prometheus grafana alertmanager

# Helm (Kubernetes)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack

# Node Exporter (хост метрики)
docker run -d \
  --name=node-exporter \
  --pid="host" \
  --net="host" \
  -v /:/host:ro,rslave \
  prom/node-exporter:latest \
  --path.rootfs=/host
```

---

## 🔧 Практические примеры

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    environment: 'prod'

# Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Rules
rule_files:
  - 'rules/*.yml'

# Scrape configs
scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          service: 'prometheus'

  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'node_.*'
        action: keep

  # Application
  - job_name: 'app'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['app:8080']
    
  # Kubernetes service discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

  # Kubernetes nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

### Alert Rules

```yaml
# rules/alerts.yml
groups:
  - name: application
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
          
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s (threshold: 1s)"
          
      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"

  - name: infrastructure
    rules:
      # High CPU
      - alert: HighCPU
        expr: |
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value }}% on {{ $labels.instance }}"
          
      # High Memory
      - alert: HighMemory
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}% on {{ $labels.instance }}"
          
      # Disk Space
      - alert: DiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space"
          description: "Disk space is {{ $value }}% on {{ $labels.instance }}"
```

### AlertManager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'
  
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
  
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
      
    - match:
        severity: critical
      receiver: 'slack-critical'
      
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Grafana Dashboards

```json
{
  "dashboard": {
    "title": "Application Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m]))",
            "legendFormat": "Requests/s"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "legendFormat": "Error %"
          }
        ]
      },
      {
        "title": "Latency (P50, P95, P99)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P99"
          }
        ]
      },
      {
        "title": "CPU Usage",
        "type": "gauge",
        "targets": [
          {
            "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
          }
        ]
      }
    ]
  }
}
```

### Application Metrics

```typescript
// Node.js + Prometheus Client
import client from 'prom-client';

// Collect default metrics
client.collectDefaultMetrics();

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

const activeConnections = new client.Gauge({
  name: 'http_active_connections',
  help: 'Number of active HTTP connections',
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  activeConnections.inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || 'unknown', res.statusCode)
      .observe(duration);
    
    httpRequestTotal
      .labels(req.method, req.route?.path || 'unknown', res.statusCode)
      .inc();
    
    activeConnections.dec();
  });
  
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

```go
// Go + Prometheus Client
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequests = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "status"},
    )
    
    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method"},
    )
)

func init() {
    prometheus.MustRegister(httpRequests, httpDuration)
}

func metricsHandler() http.Handler {
    return promhttp.Handler()
}

// Usage
func handler(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    
    // Handle request...
    
    httpDuration
        .WithLabelValues(r.Method)
        .Observe(time.Since(start).Seconds())
    
    httpRequests
        .WithLabelValues(r.Method, "200")
        .Inc()
}
```

### Logging (Loki)

```yaml
# loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h  # 7 days
```

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: container-logs
          __path__: /var/lib/docker/containers/*/*log
    
    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs:
      - json:
          expressions:
            tag:
          source: attrs
      - regex:
          expression: (?P<container_name>(?:|(?:[^|]*[^|])))
          source: tag
      - labels:
          - container_name
          - stream
```

### Distributed Tracing (Jaeger)

```yaml
# docker-compose.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14268:14268"  # Collector
      - "6831:6831/udp"  # Agent
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  # Application with tracing
  app:
    build: .
    environment:
      - JAEGER_ENDPOINT=http://jaeger:14268/api/traces
      - OTEL_EXPORTER_JAEGER_ENDPOINT=http://jaeger:4317
```

```typescript
// OpenTelemetry (Node.js)
import { NodeTracerProvider } from '@opentelemetry/node';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';

const provider = new NodeTracerProvider();
provider.register();

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation({
      applyCustomAttributesOnSpan: (span, request, response) => {
        span.setAttribute('custom-attribute', 'value');
      },
    }),
  ],
});

const tracer = provider.getTracer('my-app');

// Manual tracing
const span = tracer.startSpan('my-operation');
try {
  // Do something...
  span.setAttribute('key', 'value');
} catch (err) {
  span.recordException(err);
  throw err;
} finally {
  span.end();
}
```

---

## 🎯 Best Practices

### ✅ Делать

```yaml
# 1. Правильные buckets для histogram
buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

# 2. Информативные labels
labels: ['service', 'endpoint', 'method', 'status']

# 3. Разумные интервалы
scrape_interval: 15s  # Не слишком часто
evaluation_interval: 15s

# 4. Alert с for clause
for: 5m  # Избегает ложных срабатываний

# 5. Runbook в annotations
annotations:
  summary: "..."
  description: "..."
  runbook_url: "https://wiki.example.com/runbooks/alert-name"
```

### ❌ Не делать

```yaml
# 1. High cardinality labels
labels: ['user_id']  # ❌ Миллионы уникальных значений

# 2. Слишком частый scrape
scrape_interval: 1s  # ❌

# 3. Alerts без for
for: 0s  # ❌ Ложные срабатывания

# 4. Игнорирование recording rules
# Используйте для сложных запросов
```

---

## 🔗 Связанные заметки

- [[Kubernetes-Cheatsheet]] — K8s мониторинг
- [[Docker-Cheatsheet]] — Docker метрики
- [[Logging-Best-Practices]] — Логирование

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **4 Golden Signals** — latency, traffic, errors, saturation
> 2. **Alert на симптомы** — не на причины
> 3. **Runbooks** — документация для каждого алерта
> 4. **SLO/SLI** — определяйте целевые показатели
> 5. **Tracing в production** — sampling для экономии

> [!INFO] Полезные инструменты
> ```bash
> # Prometheus
> # https://prometheus.io/
> 
> # Grafana
> # https://grafana.com/
> 
> # Loki (логи)
> # https://grafana.com/oss/loki/
> 
> # Jaeger (tracing)
> # https://www.jaegertracing.io/
> 
> # Tempo (Grafana tracing)
> # https://grafana.com/oss/tempo/
> 
> # Mimir (long-term storage)
> # https://grafana.com/oss/mimir/
> ```
