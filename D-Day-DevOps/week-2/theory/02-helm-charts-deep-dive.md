# Week 2, Day 3-4: Helm Charts Deep Dive

## What is Helm?

Helm is the **package manager for Kubernetes**. It helps you:
- Package applications as reusable charts
- Manage application releases
- Handle complex deployments with dependencies
- Template Kubernetes manifests

## Helm Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HELM WORKFLOW                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────────────────────┐  │
│   │   Chart     │     │   Values    │     │      Kubernetes API         │  │
│   │  Templates  │ ──► │   + Data    │ ──► │                             │  │
│   │             │     │             │     │   Rendered Manifests        │  │
│   └─────────────┘     └─────────────┘     └─────────────────────────────┘  │
│                                                                              │
│   Chart Repository                                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  bitnami/nginx    prometheus-community/kube-prometheus-stack        │  │
│   │  ingress-nginx    cert-manager    external-dns    ...               │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Helm Chart Structure

```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── values.schema.json  # JSON schema for values validation (optional)
├── charts/             # Dependency charts
├── templates/          # Kubernetes manifest templates
│   ├── NOTES.txt       # Post-install notes
│   ├── _helpers.tpl    # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   └── serviceaccount.yaml
├── crds/               # Custom Resource Definitions (optional)
└── README.md           # Documentation
```

---

## Creating a Helm Chart

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: ecommerce-backend
description: A Helm chart for the e-commerce backend service
type: application
version: 1.0.0        # Chart version
appVersion: "1.0.0"   # Application version

keywords:
  - ecommerce
  - backend
  - api

maintainers:
  - name: DevOps Team
    email: devops@company.com

dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### values.yaml

```yaml
# values.yaml - Default configuration

# Number of replicas
replicaCount: 3

# Image configuration
image:
  repository: ghcr.io/company/ecommerce-backend
  tag: "1.0.0"
  pullPolicy: IfNotPresent

# Image pull secrets for private registries
imagePullSecrets: []

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Pod annotations
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "3000"
  prometheus.io/path: "/metrics"

# Pod security context
podSecurityContext:
  fsGroup: 1000

# Container security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 3000

# Ingress configuration
ingress:
  enabled: false
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

# Resource limits
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity
affinity: {}

# Health probes
probes:
  liveness:
    enabled: true
    path: /health
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  readiness:
    enabled: true
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3
  startup:
    enabled: true
    path: /health
    failureThreshold: 30
    periodSeconds: 2

# Environment variables
env:
  NODE_ENV: production
  LOG_LEVEL: info

# Environment variables from secrets/configmaps
envFrom: []

# ConfigMap data
config:
  APP_NAME: "E-Commerce Backend"

# Secrets (will be base64 encoded)
secrets: {}

# External secrets (reference existing secrets)
externalSecrets:
  enabled: false
  secretName: ""

# Database configuration
database:
  host: ""
  port: 5432
  name: ecommerce
  existingSecret: ""
  secretKeys:
    username: username
    password: password

# PostgreSQL subchart
postgresql:
  enabled: false
  auth:
    database: ecommerce
    username: ecommerce

# Redis subchart
redis:
  enabled: false
  architecture: standalone
```

---

## Template Files

### _helpers.tpl

```yaml
{{/* templates/_helpers.tpl */}}

{{/*
Expand the name of the chart.
*/}}
{{- define "ecommerce-backend.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "ecommerce-backend.fullname" -}}
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
{{- define "ecommerce-backend.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "ecommerce-backend.labels" -}}
helm.sh/chart: {{ include "ecommerce-backend.chart" . }}
{{ include "ecommerce-backend.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "ecommerce-backend.selectorLabels" -}}
app.kubernetes.io/name: {{ include "ecommerce-backend.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "ecommerce-backend.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "ecommerce-backend.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Database host
*/}}
{{- define "ecommerce-backend.databaseHost" -}}
{{- if .Values.postgresql.enabled }}
{{- printf "%s-postgresql" .Release.Name }}
{{- else }}
{{- .Values.database.host }}
{{- end }}
{{- end }}
```

### deployment.yaml

```yaml
{{/* templates/deployment.yaml */}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "ecommerce-backend.fullname" . }}
  labels:
    {{- include "ecommerce-backend.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "ecommerce-backend.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "ecommerce-backend.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "ecommerce-backend.serviceAccountName" . }}
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
          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: http
            failureThreshold: {{ .Values.probes.startup.failureThreshold }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
          {{- end }}
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          {{- end }}
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            - name: DATABASE_HOST
              value: {{ include "ecommerce-backend.databaseHost" . | quote }}
            - name: DATABASE_PORT
              value: {{ .Values.database.port | quote }}
            - name: DATABASE_NAME
              value: {{ .Values.database.name | quote }}
            {{- if or .Values.database.existingSecret .Values.postgresql.enabled }}
            - name: DATABASE_USERNAME
              valueFrom:
                secretKeyRef:
                  {{- if .Values.postgresql.enabled }}
                  name: {{ .Release.Name }}-postgresql
                  key: username
                  {{- else }}
                  name: {{ .Values.database.existingSecret }}
                  key: {{ .Values.database.secretKeys.username }}
                  {{- end }}
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  {{- if .Values.postgresql.enabled }}
                  name: {{ .Release.Name }}-postgresql
                  key: password
                  {{- else }}
                  name: {{ .Values.database.existingSecret }}
                  key: {{ .Values.database.secretKeys.password }}
                  {{- end }}
            {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
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

### service.yaml

```yaml
{{/* templates/service.yaml */}}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "ecommerce-backend.fullname" . }}
  labels:
    {{- include "ecommerce-backend.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "ecommerce-backend.selectorLabels" . | nindent 4 }}
```

### hpa.yaml

```yaml
{{/* templates/hpa.yaml */}}
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "ecommerce-backend.fullname" . }}
  labels:
    {{- include "ecommerce-backend.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "ecommerce-backend.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### ingress.yaml

```yaml
{{/* templates/ingress.yaml */}}
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "ecommerce-backend.fullname" . }}
  labels:
    {{- include "ecommerce-backend.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "ecommerce-backend.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

---

## Helm Hooks for Database Migrations

```yaml
{{/* templates/migration-job.yaml */}}
{{- if .Values.migrations.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "ecommerce-backend.fullname" . }}-migration
  labels:
    {{- include "ecommerce-backend.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        {{- include "ecommerce-backend.selectorLabels" . | nindent 8 }}
    spec:
      restartPolicy: Never
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command: ['sh', '-c', 'until nc -z {{ include "ecommerce-backend.databaseHost" . }} {{ .Values.database.port }}; do echo waiting for database; sleep 2; done']
      containers:
        - name: migration
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          command: ["npm", "run", "migrate"]
          env:
            - name: DATABASE_HOST
              value: {{ include "ecommerce-backend.databaseHost" . | quote }}
            - name: DATABASE_PORT
              value: {{ .Values.database.port | quote }}
            - name: DATABASE_NAME
              value: {{ .Values.database.name | quote }}
            - name: DATABASE_USERNAME
              valueFrom:
                secretKeyRef:
                  {{- if .Values.postgresql.enabled }}
                  name: {{ .Release.Name }}-postgresql
                  key: username
                  {{- else }}
                  name: {{ .Values.database.existingSecret }}
                  key: {{ .Values.database.secretKeys.username }}
                  {{- end }}
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  {{- if .Values.postgresql.enabled }}
                  name: {{ .Release.Name }}-postgresql
                  key: password
                  {{- else }}
                  name: {{ .Values.database.existingSecret }}
                  key: {{ .Values.database.secretKeys.password }}
                  {{- end }}
{{- end }}
```

### Helm Hook Types

| Hook | When it runs |
|------|--------------|
| `pre-install` | Before any resources are installed |
| `post-install` | After all resources are installed |
| `pre-delete` | Before any resources are deleted |
| `post-delete` | After all resources are deleted |
| `pre-upgrade` | Before any resources are upgraded |
| `post-upgrade` | After all resources are upgraded |
| `pre-rollback` | Before any resources are rolled back |
| `post-rollback` | After all resources are rolled back |

### Hook Delete Policies

| Policy | Behavior |
|--------|----------|
| `before-hook-creation` | Delete previous hook resource before new hook is launched |
| `hook-succeeded` | Delete hook resource after hook succeeds |
| `hook-failed` | Delete hook resource if hook fails |

---

## Umbrella Charts

An umbrella chart bundles multiple charts together.

```yaml
# umbrella-chart/Chart.yaml
apiVersion: v2
name: ecommerce-platform
description: Complete e-commerce platform
version: 1.0.0
type: application

dependencies:
  - name: ecommerce-frontend
    version: "1.x.x"
    repository: "file://../ecommerce-frontend"
  - name: ecommerce-backend
    version: "1.x.x"
    repository: "file://../ecommerce-backend"
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# umbrella-chart/values.yaml
# Global values available to all subcharts
global:
  environment: production
  domain: example.com

# Frontend subchart values
ecommerce-frontend:
  replicaCount: 2
  ingress:
    enabled: true
    hosts:
      - host: www.example.com
        paths:
          - path: /
            pathType: Prefix

# Backend subchart values
ecommerce-backend:
  replicaCount: 3
  ingress:
    enabled: true
    hosts:
      - host: api.example.com
        paths:
          - path: /
            pathType: Prefix

# PostgreSQL values
postgresql:
  enabled: true
  auth:
    database: ecommerce
    username: ecommerce

# Redis values
redis:
  enabled: true
  architecture: standalone
```

---

## Helm Commands Reference

```bash
# Create a new chart
helm create my-chart

# Lint chart
helm lint ./my-chart

# Template locally (see rendered manifests)
helm template my-release ./my-chart -f values.yaml

# Dry run install
helm install my-release ./my-chart --dry-run --debug

# Install chart
helm install my-release ./my-chart -f values.yaml -n namespace

# Upgrade release
helm upgrade my-release ./my-chart -f values.yaml -n namespace

# Install or upgrade
helm upgrade --install my-release ./my-chart -f values.yaml -n namespace

# List releases
helm list -A

# Get release status
helm status my-release -n namespace

# Get release history
helm history my-release -n namespace

# Rollback to previous revision
helm rollback my-release 1 -n namespace

# Uninstall release
helm uninstall my-release -n namespace

# Package chart
helm package ./my-chart

# Update dependencies
helm dependency update ./my-chart

# Search hub
helm search hub nginx

# Search repo
helm search repo bitnami/nginx

# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## Values Hierarchy

Values are merged in this order (later overrides earlier):

1. Parent chart's `values.yaml`
2. Subchart's `values.yaml`
3. `-f` or `--values` files (in order specified)
4. `--set` parameters

```bash
# Multiple values files
helm install my-release ./my-chart \
  -f values.yaml \
  -f values-production.yaml \
  --set image.tag=v2.0.0
```

---

## Best Practices

1. **Always use helpers** for names and labels
2. **Validate values** with JSON schema
3. **Use semantic versioning** for charts
4. **Document values** in values.yaml with comments
5. **Test charts** with `helm lint` and `helm template`
6. **Use hooks** for migrations and setup tasks
7. **Keep charts focused** — one application per chart
8. **Use umbrella charts** for complex deployments

---

## Next: IRSA, Load Balancing, and DNS

In the next section, we'll configure AWS integrations for our EKS cluster.
