# Week 1 Capstone Project: 2-Tier E-Commerce Application

## Project Overview

Build and deploy a complete 2-tier e-commerce application with:
- **Frontend**: React-based product catalog
- **Backend**: Node.js API server
- **Full monitoring**: Prometheus + Grafana
- **GitOps deployment**: ArgoCD
- **CI/CD pipeline**: GitHub Actions

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KIND CLUSTER                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        INGRESS (Port 80)                             │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                          │
│                    ┌─────────────┴─────────────┐                           │
│                    │                           │                           │
│                    ▼                           ▼                           │
│  ┌─────────────────────────┐    ┌─────────────────────────┐               │
│  │       FRONTEND          │    │        BACKEND          │               │
│  │                         │    │                         │               │
│  │  - React App            │───►│  - Node.js API          │               │
│  │  - 2 replicas           │    │  - 3 replicas           │               │
│  │  - HPA (2-5)            │    │  - HPA (2-10)           │               │
│  │  - Probes               │    │  - Probes               │               │
│  │  - PDB                  │    │  - PDB                  │               │
│  └─────────────────────────┘    └─────────────────────────┘               │
│                                              │                             │
│                                              │ /metrics                    │
│                                              ▼                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      MONITORING STACK                                │   │
│  │                                                                      │   │
│  │  Prometheus ──► Grafana ──► AlertManager                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          ARGOCD                                      │   │
│  │                                                                      │   │
│  │  Syncs from Git repository automatically                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
week-1-project/
├── kind-config.yaml
├── apps/
│   ├── backend/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   ├── server.js
│   │   └── products.json
│   └── frontend/
│       ├── Dockerfile
│       ├── nginx.conf
│       └── index.html
├── k8s/
│   ├── namespace.yaml
│   ├── backend/
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── pdb.yaml
│   ├── frontend/
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── pdb.yaml
│   └── ingress.yaml
├── monitoring/
│   ├── servicemonitor.yaml
│   └── grafana-dashboard.json
└── argocd/
    └── application.yaml
```

---

## Step 1: Create Kind Cluster

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ecommerce-cluster
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-config.yaml
```

---

## Step 2: Backend Application

### Backend Code

```javascript
// apps/backend/server.js
const express = require('express');
const cors = require('cors');
const promClient = require('prom-client');

const app = express();
const port = process.env.PORT || 3000;

// Prometheus metrics
const collectDefaultMetrics = promClient.collectDefaultMetrics;
collectDefaultMetrics();

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'path', 'status']
});

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'path'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

// Middleware
app.use(cors());
app.use(express.json());

// Request tracking middleware
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestsTotal.inc({ method: req.method, path: req.path, status: res.statusCode });
    httpRequestDuration.observe({ method: req.method, path: req.path }, duration);
  });
  next();
});

// Sample products data
const products = [
  { id: 1, name: 'Laptop', price: 999.99, category: 'Electronics', stock: 50 },
  { id: 2, name: 'Headphones', price: 149.99, category: 'Electronics', stock: 100 },
  { id: 3, name: 'Coffee Maker', price: 79.99, category: 'Home', stock: 30 },
  { id: 4, name: 'Running Shoes', price: 129.99, category: 'Sports', stock: 75 },
  { id: 5, name: 'Backpack', price: 59.99, category: 'Accessories', stock: 200 }
];

// Health endpoints
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.get('/ready', (req, res) => {
  res.json({ status: 'ready', timestamp: new Date().toISOString() });
});

// Metrics endpoint for Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});

// API endpoints
app.get('/api/products', (req, res) => {
  const { category } = req.query;
  let result = products;
  if (category) {
    result = products.filter(p => p.category.toLowerCase() === category.toLowerCase());
  }
  res.json(result);
});

app.get('/api/products/:id', (req, res) => {
  const product = products.find(p => p.id === parseInt(req.params.id));
  if (!product) {
    return res.status(404).json({ error: 'Product not found' });
  }
  res.json(product);
});

app.get('/api/categories', (req, res) => {
  const categories = [...new Set(products.map(p => p.category))];
  res.json(categories);
});

// Info endpoint
app.get('/api/info', (req, res) => {
  res.json({
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
    hostname: process.env.HOSTNAME || 'unknown',
    uptime: process.uptime()
  });
});

app.listen(port, () => {
  console.log(`Backend server running on port ${port}`);
});
```

```json
// apps/backend/package.json
{
  "name": "ecommerce-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "prom-client": "^15.0.0"
  }
}
```

```dockerfile
# apps/backend/Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

---

## Step 3: Frontend Application

```html
<!-- apps/frontend/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>E-Commerce Store</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        header {
            background: white;
            padding: 20px;
            border-radius: 10px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
            margin-bottom: 10px;
        }
        .status {
            display: flex;
            gap: 20px;
            font-size: 14px;
            color: #666;
        }
        .status-item {
            display: flex;
            align-items: center;
            gap: 5px;
        }
        .status-dot {
            width: 10px;
            height: 10px;
            border-radius: 50%;
            background: #10b981;
        }
        .status-dot.error {
            background: #ef4444;
        }
        .filters {
            background: white;
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        .filters select {
            padding: 10px 15px;
            border: 1px solid #ddd;
            border-radius: 5px;
            font-size: 14px;
        }
        .products {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 20px;
        }
        .product-card {
            background: white;
            border-radius: 10px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            transition: transform 0.2s;
        }
        .product-card:hover {
            transform: translateY(-5px);
        }
        .product-name {
            font-size: 18px;
            font-weight: 600;
            color: #333;
            margin-bottom: 10px;
        }
        .product-category {
            font-size: 12px;
            color: #666;
            background: #f3f4f6;
            padding: 4px 8px;
            border-radius: 4px;
            display: inline-block;
            margin-bottom: 10px;
        }
        .product-price {
            font-size: 24px;
            font-weight: 700;
            color: #667eea;
            margin-bottom: 10px;
        }
        .product-stock {
            font-size: 14px;
            color: #10b981;
        }
        .product-stock.low {
            color: #f59e0b;
        }
        .loading {
            text-align: center;
            padding: 40px;
            color: white;
            font-size: 18px;
        }
        .error-message {
            background: #fee2e2;
            color: #dc2626;
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>🛒 E-Commerce Store</h1>
            <div class="status">
                <div class="status-item">
                    <div class="status-dot" id="backend-status"></div>
                    <span>Backend: <span id="backend-text">Checking...</span></span>
                </div>
                <div class="status-item">
                    <span>Pod: <span id="pod-name">-</span></span>
                </div>
                <div class="status-item">
                    <span>Version: <span id="app-version">-</span></span>
                </div>
            </div>
        </header>

        <div class="filters">
            <label for="category">Filter by category: </label>
            <select id="category" onchange="loadProducts()">
                <option value="">All Categories</option>
            </select>
        </div>

        <div id="error-container"></div>
        <div id="products" class="products">
            <div class="loading">Loading products...</div>
        </div>
    </div>

    <script>
        const API_URL = window.location.origin + '/api';

        async function checkBackendHealth() {
            try {
                const response = await fetch(API_URL + '/info');
                const data = await response.json();
                document.getElementById('backend-status').classList.remove('error');
                document.getElementById('backend-text').textContent = 'Connected';
                document.getElementById('pod-name').textContent = data.hostname;
                document.getElementById('app-version').textContent = data.version;
            } catch (error) {
                document.getElementById('backend-status').classList.add('error');
                document.getElementById('backend-text').textContent = 'Disconnected';
            }
        }

        async function loadCategories() {
            try {
                const response = await fetch(API_URL + '/categories');
                const categories = await response.json();
                const select = document.getElementById('category');
                categories.forEach(cat => {
                    const option = document.createElement('option');
                    option.value = cat;
                    option.textContent = cat;
                    select.appendChild(option);
                });
            } catch (error) {
                console.error('Failed to load categories:', error);
            }
        }

        async function loadProducts() {
            const category = document.getElementById('category').value;
            const productsDiv = document.getElementById('products');
            const errorDiv = document.getElementById('error-container');
            
            try {
                const url = category ? `${API_URL}/products?category=${category}` : `${API_URL}/products`;
                const response = await fetch(url);
                const products = await response.json();
                
                errorDiv.innerHTML = '';
                productsDiv.innerHTML = products.map(product => `
                    <div class="product-card">
                        <div class="product-category">${product.category}</div>
                        <div class="product-name">${product.name}</div>
                        <div class="product-price">$${product.price.toFixed(2)}</div>
                        <div class="product-stock ${product.stock < 50 ? 'low' : ''}">
                            ${product.stock} in stock
                        </div>
                    </div>
                `).join('');
            } catch (error) {
                errorDiv.innerHTML = `<div class="error-message">Failed to load products. Please try again.</div>`;
                productsDiv.innerHTML = '';
            }
        }

        // Initialize
        checkBackendHealth();
        loadCategories();
        loadProducts();

        // Refresh health status every 10 seconds
        setInterval(checkBackendHealth, 10000);
    </script>
</body>
</html>
```

```nginx
# apps/frontend/nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }

    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
}
```

```dockerfile
# apps/frontend/Dockerfile
FROM nginx:1.25-alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## Step 4: Kubernetes Manifests

### Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    app: ecommerce
```

### Backend Manifests

```yaml
# k8s/backend/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: ecommerce
data:
  NODE_ENV: "production"
  APP_VERSION: "1.0.0"
  PORT: "3000"
```

```yaml
# k8s/backend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ecommerce
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: backend
          image: ecommerce-backend:1.0.0
          ports:
            - containerPort: 3000
              name: http
          envFrom:
            - configMapRef:
                name: backend-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
```

```yaml
# k8s/backend/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: ecommerce
  labels:
    app: backend
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 3000
      targetPort: 3000
```

```yaml
# k8s/backend/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ecommerce
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

```yaml
# k8s/backend/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
  namespace: ecommerce
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: backend
```

### Frontend Manifests

```yaml
# k8s/frontend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: ecommerce-frontend:1.0.0
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

```yaml
# k8s/frontend/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
  labels:
    app: frontend
spec:
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
```

```yaml
# k8s/frontend/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```yaml
# k8s/frontend/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: ecommerce
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: frontend
```

### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 3000
```

---

## Step 5: Monitoring

```yaml
# monitoring/servicemonitor.yaml
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
      - ecommerce
  endpoints:
    - port: http
      interval: 30s
      path: /metrics
```

---

## Step 6: ArgoCD Application

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/ecommerce-k8s.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Deployment Steps

```bash
# 1. Create Kind cluster
kind create cluster --config kind-config.yaml

# 2. Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# 3. Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# 4. Install Prometheus Stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# 5. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 6. Build and load images into Kind
docker build -t ecommerce-backend:1.0.0 ./apps/backend
docker build -t ecommerce-frontend:1.0.0 ./apps/frontend
kind load docker-image ecommerce-backend:1.0.0 --name ecommerce-cluster
kind load docker-image ecommerce-frontend:1.0.0 --name ecommerce-cluster

# 7. Deploy application
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/backend/
kubectl apply -f k8s/frontend/
kubectl apply -f k8s/ingress.yaml

# 8. Apply monitoring
kubectl apply -f monitoring/servicemonitor.yaml

# 9. Access the application
# Open http://localhost in your browser

# 10. Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Open http://localhost:3000 (admin/prom-operator)

# 11. Access ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Get password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## Verification Checklist

- [ ] All pods running in ecommerce namespace
- [ ] Frontend accessible at http://localhost
- [ ] Backend API responding at http://localhost/api/products
- [ ] HPA configured and responding to load
- [ ] PDBs protecting minimum availability
- [ ] Prometheus scraping backend metrics
- [ ] Grafana dashboards showing data
- [ ] ArgoCD syncing from Git repository

---

## Congratulations!

You've completed the Week 1 Capstone Project! You now have a production-like Kubernetes deployment with:

- Multi-tier application architecture
- Health probes for reliability
- Horizontal Pod Autoscaling
- Pod Disruption Budgets
- Prometheus monitoring
- Grafana dashboards
- GitOps with ArgoCD

You're ready for Week 2: Production-Grade EKS with Terraform!
