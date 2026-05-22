# Week 4: Logging, Monitoring, Secret Management & SRE

## Overview

This week covers the complete observability stack, secret management with Vault, and SRE practices including SLOs, SLIs, and error budgets.

---

## Day 1-2: Advanced Prometheus & Loki

### Prometheus Operator Deep Dive

```yaml
# ServiceMonitor for custom application
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: backend-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: backend
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
      scrapeTimeout: 10s
      honorLabels: true
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: 'go_.*'
          action: drop
---
# PodMonitor for pods without service
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-jobs
  namespace: monitoring
spec:
  selector:
    matchLabels:
      type: batch
  namespaceSelector:
    any: true
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

### PrometheusRule for Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backend-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: backend.rules
      rules:
        # High error rate
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
            > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }}"
        
        # High latency
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99, 
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
            ) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High latency on {{ $labels.service }}"
            description: "P99 latency is {{ $value }}s"
        
        # Pod restarts
        - alert: PodRestartingTooMuch
          expr: |
            increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} restarting frequently"
```

### Loki for Log Aggregation

```yaml
# Loki installation with Helm
# values-loki.yaml
loki:
  auth_enabled: false
  storage:
    type: s3
    s3:
      endpoint: s3.amazonaws.com
      bucketnames: loki-logs-bucket
      region: us-west-2
  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: s3
        schema: v12
        index:
          prefix: loki_index_
          period: 24h

promtail:
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
```

### LogQL Queries

```logql
# Find errors in backend
{app="backend"} |= "error"

# Parse JSON logs
{app="backend"} | json | level="error"

# Count errors per minute
sum(rate({app="backend"} |= "error" [1m])) by (pod)

# Extract and filter
{app="backend"} 
  | json 
  | status_code >= 500 
  | line_format "{{.method}} {{.path}} {{.status_code}}"

# Latency from logs
{app="backend"} 
  | json 
  | unwrap duration 
  | quantile_over_time(0.99, [5m])
```

---

## Day 3-4: Distributed Tracing & AlertManager

### OpenTelemetry Setup

```yaml
# OpenTelemetry Collector
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
    
    processors:
      batch:
        timeout: 10s
      memory_limiter:
        check_interval: 1s
        limit_mib: 1000
    
    exporters:
      otlp:
        endpoint: tempo:4317
        tls:
          insecure: true
      prometheus:
        endpoint: 0.0.0.0:8889
    
    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger]
          processors: [memory_limiter, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus]
```

### AlertManager Configuration

```yaml
# AlertManager config
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/xxx'
    
    route:
      group_by: ['alertname', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'default'
      routes:
        - match:
            severity: critical
          receiver: 'pagerduty-critical'
          continue: true
        - match:
            severity: critical
          receiver: 'slack-critical'
        - match:
            severity: warning
          receiver: 'slack-warning'
    
    receivers:
      - name: 'default'
        slack_configs:
          - channel: '#alerts-default'
            send_resolved: true
      
      - name: 'slack-critical'
        slack_configs:
          - channel: '#alerts-critical'
            send_resolved: true
            title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      
      - name: 'slack-warning'
        slack_configs:
          - channel: '#alerts-warning'
            send_resolved: true
      
      - name: 'pagerduty-critical'
        pagerduty_configs:
          - service_key: 'your-pagerduty-key'
            severity: critical
    
    inhibit_rules:
      - source_match:
          severity: 'critical'
        target_match:
          severity: 'warning'
        equal: ['alertname', 'namespace']
```

---

## Day 5-6: HashiCorp Vault & External Secrets

### Vault on Kubernetes

```bash
# Install Vault with Helm
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  --set server.dev.enabled=true  # Dev mode for learning
```

```yaml
# Production Vault values
server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
  
  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: gp3
  
  auditStorage:
    enabled: true
    size: 10Gi

injector:
  enabled: true
  
ui:
  enabled: true
```

### Vault Agent Injector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "backend"
        vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/production/database"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "secret/data/production/database" -}}
          export DATABASE_URL="postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/app"
          {{- end }}
    spec:
      serviceAccountName: backend
      containers:
        - name: backend
          command: ["/bin/sh", "-c"]
          args:
            - source /vault/secrets/db-creds && ./app
```

### Vault Dynamic Secrets

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/app" \
  allowed_roles="backend-role" \
  username="vault-admin" \
  password="admin-password"

# Create role for dynamic credentials
vault write database/roles/backend-role \
  db_name=postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

### External Secrets Operator

```yaml
# ClusterSecretStore for Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
# ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

---

## Day 7-8: SRE Practices

### SLO/SLI/SLA Framework

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SRE HIERARCHY                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SLA (Service Level Agreement)                                              │
│  └── External commitment to customers                                       │
│      "99.9% availability per month"                                         │
│                                                                              │
│  SLO (Service Level Objective)                                              │
│  └── Internal target (stricter than SLA)                                    │
│      "99.95% availability per month"                                        │
│                                                                              │
│  SLI (Service Level Indicator)                                              │
│  └── Actual measurement                                                     │
│      "Successful requests / Total requests"                                 │
│                                                                              │
│  Error Budget = 100% - SLO                                                  │
│  └── "0.05% of requests can fail per month"                                │
│      = 21.6 minutes of downtime allowed                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Defining SLIs

```yaml
# PrometheusRule for SLI recording
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sli-rules
  namespace: monitoring
spec:
  groups:
    - name: sli.rules
      interval: 30s
      rules:
        # Availability SLI
        - record: sli:availability:ratio
          expr: |
            sum(rate(http_requests_total{status!~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
        
        # Latency SLI (requests under 200ms)
        - record: sli:latency:ratio
          expr: |
            sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m])) by (service)
            /
            sum(rate(http_request_duration_seconds_count[5m])) by (service)
        
        # Error budget remaining (30-day window)
        - record: sli:error_budget:remaining
          expr: |
            1 - (
              (1 - sli:availability:ratio) 
              / 
              (1 - 0.999)  # SLO target
            )
```

### SLO Dashboard in Grafana

```json
{
  "panels": [
    {
      "title": "Availability SLO",
      "type": "gauge",
      "targets": [
        {
          "expr": "sli:availability:ratio{service=\"backend\"} * 100",
          "legendFormat": "Current"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              {"color": "red", "value": 0},
              {"color": "yellow", "value": 99},
              {"color": "green", "value": 99.9}
            ]
          },
          "min": 95,
          "max": 100,
          "unit": "percent"
        }
      }
    },
    {
      "title": "Error Budget Remaining",
      "type": "stat",
      "targets": [
        {
          "expr": "sli:error_budget:remaining{service=\"backend\"} * 100"
        }
      ]
    },
    {
      "title": "Error Budget Burn Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "1 - sli:availability:ratio{service=\"backend\"}",
          "legendFormat": "Error Rate"
        }
      ]
    }
  ]
}
```

### Burn Rate Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts
spec:
  groups:
    - name: slo.alerts
      rules:
        # Fast burn - 2% of monthly budget in 1 hour
        - alert: SLOBurnRateFast
          expr: |
            (
              (1 - sli:availability:ratio) / (1 - 0.999)
            ) > 14.4
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Fast error budget burn on {{ $labels.service }}"
            description: "Burning 2% of monthly error budget per hour"
        
        # Slow burn - 5% of monthly budget in 6 hours
        - alert: SLOBurnRateSlow
          expr: |
            (
              (1 - sli:availability:ratio) / (1 - 0.999)
            ) > 6
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "Slow error budget burn on {{ $labels.service }}"
```

### Kubecost for Cost Visibility

```bash
# Install Kubecost
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace \
  --set kubecostToken="your-token"
```

---

## Week 4 Capstone

Build complete observability stack:
- Prometheus with custom ServiceMonitors
- Loki for centralized logging
- Grafana dashboards with SLO tracking
- AlertManager with Slack/PagerDuty
- Vault for dynamic secrets
- Error budget dashboards
- Cost visibility with Kubecost

---

## Key Takeaways

1. **Prometheus Operator** simplifies monitoring configuration
2. **Loki** provides cost-effective log aggregation
3. **OpenTelemetry** standardizes distributed tracing
4. **Vault** provides dynamic, rotating secrets
5. **SLOs** define reliability targets
6. **Error budgets** balance reliability and velocity
7. **Burn rate alerts** catch issues before SLO breach

---

## Next Week: Scaling, StatefulSets, Storage

Week 5 covers Karpenter, KEDA, and stateful workloads.
