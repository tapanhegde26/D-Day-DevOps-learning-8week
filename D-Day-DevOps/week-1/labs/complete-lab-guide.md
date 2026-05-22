# Week 1 Lab: Complete Hands-On Exercises

This lab guide walks you through all the practical exercises for Week 1.

## Prerequisites

Ensure you have installed:
- Docker Desktop
- kubectl
- kind
- helm

## Lab 1: Setting Up a Multi-Node Kind Cluster

### Step 1: Create the Kind Configuration

```bash
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: week1-cluster
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
      - containerPort: 30000
        hostPort: 30000
        protocol: TCP
  - role: worker
  - role: worker
EOF
```

### Step 2: Create the Cluster

```bash
kind create cluster --config kind-config.yaml

# Verify the cluster
kubectl cluster-info
kubectl get nodes
```

### Step 3: Explore the Cluster

```bash
# See all system pods
kubectl get pods -n kube-system

# Describe a node
kubectl describe node week1-cluster-control-plane

# Check Docker containers (Kind nodes are Docker containers)
docker ps
```

---

## Lab 2: Deploying Your First Application

### Step 1: Create a Simple Pod

```bash
cat > simple-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: lab
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "200m"
EOF

kubectl apply -f simple-pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
```

### Step 2: Create a Deployment

```bash
cat > nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
EOF

kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods -l app=nginx
kubectl get replicasets
```

### Step 3: Create a Service

```bash
cat > nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF

kubectl apply -f nginx-service.yaml
kubectl get svc nginx-service
kubectl get endpoints nginx-service
```

### Step 4: Test the Service

```bash
# Test from within the cluster
kubectl run curl-test --image=curlimages/curl -it --rm -- curl http://nginx-service

# Port forward to access locally
kubectl port-forward svc/nginx-service 8080:80 &
curl http://localhost:8080
```

---

## Lab 3: ConfigMaps and Secrets

### Step 1: Create a ConfigMap

```bash
cat > app-configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /health {
            return 200 'healthy';
            add_header Content-Type text/plain;
        }
    }
EOF

kubectl apply -f app-configmap.yaml
kubectl describe configmap app-config
```

### Step 2: Create a Secret

```bash
# Create secret imperatively
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='SuperSecret123!'

# Or declaratively (values must be base64 encoded)
cat > db-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-yaml
type: Opaque
stringData:
  username: admin
  password: SuperSecret123!
EOF

kubectl apply -f db-secret.yaml
kubectl get secrets
kubectl describe secret db-credentials
```

### Step 3: Use ConfigMap and Secret in a Pod

```bash
cat > app-with-config.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ['sh', '-c', 'echo "Config loaded" && env && sleep 3600']
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      envFrom:
        - configMapRef:
            name: app-config
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: default.conf
EOF

kubectl apply -f app-with-config.yaml
kubectl exec app-with-config -- env | grep -E 'DB_|APP_|LOG_|DATABASE_'
kubectl exec app-with-config -- cat /etc/nginx/conf.d/default.conf
```

---

## Lab 4: Health Probes

### Step 1: Deploy App with Probes

```bash
cat > app-with-probes.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-probes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probes-demo
  template:
    metadata:
      labels:
        app: probes-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
EOF

kubectl apply -f app-with-probes.yaml
kubectl get pods -l app=probes-demo -w
```

### Step 2: Simulate Probe Failure

```bash
# Get a pod name
POD_NAME=$(kubectl get pods -l app=probes-demo -o jsonpath='{.items[0].metadata.name}')

# Delete the index.html to make probes fail
kubectl exec $POD_NAME -- rm /usr/share/nginx/html/index.html

# Watch the pod become not ready and eventually restart
kubectl get pods -l app=probes-demo -w

# Check events
kubectl describe pod $POD_NAME | tail -20
```

---

## Lab 5: Resource Management and Quotas

### Step 1: Create a Namespace with Quota

```bash
kubectl create namespace quota-lab

cat > resource-quota.yaml << 'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-quota
  namespace: quota-lab
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "5"
EOF

kubectl apply -f resource-quota.yaml
kubectl describe resourcequota lab-quota -n quota-lab
```

### Step 2: Create LimitRange

```bash
cat > limit-range.yaml << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: quota-lab
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "500m"
        memory: "512Mi"
EOF

kubectl apply -f limit-range.yaml
```

### Step 3: Test Quota Limits

```bash
# Create pods until quota is hit
for i in {1..6}; do
  kubectl run test-pod-$i --image=nginx -n quota-lab
  echo "Created pod $i"
  kubectl get resourcequota lab-quota -n quota-lab
done

# The 6th pod should fail due to quota
kubectl get pods -n quota-lab
```

---

## Lab 6: Horizontal Pod Autoscaler

### Step 1: Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for Kind (disable TLS verification)
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Wait for metrics server
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=120s

# Test metrics
kubectl top nodes
kubectl top pods -A
```

### Step 2: Deploy HPA Test Application

```bash
cat > hpa-demo.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo
spec:
  selector:
    app: hpa-demo
  ports:
    - port: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
EOF

kubectl apply -f hpa-demo.yaml
kubectl get hpa
```

### Step 3: Generate Load and Watch Scaling

```bash
# In terminal 1: Watch HPA
kubectl get hpa hpa-demo -w

# In terminal 2: Watch pods
kubectl get pods -l app=hpa-demo -w

# In terminal 3: Generate load
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hpa-demo; done"

# After a few minutes, you should see pods scale up
# Stop the load generator (Ctrl+C) and watch pods scale down
```

---

## Lab 7: Deployment Strategies

### Step 1: Rolling Update

```bash
cat > rolling-update-demo.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: rolling-demo
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
          ports:
            - containerPort: 80
EOF

kubectl apply -f rolling-update-demo.yaml
kubectl get pods -l app=rolling-demo -w
```

### Step 2: Trigger Rolling Update

```bash
# In terminal 1: Watch pods
kubectl get pods -l app=rolling-demo -w

# In terminal 2: Trigger update
kubectl set image deployment/rolling-demo nginx=nginx:1.25

# Watch the rolling update happen
# Notice: never goes below 4 pods (maxUnavailable=0)
```

### Step 3: Rollback

```bash
# View rollout history
kubectl rollout history deployment/rolling-demo

# Rollback to previous version
kubectl rollout undo deployment/rolling-demo

# Verify
kubectl get pods -l app=rolling-demo -o jsonpath='{.items[*].spec.containers[0].image}'
```

---

## Lab 8: Monitoring with Prometheus and Grafana

### Step 1: Install Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Wait for all pods
kubectl get pods -n monitoring -w
```

### Step 2: Access Prometheus

```bash
# Port forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Open http://localhost:9090 in browser
# Try these queries:
# - up
# - kube_pod_status_phase
# - container_memory_usage_bytes
```

### Step 3: Access Grafana

```bash
# Port forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &

# Open http://localhost:3000
# Login: admin / admin123
# Explore pre-built dashboards under "Dashboards" menu
```

---

## Lab 9: GitOps with ArgoCD

### Step 1: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w

# Get admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD Password: $ARGOCD_PASSWORD"
```

### Step 2: Access ArgoCD UI

```bash
# Port forward ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Open https://localhost:8080
# Login: admin / <password from above>
```

### Step 3: Create a Sample Application

```bash
# Create a sample GitOps repo structure locally
mkdir -p gitops-demo/nginx
cat > gitops-demo/nginx/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-gitops
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-gitops
  template:
    metadata:
      labels:
        app: nginx-gitops
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
EOF

cat > gitops-demo/nginx/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-gitops
spec:
  selector:
    app: nginx-gitops
  ports:
    - port: 80
EOF

# For a real GitOps setup, push this to a Git repository
# Then create an ArgoCD Application pointing to it
```

### Step 4: Create ArgoCD Application (using public example)

```bash
cat > argocd-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl apply -f argocd-app.yaml

# Check the application in ArgoCD UI
# It should automatically sync and deploy the guestbook app
kubectl get pods -l app=guestbook-ui
```

---

## Lab 10: Debugging Practice

### Step 1: Debug ImagePullBackOff

```bash
# Create a pod with wrong image
kubectl run broken-image --image=nginx:nonexistent-tag

# Debug
kubectl get pods broken-image
kubectl describe pod broken-image | grep -A 10 "Events"

# Fix: Update to correct image
kubectl set image pod/broken-image broken-image=nginx:1.25
```

### Step 2: Debug CrashLoopBackOff

```bash
# Create a crashing pod
kubectl run crasher --image=busybox -- /bin/sh -c "exit 1"

# Debug
kubectl get pods crasher
kubectl describe pod crasher
kubectl logs crasher --previous

# The pod exits with code 1, causing CrashLoopBackOff
kubectl delete pod crasher
```

### Step 3: Debug OOMKilled

```bash
# Create a pod that will be OOMKilled
cat > oom-demo.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
    - name: stress
      image: polinux/stress
      resources:
        limits:
          memory: "50Mi"
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
EOF

kubectl apply -f oom-demo.yaml

# Watch it get OOMKilled
kubectl get pods oom-demo -w

# Debug
kubectl describe pod oom-demo | grep -A 5 "Last State"
```

---

## Cleanup

```bash
# Delete all resources created in labs
kubectl delete -f .
kubectl delete namespace quota-lab
kubectl delete namespace monitoring
kubectl delete namespace argocd

# Delete the Kind cluster
kind delete cluster --name week1-cluster
```

---

## Summary

You've completed all Week 1 hands-on labs covering:

1. ✅ Kind cluster setup
2. ✅ Pods, Deployments, Services
3. ✅ ConfigMaps and Secrets
4. ✅ Health Probes
5. ✅ Resource Quotas and Limits
6. ✅ Horizontal Pod Autoscaler
7. ✅ Deployment Strategies
8. ✅ Prometheus and Grafana
9. ✅ ArgoCD GitOps
10. ✅ Debugging techniques

You're now ready for the Week 1 capstone project!
