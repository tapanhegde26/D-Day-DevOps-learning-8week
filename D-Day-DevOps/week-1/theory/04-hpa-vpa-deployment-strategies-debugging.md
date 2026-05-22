# Week 1, Day 5-6: HPA, VPA, Deployment Strategies & Debugging

## Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of pods based on observed metrics.

### How HPA Works

```
┌─────────────────────────────────────────────────────────────────┐
│                        HPA Control Loop                          │
│                                                                  │
│  1. Metrics Server collects CPU/Memory from kubelets            │
│  2. HPA queries Metrics Server every 15 seconds                 │
│  3. HPA calculates desired replicas:                            │
│     desiredReplicas = ceil(currentReplicas * (current/target))  │
│  4. HPA updates Deployment's replica count                      │
│  5. Deployment creates/removes pods                             │
└─────────────────────────────────────────────────────────────────┘
```

### Installing Metrics Server (Required for HPA)

```bash
# For Kind clusters
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for Kind (disable TLS verification)
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Verify metrics server is running
kubectl get pods -n kube-system | grep metrics-server

# Test metrics
kubectl top nodes
kubectl top pods
```

### Creating HPA

**Imperative:**
```bash
# Scale based on CPU (target 50%)
kubectl autoscale deployment nginx --cpu-percent=50 --min=2 --max=10
```

**Declarative:**
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
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

### HPA with Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Scale based on requests per second (from Prometheus)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1000
    # Scale based on queue length (external metric)
    - type: External
      external:
        metric:
          name: sqs_queue_length
          selector:
            matchLabels:
              queue: orders
        target:
          type: AverageValue
          averageValue: 30
```

### Testing HPA

```bash
# Deploy a test application
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
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
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
    - port: 80
EOF

# Create HPA
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# Generate load (in another terminal)
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

# Watch HPA scale up
kubectl get hpa php-apache -w

# Stop load generator (Ctrl+C) and watch scale down
```

---

## Vertical Pod Autoscaler (VPA)

VPA automatically adjusts CPU and memory **requests** for pods.

### When to Use VPA vs HPA

| Scenario | Use |
|----------|-----|
| Stateless apps, variable load | HPA |
| Right-sizing resource requests | VPA |
| Stateful apps (databases) | VPA |
| Unknown resource requirements | VPA (recommendation mode) |

**Important**: Don't use HPA and VPA together on CPU/memory for the same deployment.

### Installing VPA

```bash
# Clone VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA
./hack/vpa-up.sh

# Verify
kubectl get pods -n kube-system | grep vpa
```

### VPA Modes

| Mode | Behavior |
|------|----------|
| `Off` | Only provides recommendations |
| `Initial` | Sets requests only at pod creation |
| `Auto` | Updates requests (may restart pods) |

### VPA Example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, or Auto
  resourcePolicy:
    containerPolicies:
      - containerName: nginx
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
```

```bash
# Check VPA recommendations
kubectl describe vpa nginx-vpa
```

---

## Pod Disruption Budgets (PDB)

PDBs ensure a minimum number of pods remain available during voluntary disruptions (node drains, upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  # Option 1: Minimum available
  minAvailable: 2
  # OR Option 2: Maximum unavailable
  # maxUnavailable: 1
  # OR use percentages
  # minAvailable: "50%"
  selector:
    matchLabels:
      app: nginx
```

**When PDBs apply:**
- `kubectl drain` (node maintenance)
- Cluster autoscaler removing nodes
- Voluntary pod evictions

**When PDBs don't apply:**
- Node failures (involuntary)
- Deleting pods directly
- Application crashes

```bash
# Check PDB status
kubectl get pdb

# Try to drain a node (will respect PDB)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

---

## Deployment Strategies

### 1. Rolling Update (Default)

Gradually replaces old pods with new ones.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods over desired (can be % or number)
      maxUnavailable: 0  # Max pods unavailable (can be % or number)
```

**Timeline with maxSurge=1, maxUnavailable=0:**
```
Start:    [v1] [v1] [v1] [v1]           (4 pods)
Step 1:   [v1] [v1] [v1] [v1] [v2]      (5 pods, 1 new)
Step 2:   [v1] [v1] [v1] [v2] [v2]      (v1 terminated, v2 ready)
Step 3:   [v1] [v1] [v2] [v2] [v2]
Step 4:   [v1] [v2] [v2] [v2] [v2]
End:      [v2] [v2] [v2] [v2]           (4 pods, all v2)
```

### 2. Recreate

Terminates all old pods before creating new ones. **Causes downtime.**

```yaml
spec:
  strategy:
    type: Recreate
```

**Use when:**
- Application can't run multiple versions simultaneously
- Database schema changes require single version
- Development environments where downtime is acceptable

### 3. Blue-Green Deployment

Run two identical environments, switch traffic instantly.

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:1.0
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: app
          image: myapp:2.0
---
# Service pointing to blue
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Switch to 'green' to cutover
  ports:
    - port: 80
```

**Cutover:**
```bash
# Switch traffic to green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to blue
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

### 4. Canary Deployment

Route a small percentage of traffic to the new version.

**Simple canary with replica ratio:**
```yaml
# Stable deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
        - name: app
          image: myapp:1.0
---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
        - name: app
          image: myapp:2.0
---
# Service routes to both (based on replica count)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Matches both deployments
  ports:
    - port: 80
```

**For precise traffic control, use Istio or a service mesh (covered in Week 3).**

---

## Rollbacks

### Viewing Rollout History

```bash
# View history
kubectl rollout history deployment/nginx

# View specific revision
kubectl rollout history deployment/nginx --revision=2
```

### Performing Rollbacks

```bash
# Rollback to previous version
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Check rollout status
kubectl rollout status deployment/nginx
```

### Pausing and Resuming Rollouts

```bash
# Pause rollout (for canary testing)
kubectl rollout pause deployment/nginx

# Make changes while paused
kubectl set image deployment/nginx nginx=nginx:1.26
kubectl set resources deployment/nginx -c nginx --limits=cpu=200m,memory=512Mi

# Resume rollout (applies all changes at once)
kubectl rollout resume deployment/nginx
```

---

## Debugging Kubernetes Applications

### Common Pod States and Issues

| State | Meaning | Common Causes |
|-------|---------|---------------|
| `Pending` | Not scheduled | No resources, node selector, taints |
| `ContainerCreating` | Setting up | Pulling image, mounting volumes |
| `Running` | Containers running | Normal state |
| `CrashLoopBackOff` | Repeatedly crashing | App error, missing config, bad command |
| `ImagePullBackOff` | Can't pull image | Wrong image name, no credentials |
| `OOMKilled` | Out of memory | Memory limit too low |
| `Evicted` | Node pressure | Node out of resources |

### Debugging CrashLoopBackOff

```bash
# 1. Check pod status and events
kubectl describe pod <pod-name>

# 2. Check logs (current container)
kubectl logs <pod-name>

# 3. Check logs (previous crashed container)
kubectl logs <pod-name> --previous

# 4. Check if it's a specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# 5. Common fixes:
#    - Check command/args in pod spec
#    - Verify environment variables
#    - Check ConfigMaps/Secrets exist
#    - Verify image exists and is correct
```

### Debugging OOMKilled

```bash
# 1. Check pod events
kubectl describe pod <pod-name> | grep -A 10 "Last State"

# 2. Check resource usage
kubectl top pod <pod-name>

# 3. Check limits
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'

# 4. Solutions:
#    - Increase memory limits
#    - Fix memory leak in application
#    - Use VPA to right-size
```

### Debugging ImagePullBackOff

```bash
# 1. Check events for error message
kubectl describe pod <pod-name> | grep -A 5 "Events"

# 2. Common causes:
#    - Image doesn't exist (typo in name/tag)
#    - Private registry without imagePullSecrets
#    - Network issues reaching registry

# 3. Verify image exists
docker pull <image-name>

# 4. Check imagePullSecrets
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'
```

### Debugging Pending Pods

```bash
# 1. Check events
kubectl describe pod <pod-name> | grep -A 10 "Events"

# 2. Common causes and solutions:

# Insufficient resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Node selector doesn't match
kubectl get nodes --show-labels
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeSelector}'

# Taints preventing scheduling
kubectl describe nodes | grep Taints

# PVC not bound
kubectl get pvc
```

### Debugging Services

```bash
# 1. Check service exists and has endpoints
kubectl get svc <service-name>
kubectl get endpoints <service-name>

# 2. If no endpoints, check selector matches pod labels
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels

# 3. Test connectivity from within cluster
kubectl run curl --image=curlimages/curl -it --rm -- curl http://<service-name>

# 4. Check if pods are ready (readiness probe)
kubectl get pods -l <selector>
```

### Debugging Network Issues

```bash
# 1. Test DNS resolution
kubectl run dnstest --image=busybox:1.36 -it --rm -- nslookup <service-name>

# 2. Test connectivity between pods
kubectl exec <pod-a> -- curl http://<pod-b-ip>:port

# 3. Check network policies
kubectl get networkpolicies -A

# 4. Check kube-proxy is running
kubectl get pods -n kube-system | grep kube-proxy
```

### Reading Kubernetes Events

```bash
# All events in namespace
kubectl get events --sort-by='.lastTimestamp'

# Events for specific resource
kubectl get events --field-selector involvedObject.name=<pod-name>

# Watch events in real-time
kubectl get events -w

# Events across all namespaces
kubectl get events -A --sort-by='.lastTimestamp'
```

---

## Kubernetes DNS and Service Discovery

### DNS Structure

```
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**
```bash
# Same namespace
curl http://nginx

# Different namespace
curl http://nginx.production.svc.cluster.local

# Headless service (returns pod IPs)
curl http://nginx-headless
# Returns: pod-0.nginx-headless.default.svc.cluster.local
```

### DNS Debugging

```bash
# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 -it --rm -- nslookup kubernetes

# Check DNS config in pod
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

## Useful Tools for Debugging

### k9s — Terminal UI for Kubernetes

```bash
# Install
brew install k9s  # macOS
# or download from https://k9scli.io/

# Run
k9s

# Key shortcuts:
# :pods - view pods
# :svc - view services
# :deploy - view deployments
# l - view logs
# d - describe
# s - shell into pod
# ctrl+d - delete
```

### Lens — Desktop Kubernetes IDE

Download from https://k8slens.dev/

Features:
- Visual cluster management
- Built-in terminal
- Log viewing
- Resource editing
- Prometheus integration

### stern — Multi-pod Log Tailing

```bash
# Install
brew install stern

# Tail logs from all pods matching pattern
stern nginx

# Tail from specific namespace
stern nginx -n production

# With timestamps
stern nginx -t

# Specific container
stern nginx -c sidecar
```

---

## Hands-On Labs

### Lab 1: HPA Testing

```bash
# Create deployment with resource requests
kubectl apply -f - <<EOF
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
        - name: app
          image: registry.k8s.io/hpa-example
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
EOF

# Create HPA
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5

# Generate load
kubectl run -i --tty load-generator --rm --image=busybox:1.36 -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo; done"

# In another terminal, watch scaling
kubectl get hpa hpa-demo -w
kubectl get pods -l app=hpa-demo -w
```

### Lab 2: Deployment Strategies

```bash
# Create initial deployment
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strategy-demo
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: strategy-demo
  template:
    metadata:
      labels:
        app: strategy-demo
    spec:
      containers:
        - name: app
          image: nginx:1.24
EOF

# Watch pods
kubectl get pods -l app=strategy-demo -w

# In another terminal, trigger rolling update
kubectl set image deployment/strategy-demo app=nginx:1.25

# Watch the rolling update happen
# Notice: never goes below 4 pods (maxUnavailable=0)
# Notice: goes up to 5 pods temporarily (maxSurge=1)
```

### Lab 3: Debugging Practice

```bash
# Create a broken pod (wrong image)
kubectl run broken-image --image=nginx:nonexistent

# Debug it
kubectl describe pod broken-image
kubectl get events --field-selector involvedObject.name=broken-image

# Create a crashing pod
kubectl run crasher --image=busybox -- /bin/sh -c "exit 1"

# Debug it
kubectl describe pod crasher
kubectl logs crasher --previous

# Create OOMKilled pod
kubectl run oom-demo --image=polinux/stress --limits=memory=50Mi -- stress --vm 1 --vm-bytes 100M

# Debug it
kubectl describe pod oom-demo | grep -A 5 "Last State"
```

---

## Key Takeaways

1. **HPA** scales pods horizontally based on metrics (CPU, memory, custom)
2. **VPA** adjusts resource requests vertically
3. **PDBs** protect availability during voluntary disruptions
4. **Rolling updates** are the default, safest deployment strategy
5. **Blue-green** and **canary** provide more control over releases
6. **Debugging** starts with `describe`, `logs`, and `events`
7. **Tools like k9s and stern** make debugging much easier

---

## Next: Prometheus, Grafana, ArgoCD, and CI/CD

In the next section, we'll set up monitoring and GitOps for our cluster.
