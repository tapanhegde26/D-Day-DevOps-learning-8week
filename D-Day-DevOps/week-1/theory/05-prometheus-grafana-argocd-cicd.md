# Week 1, Day 7-8: Prometheus, Grafana, ArgoCD & CI/CD Pipeline

## Monitoring with Prometheus + Grafana

### Why Monitoring Matters

Without monitoring, you're flying blind:
- How do you know if your app is healthy?
- When do you need to scale?
- What caused that outage at 3 AM?

### Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROMETHEUS ECOSYSTEM                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                   │
│  │   Target    │     │   Target    │     │   Target    │                   │
│  │  (Pod/Svc)  │     │  (Pod/Svc)  │     │   (Node)    │                   │
│  │  /metrics   │     │  /metrics   │     │  /metrics   │                   │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘                   │
│         │                   │                   │                           │
│         └───────────────────┼───────────────────┘                           │
│                             │ scrape                                        │
│                             ▼                                               │
│                    ┌─────────────────┐                                      │
│                    │   PROMETHEUS    │                                      │
│                    │                 │                                      │
│                    │  - Scrapes      │                                      │
│                    │  - Stores TSDB  │◄──── PromQL queries                  │
│                    │  - Evaluates    │                                      │
│                    │    rules        │                                      │
│                    └────────┬────────┘                                      │
│                             │                                               │
│              ┌──────────────┼──────────────┐                               │
│              │              │              │                               │
│              ▼              ▼              ▼                               │
│     ┌─────────────┐  ┌───────────┐  ┌─────────────┐                       │
│     │   GRAFANA   │  │ALERTMANAGER│ │  API/CLI    │                       │
│     │             │  │            │  │             │                       │
│     │ Dashboards  │  │ Routes to: │  │ Ad-hoc     │                       │
│     │ Visualize   │  │ - Slack    │  │ queries    │                       │
│     │             │  │ - PagerDuty│  │             │                       │
│     └─────────────┘  └────────────┘  └─────────────┘                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Installing Prometheus Stack on Kind

We'll use the **kube-prometheus-stack** Helm chart, which includes:
- Prometheus
- Grafana
- AlertManager
- Node Exporter
- kube-state-metrics

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install the stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Wait for pods to be ready
kubectl get pods -n monitoring -w

# Verify all components
kubectl get all -n monitoring
```

### Accessing Prometheus

```bash
# Port forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Open http://localhost:9090
```

**Basic PromQL queries:**
```promql
# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Memory usage by pod
sum(container_memory_usage_bytes{namespace="default"}) by (pod)

# HTTP request rate
sum(rate(http_requests_total[5m])) by (service)

# Pod restart count
kube_pod_container_status_restarts_total

# Node CPU usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Accessing Grafana

```bash
# Port forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Open http://localhost:3000
# Login: admin / admin123
```

**Pre-built dashboards included:**
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Pods)
- Kubernetes / Compute Resources / Node (Pods)
- Node Exporter / Nodes

### Creating Custom Dashboards

1. Click "+" → "New Dashboard"
2. Add a new panel
3. Select Prometheus as data source
4. Enter PromQL query
5. Configure visualization

**Example: Pod CPU Usage Panel**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="default", container!=""}[5m])) by (pod)
```

### ServiceMonitor — Scraping Your Apps

To monitor your own applications, create a ServiceMonitor:

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: prometheus  # Must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - default
  endpoints:
    - port: metrics      # Name of the port in Service
      interval: 30s
      path: /metrics
```

**Your app's Service must expose a metrics port:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
    - name: metrics
      port: 9090
```

---

## GitOps with ArgoCD

### What is GitOps?

GitOps is a way of implementing Continuous Deployment where:
- **Git is the single source of truth** for declarative infrastructure
- **Automated processes** sync the desired state in Git to the cluster
- **Changes are made via Git** (pull requests, reviews, audit trail)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GITOPS WORKFLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Developer                                                                  │
│      │                                                                       │
│      │ 1. Push code                                                         │
│      ▼                                                                       │
│  ┌─────────┐    2. CI builds     ┌─────────┐    3. Push image              │
│  │  Code   │ ──────────────────► │   CI    │ ──────────────────►           │
│  │  Repo   │                     │ Pipeline│                    │           │
│  └─────────┘                     └─────────┘                    │           │
│                                       │                         │           │
│                                       │ 4. Update               ▼           │
│                                       │    image tag    ┌─────────────┐    │
│                                       │                 │  Container  │    │
│                                       ▼                 │  Registry   │    │
│                                  ┌─────────┐            └─────────────┘    │
│                                  │ Config  │                    │           │
│                                  │  Repo   │◄───────────────────┘           │
│                                  │ (GitOps)│    5. ArgoCD                   │
│                                  └────┬────┘       detects                  │
│                                       │            change                   │
│                                       │                                     │
│                                       ▼                                     │
│                                  ┌─────────┐                                │
│                                  │ ArgoCD  │                                │
│                                  │         │                                │
│                                  └────┬────┘                                │
│                                       │                                     │
│                                       │ 6. Sync to cluster                  │
│                                       ▼                                     │
│                                  ┌─────────┐                                │
│                                  │   K8s   │                                │
│                                  │ Cluster │                                │
│                                  └─────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Installing ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open https://localhost:8080
# Login: admin / <password from above>
```

### Installing ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login via CLI
argocd login localhost:8080 --insecure
```

### Creating an ArgoCD Application

**Via CLI:**
```bash
argocd app create nginx-app \
  --repo https://github.com/your-org/k8s-manifests.git \
  --path nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

**Via YAML (recommended):**
```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/k8s-manifests.git
    targetRevision: HEAD
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true      # Delete resources removed from Git
      selfHeal: true   # Revert manual changes in cluster
    syncOptions:
      - CreateNamespace=true
```

### ArgoCD Sync Policies

| Policy | Behavior |
|--------|----------|
| Manual | Requires manual sync via UI/CLI |
| Automated | Syncs automatically when Git changes |
| Prune | Deletes resources removed from Git |
| Self-Heal | Reverts manual cluster changes |

### ArgoCD with Helm

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 15.0.0
    helm:
      values: |
        replicaCount: 3
        service:
          type: ClusterIP
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

### ArgoCD with Kustomize

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-kustomize
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/k8s-manifests.git
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

---

## CI/CD Pipeline with GitHub Actions

### Complete Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CI/CD PIPELINE                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      GITHUB ACTIONS (CI)                             │   │
│  │                                                                      │   │
│  │  1. Checkout ──► 2. Test ──► 3. Build ──► 4. Push ──► 5. Update    │   │
│  │     code           app        image       to registry   manifests   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                      │                      │
│                                                      ▼                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         ARGOCD (CD)                                  │   │
│  │                                                                      │   │
│  │  6. Detect ──► 7. Sync ──► 8. Deploy ──► 9. Health ──► 10. Done    │   │
│  │     change      to cluster   pods         check                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### GitHub Actions Workflow

Create `.github/workflows/ci-cd.yaml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Run linting
        run: npm run lint

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  update-manifests:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: Checkout manifests repo
        uses: actions/checkout@v4
        with:
          repository: your-org/k8s-manifests
          token: ${{ secrets.MANIFEST_REPO_TOKEN }}
          path: manifests

      - name: Update image tag
        run: |
          cd manifests
          # Using kustomize
          cd overlays/production
          kustomize edit set image app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          
          # Or using sed for plain YAML
          # sed -i "s|image:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|" deployment.yaml

      - name: Commit and push
        run: |
          cd manifests
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Update image to ${{ github.sha }}"
          git push
```

### Repository Structure

**Application Repository:**
```
app-repo/
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
├── src/
│   └── ...
├── tests/
│   └── ...
├── Dockerfile
├── package.json
└── README.md
```

**Manifests Repository (GitOps):**
```
k8s-manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   └── production/
│       ├── kustomization.yaml
│       └── patches/
└── argocd/
    └── application.yaml
```

### Kustomize Base

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: app:latest  # Will be overridden
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
---
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### Kustomize Overlay (Production)

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

replicas:
  - name: app
    count: 3

images:
  - name: app
    newName: ghcr.io/your-org/app
    newTag: abc123  # Updated by CI

patches:
  - path: patches/resources.yaml
---
# overlays/production/patches/resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

---

## Week 1 Capstone Project

### Project: 2-Tier E-Commerce App

Build and deploy a complete 2-tier application with:
- Frontend (React/Vue/static)
- Backend API (Node.js/Python/Go)
- Full monitoring
- GitOps deployment

### Project Structure

```
week-1-project/
├── app/
│   ├── frontend/
│   │   ├── Dockerfile
│   │   └── ...
│   └── backend/
│       ├── Dockerfile
│       └── ...
├── k8s/
│   ├── base/
│   │   ├── frontend-deployment.yaml
│   │   ├── frontend-service.yaml
│   │   ├── backend-deployment.yaml
│   │   ├── backend-service.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       └── dev/
│           ├── kustomization.yaml
│           └── hpa.yaml
├── monitoring/
│   ├── servicemonitor.yaml
│   └── grafana-dashboard.json
├── argocd/
│   └── application.yaml
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
└── kind-config.yaml
```

### Step-by-Step Implementation

**1. Create Kind Cluster:**
```bash
kind create cluster --config kind-config.yaml --name week1-project
```

**2. Install Monitoring Stack:**
```bash
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

**3. Install ArgoCD:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**4. Deploy Application via ArgoCD:**
```bash
kubectl apply -f argocd/application.yaml
```

**5. Configure HPA:**
```yaml
# k8s/overlays/dev/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**6. Add Pod Disruption Budget:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: backend
```

**7. Configure Probes:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Week 1 Summary

### What You've Learned

1. **Kubernetes Architecture** — Control plane, worker nodes, how components interact
2. **Kind** — Local multi-node clusters for development
3. **Core Objects** — Pods, Deployments, Services, ReplicaSets
4. **Configuration** — ConfigMaps, Secrets, resource management
5. **Health** — Liveness, readiness, startup probes
6. **Scaling** — HPA, VPA, Pod Disruption Budgets
7. **Deployments** — Rolling updates, blue-green, canary
8. **Debugging** — Common issues and troubleshooting techniques
9. **Monitoring** — Prometheus + Grafana stack
10. **GitOps** — ArgoCD for continuous deployment
11. **CI/CD** — GitHub Actions pipeline

### Key Commands Cheat Sheet

```bash
# Cluster
kind create cluster --config config.yaml
kubectl cluster-info
kubectl get nodes

# Workloads
kubectl apply -f manifest.yaml
kubectl get pods,svc,deploy
kubectl describe pod <name>
kubectl logs <pod> [-f] [--previous]
kubectl exec -it <pod> -- /bin/bash

# Scaling
kubectl scale deployment <name> --replicas=5
kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10

# Rollouts
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Debugging
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods
kubectl describe node <name>

# Port forwarding
kubectl port-forward svc/<name> 8080:80
```

### Interview Questions to Practice

1. Explain the Kubernetes architecture and the role of each component.
2. What's the difference between a Deployment and a StatefulSet?
3. How does a Service route traffic to Pods?
4. Explain the difference between liveness and readiness probes.
5. How does HPA work? What metrics can it use?
6. What is GitOps and why is it beneficial?
7. How would you debug a pod stuck in CrashLoopBackOff?
8. Explain the rolling update strategy and its parameters.

---

## Next Week Preview: Production-Grade EKS with Terraform

In Week 2, we'll move to AWS and learn:
- Terraform modules for EKS
- Helm chart development
- IRSA (IAM Roles for Service Accounts)
- Database migrations with Jobs
- Production RBAC and security
