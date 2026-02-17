---
created: 2026-02-17
tags:
  - kubernetes
  - helm
  - charts
  - package-manager
aliases:
  - Helm Charts
  - Kubernetes Helm
related:
  - Kubernetes-Cheatsheet
  - Kubernetes-Deployments
  - MOC-Infrastructure
---

# Helm — Charts & Package Manager

> [!SUMMARY] Обзор
> Helm — пакетный менеджер для Kubernetes. Charts, templates, releases, repositories.

---

## 📦 Установка

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm

# Verify
helm version
```

---

## 🏗️ Chart Structure

```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── values-prod.yaml    # Production overrides
├── .helmignore         # Files to ignore
├── charts/             # Sub-charts (dependencies)
│   └── postgresql/
└── templates/          # Kubernetes manifests
    ├── _helpers.tpl    # Template helpers
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── pvc.yaml
    └── tests/
        └── test-connection.yaml
```

---

## 📝 Chart.yaml

```yaml
apiVersion: v2  # Helm 3
name: my-app
description: My Application Helm Chart
type: application
version: 1.0.0  # Chart version
appVersion: "1.0.0"  # Application version
keywords:
  - my-app
  - web
home: https://github.com/myorg/my-app
sources:
  - https://github.com/myorg/my-app
maintainers:
  - name: John Doe
    email: john@example.com
dependencies:
  - name: postgresql
    version: "12.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

---

## 📋 values.yaml

```yaml
# Default values
replicaCount: 3

image:
  repository: myorg/my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  name: ""
  annotations: {}

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}

# Dependencies
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: "changeme"

redis:
  enabled: true
  auth:
    password: "changeme"
```

---

## 📄 Templates

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### _helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## 🚀 Commands

```bash
# Create new chart
helm create my-app

# Lint chart
helm lint ./my-app

# Validate templates
helm template my-app ./my-app

# Install
helm install my-release ./my-app
helm install my-release ./my-app -f values-prod.yaml
helm install my-release ./my-app --set replicaCount=5

# Upgrade
helm upgrade my-release ./my-app
helm upgrade my-release ./my-app -f values-prod.yaml
helm upgrade my-release ./my-app --set image.tag=2.0.0

# Uninstall
helm uninstall my-release

# List releases
helm list
helm list -n my-namespace

# History
helm history my-release

# Rollback
helm rollback my-release 1  # Rollback to revision 1

# Status
helm status my-release

# Get values
helm get values my-release
helm get values my-release --revision 2

# Diff (requires helm-diff plugin)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-release ./my-app -f values-prod.yaml

# Repository management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami
helm repo remove bitnami

# Package chart
helm package ./my-app
helm push my-app-1.0.0.tgz oci://registry.example.com/charts
```

---

## 🔗 Связанные заметки

- [[Kubernetes-Cheatsheet]] — Kubernetes overview
- [[Kubernetes-Deployments]] — Deployments
- [[Docker-Cheatsheet]] — Docker
