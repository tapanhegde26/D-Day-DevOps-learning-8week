# Week 3: Microservices, Service Mesh, mTLS & API Gateway

## Overview

This week focuses on complex microservices patterns, service mesh implementation with Istio, and advanced traffic management.

## Learning Objectives

- Understand microservices communication patterns
- Install and configure Istio service mesh
- Implement mTLS for zero-trust security
- Configure traffic management (canary, blue-green)
- Use Gateway API for ingress
- Implement Network Policies
- Set up ArgoCD App-of-Apps pattern

---

## Day 1-2: Microservices Patterns & Istio Fundamentals

### Why Microservices on Kubernetes?

**Benefits:**
- Independent deployment and scaling
- Technology diversity
- Fault isolation
- Team autonomy

**Challenges:**
- Service discovery complexity
- Distributed tracing
- Network reliability
- Security between services

### Service Communication Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES COMMUNICATION                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Synchronous (Request/Response)                                          │
│     ┌─────────┐  HTTP/gRPC  ┌─────────┐                                    │
│     │Service A│────────────►│Service B│                                    │
│     └─────────┘             └─────────┘                                    │
│                                                                              │
│  2. Asynchronous (Event-Driven)                                             │
│     ┌─────────┐   publish   ┌─────────┐  subscribe  ┌─────────┐           │
│     │Service A│────────────►│  Queue  │────────────►│Service B│           │
│     └─────────┘             └─────────┘             └─────────┘           │
│                                                                              │
│  3. Service Mesh (Sidecar Proxy)                                            │
│     ┌─────────────────┐         ┌─────────────────┐                        │
│     │ ┌─────┐ ┌─────┐ │  mTLS   │ ┌─────┐ ┌─────┐ │                        │
│     │ │ App │ │Envoy│─┼────────►┼─│Envoy│ │ App │ │                        │
│     │ └─────┘ └─────┘ │         │ └─────┘ └─────┘ │                        │
│     └─────────────────┘         └─────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Istio Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ISTIO ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Control Plane (istiod)                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────────┐    │   │
│  │  │  Pilot  │  │ Citadel │  │  Galley │  │  Sidecar Injector   │    │   │
│  │  │         │  │         │  │         │  │                     │    │   │
│  │  │ Config  │  │  mTLS   │  │ Config  │  │ Injects Envoy into  │    │   │
│  │  │ & Disc. │  │  Certs  │  │ Valid.  │  │ pods automatically  │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    │ xDS API                                │
│                                    ▼                                        │
│  Data Plane (Envoy Sidecars)                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │   │
│  │  │ Pod A           │  │ Pod B           │  │ Pod C           │     │   │
│  │  │ ┌─────┐ ┌─────┐ │  │ ┌─────┐ ┌─────┐ │  │ ┌─────┐ ┌─────┐ │     │   │
│  │  │ │ App │ │Envoy│ │  │ │ App │ │Envoy│ │  │ │ App │ │Envoy│ │     │   │
│  │  │ └─────┘ └─────┘ │  │ └─────┘ └─────┘ │  │ └─────┘ └─────┘ │     │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Installing Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install with demo profile (includes all features)
istioctl install --set profile=demo -y

# Or production profile (minimal, secure)
istioctl install --set profile=default -y

# Enable sidecar injection for namespace
kubectl label namespace default istio-injection=enabled

# Verify installation
kubectl get pods -n istio-system
istioctl analyze
```

### Istio Profiles

| Profile | Use Case | Components |
|---------|----------|------------|
| `default` | Production | istiod, ingress gateway |
| `demo` | Learning/Testing | All components |
| `minimal` | Control plane only | istiod only |
| `empty` | Custom builds | Nothing |

---

## Day 3-4: mTLS & Traffic Management

### mTLS (Mutual TLS)

mTLS ensures both client and server authenticate each other.

```yaml
# Enable strict mTLS for entire mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# Enable mTLS for specific namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# Permissive mode (allows both mTLS and plaintext)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: legacy
spec:
  mtls:
    mode: PERMISSIVE
```

### Virtual Services

Control how requests are routed to services.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend
  namespace: production
spec:
  hosts:
    - backend
  http:
    # Route based on headers
    - match:
        - headers:
            x-version:
              exact: "v2"
      route:
        - destination:
            host: backend
            subset: v2
    # Default route
    - route:
        - destination:
            host: backend
            subset: v1
          weight: 90
        - destination:
            host: backend
            subset: v2
          weight: 10
```

### Destination Rules

Define policies for traffic after routing.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend
  namespace: production
spec:
  host: backend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### Canary Deployment with Istio

```yaml
# VirtualService for canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend
spec:
  hosts:
    - backend
  http:
    - route:
        - destination:
            host: backend
            subset: stable
          weight: 95
        - destination:
            host: backend
            subset: canary
          weight: 5
---
# DestinationRule defining subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend
spec:
  host: backend
  subsets:
    - name: stable
      labels:
        version: v1.0.0
    - name: canary
      labels:
        version: v1.1.0
```

### Circuit Breaking

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend
spec:
  host: backend
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5      # Eject after 5 consecutive 5xx
      interval: 10s                # Check every 10s
      baseEjectionTime: 30s        # Eject for 30s minimum
      maxEjectionPercent: 50       # Max 50% of hosts ejected
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
      tcp:
        maxConnections: 100
```

### Retries and Timeouts

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend
spec:
  hosts:
    - backend
  http:
    - route:
        - destination:
            host: backend
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
```

---

## Day 5-6: Gateway API & Network Policies

### Gateway API (Kubernetes Native)

Gateway API is the evolution of Ingress, providing more features.

```yaml
# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: tls-secret
      allowedRoutes:
        namespaces:
          from: All
---
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-route
  namespace: production
spec:
  parentRefs:
    - name: main-gateway
      namespace: istio-system
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/v1
      backendRefs:
        - name: backend-v1
          port: 80
          weight: 90
        - name: backend-v2
          port: 80
          weight: 10
    - matches:
        - path:
            type: PathPrefix
            value: /api/v2
      backendRefs:
        - name: backend-v2
          port: 80
```

### Network Policies

Control pod-to-pod communication at the network level.

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow traffic from frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
---
# Allow traffic from specific namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
---
# Egress policy - only allow specific destinations
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Allow database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
```

---

## Day 7-8: ArgoCD App-of-Apps & Observability

### ArgoCD App-of-Apps Pattern

Manage multiple applications with a single parent application.

```yaml
# apps/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# apps/frontend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: services/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### ApplicationSet for Multiple Environments

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - service: frontend
                - service: backend
                - service: orders
                - service: payments
          - list:
              elements:
                - env: dev
                  namespace: development
                - env: staging
                  namespace: staging
                - env: prod
                  namespace: production
  template:
    metadata:
      name: '{{service}}-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops-repo.git
        targetRevision: HEAD
        path: 'services/{{service}}/overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Kiali for Mesh Visualization

```bash
# Install Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# Access Kiali dashboard
kubectl port-forward svc/kiali -n istio-system 20001:20001
# Open http://localhost:20001
```

---

## Week 3 Capstone Project

Deploy a microservices e-commerce application with:
- 5+ microservices (frontend, backend, orders, payments, inventory)
- Istio service mesh with mTLS
- Canary deployments for backend
- Gateway API ingress
- Network policies for isolation
- ArgoCD App-of-Apps GitOps

### Architecture

```
Internet → Gateway → Frontend
                  → Backend API → Orders Service
                               → Payments Service
                               → Inventory Service
                               → PostgreSQL
```

---

## Key Takeaways

1. **Service Mesh** adds observability, security, and traffic control
2. **mTLS** provides zero-trust security between services
3. **Virtual Services** control routing, **Destination Rules** control policies
4. **Gateway API** is the future of Kubernetes ingress
5. **Network Policies** provide defense in depth
6. **App-of-Apps** scales GitOps for microservices

---

## Next Week: Logging, Monitoring, Secret Management, SRE

Week 4 covers the full observability stack and SRE practices.
