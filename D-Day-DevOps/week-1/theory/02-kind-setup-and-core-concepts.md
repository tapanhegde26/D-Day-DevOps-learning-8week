# Week 1, Day 1-2: Setting Up Kind & Core Kubernetes Concepts

## What is Kind?

**Kind** (Kubernetes IN Docker) runs Kubernetes clusters using Docker containers as "nodes."

**Why Kind for learning?**
- Runs entirely on your laptop (no cloud costs)
- Multi-node clusters in seconds
- Perfect for testing and CI/CD
- Mirrors real cluster behavior

**Kind vs Alternatives:**
| Tool | Best For | Nodes | Resource Usage |
|------|----------|-------|----------------|
| Kind | Learning, CI/CD | Multi-node | Medium |
| Minikube | Single-node testing | Single | Low |
| k3d | Lightweight K3s | Multi-node | Low |
| Docker Desktop | Quick start | Single | Medium |

---

## Installing Prerequisites

### 1. Install Docker

**macOS:**
```bash
# Install Docker Desktop from https://docker.com/products/docker-desktop
# Or via Homebrew:
brew install --cask docker
```

**Linux (Ubuntu/Debian):**
```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to docker group (logout/login required)
sudo usermod -aG docker $USER
```

### 2. Install kubectl

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Verify:**
```bash
kubectl version --client
```

### 3. Install Kind

**macOS:**
```bash
brew install kind
```

**Linux:**
```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Verify:**
```bash
kind version
```

---

## Creating Your First Kind Cluster

### Simple Single-Node Cluster

```bash
# Create a cluster named "my-cluster"
kind create cluster --name my-cluster

# This creates:
# - A Docker container running as a Kubernetes node
# - Control plane components inside that container
# - Configures kubectl to connect to it
```

**Verify:**
```bash
# Check cluster info
kubectl cluster-info

# List nodes
kubectl get nodes

# See the Docker container
docker ps
```

### Multi-Node Cluster (Production-Like)

Create a config file `kind-config.yaml`:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: dev-cluster
nodes:
  # Control plane node
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      # Map port 80 on host to node for ingress
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      # Map port 443 on host to node for ingress
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  # Worker nodes
  - role: worker
  - role: worker
  - role: worker
```

```bash
# Create the multi-node cluster
kind create cluster --config kind-config.yaml

# Verify nodes
kubectl get nodes
# NAME                        STATUS   ROLES           AGE   VERSION
# dev-cluster-control-plane   Ready    control-plane   1m    v1.27.3
# dev-cluster-worker          Ready    <none>          1m    v1.27.3
# dev-cluster-worker2         Ready    <none>          1m    v1.27.3
# dev-cluster-worker3         Ready    <none>          1m    v1.27.3
```

### Managing Kind Clusters

```bash
# List all clusters
kind get clusters

# Delete a cluster
kind delete cluster --name dev-cluster

# Get kubeconfig for a cluster
kind get kubeconfig --name dev-cluster

# Load a local Docker image into the cluster
kind load docker-image my-app:latest --name dev-cluster
```

---

## Core Kubernetes Concepts

### 1. Pods — The Smallest Deployable Unit

A **Pod** is one or more containers that:
- Share the same network namespace (same IP)
- Share the same storage volumes
- Are scheduled together on the same node

**Why not just containers?**
- Pods provide a higher-level abstraction
- Co-located containers can communicate via localhost
- Sidecar pattern: main app + helper containers

```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

```bash
# Create the pod
kubectl apply -f simple-pod.yaml

# Check pod status
kubectl get pods
kubectl get pods -o wide  # Shows node and IP

# Describe pod (detailed info)
kubectl describe pod nginx-pod

# View pod logs
kubectl logs nginx-pod

# Execute command in pod
kubectl exec -it nginx-pod -- /bin/bash

# Delete pod
kubectl delete pod nginx-pod
```

**Pod Lifecycle States:**
```
Pending → Running → Succeeded/Failed
              ↓
          Unknown (node lost)
```

### 2. ReplicaSets — Ensuring Pod Copies

A **ReplicaSet** ensures a specified number of pod replicas are running.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
```

**Key insight**: You rarely create ReplicaSets directly. Deployments manage them for you.

### 3. Deployments — The Standard Way to Deploy

A **Deployment** manages ReplicaSets and provides:
- Declarative updates
- Rolling updates and rollbacks
- Scaling
- Pause/resume deployments

```yaml
# deployment.yaml
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
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods over desired during update
      maxUnavailable: 0  # Max pods unavailable during update
```

```bash
# Create deployment
kubectl apply -f deployment.yaml

# Check deployment status
kubectl get deployments
kubectl rollout status deployment/nginx-deployment

# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# View rollout history
kubectl rollout history deployment/nginx-deployment

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### 4. Services — Exposing Your Applications

Pods are ephemeral — they get new IPs when recreated. **Services** provide stable networking.

#### ClusterIP (Default) — Internal Only
```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP  # Default, can be omitted
  selector:
    app: nginx     # Routes to pods with this label
  ports:
    - protocol: TCP
      port: 80        # Service port
      targetPort: 80  # Container port
```

```bash
# Create service
kubectl apply -f service-clusterip.yaml

# Get service details
kubectl get svc nginx-service
# NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.45.123   <none>        80/TCP    1m

# Access from within cluster
kubectl run curl --image=curlimages/curl -it --rm -- curl http://nginx-service
```

#### NodePort — External Access via Node IP
```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080  # Optional: K8s assigns if not specified (30000-32767)
```

```bash
# Access via any node's IP on port 30080
curl http://<node-ip>:30080
```

#### LoadBalancer — Cloud Load Balancer
```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Note**: LoadBalancer only works in cloud environments (AWS, GCP, Azure) or with MetalLB locally.

#### Service Discovery

Kubernetes provides DNS-based service discovery:

```bash
# Within the same namespace
curl http://nginx-service

# Across namespaces
curl http://nginx-service.default.svc.cluster.local

# Format: <service-name>.<namespace>.svc.cluster.local
```

---

## kubectl — Your Command Line Tool

### Imperative vs Declarative

**Imperative** (quick commands, not reproducible):
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
kubectl scale deployment nginx --replicas=3
```

**Declarative** (YAML files, version controlled, reproducible):
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

**Best Practice**: Use declarative for anything beyond quick testing.

### Essential kubectl Commands

```bash
# Get resources
kubectl get pods
kubectl get pods -A                    # All namespaces
kubectl get pods -o wide               # More details
kubectl get pods -o yaml               # Full YAML output
kubectl get pods -w                    # Watch for changes
kubectl get all                        # Pods, services, deployments, etc.

# Describe (detailed info + events)
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>  # Specific container
kubectl logs <pod-name> -f              # Follow/stream logs
kubectl logs <pod-name> --previous      # Previous container instance

# Execute commands
kubectl exec <pod-name> -- ls /app
kubectl exec -it <pod-name> -- /bin/bash

# Port forwarding (access pod locally)
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80

# Copy files
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl cp ./local-file <pod-name>:/path/to/file

# Delete resources
kubectl delete pod <pod-name>
kubectl delete -f deployment.yaml
kubectl delete deployment --all

# Apply changes
kubectl apply -f file.yaml
kubectl apply -f directory/          # Apply all YAML in directory
kubectl apply -k ./kustomize-dir/    # Apply with Kustomize

# Diff before applying
kubectl diff -f deployment.yaml
```

### Useful Aliases

Add to your `~/.bashrc` or `~/.zshrc`:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kga='kubectl get all'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'

# Enable kubectl autocompletion
source <(kubectl completion bash)  # or zsh
```

---

## Understanding YAML Manifests

Every Kubernetes object has four required fields:

```yaml
apiVersion: v1              # API version for this resource type
kind: Pod                   # Type of resource
metadata:                   # Data about the resource
  name: my-pod
  namespace: default
  labels:
    app: my-app
spec:                       # Desired state specification
  containers:
    - name: my-container
      image: nginx:1.25
```

### Finding API Versions

```bash
# List all API resources
kubectl api-resources

# Get API version for a resource
kubectl api-resources | grep -i deployment
# deployments    deploy    apps/v1    true    Deployment

# Explain a resource
kubectl explain deployment
kubectl explain deployment.spec
kubectl explain deployment.spec.strategy
```

---

## Hands-On Lab: Deploy Your First Application

### Lab 1: Create a Multi-Node Cluster

```bash
# Create kind-config.yaml as shown above, then:
kind create cluster --config kind-config.yaml --name lab-cluster

# Verify
kubectl get nodes
kubectl cluster-info
```

### Lab 2: Deploy nginx with Deployment and Service

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# Expose as ClusterIP service
kubectl expose deployment nginx --port=80 --target-port=80

# Verify
kubectl get all

# Test connectivity
kubectl run curl --image=curlimages/curl -it --rm -- curl http://nginx

# Clean up
kubectl delete deployment nginx
kubectl delete service nginx
```

### Lab 3: Deploy from YAML Files

Create `nginx-app.yaml`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```bash
# Apply
kubectl apply -f nginx-app.yaml

# Verify
kubectl get all -l app=nginx

# Scale
kubectl scale deployment nginx --replicas=5

# Update image
kubectl set image deployment/nginx nginx=nginx:1.26

# Watch rollout
kubectl rollout status deployment/nginx

# Rollback
kubectl rollout undo deployment/nginx

# Clean up
kubectl delete -f nginx-app.yaml
```

---

## Practice Exercises

1. Create a Kind cluster with 1 control plane and 2 workers
2. Deploy a pod running `busybox` that sleeps for 1 hour
3. Create a deployment with 3 replicas of `httpd:2.4`
4. Expose the httpd deployment as a NodePort service
5. Scale the deployment to 5 replicas
6. Update the image to `httpd:2.4.57` and watch the rollout
7. Rollback to the previous version
8. Delete all resources you created

---

## Key Takeaways

1. **Kind** is perfect for local Kubernetes development
2. **Pods** are the smallest unit, but you rarely create them directly
3. **Deployments** manage ReplicaSets and provide rolling updates
4. **Services** provide stable networking for ephemeral pods
5. **Declarative YAML** is the production-standard approach
6. **Labels** connect everything — selectors match labels

---

## Next: Namespaces, Labels, ConfigMaps, and Secrets

In the next section, we'll organize our cluster with namespaces and manage configuration properly.
