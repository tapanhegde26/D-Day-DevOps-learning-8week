# Week 1, Day 3-4: Namespaces, Labels, ConfigMaps, Secrets & Probes

## Namespaces — Virtual Clusters

Namespaces provide **logical isolation** within a cluster. Think of them as folders for your Kubernetes resources.

### Default Namespaces

```bash
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   1d    # Where resources go if no namespace specified
# kube-system       Active   1d    # Kubernetes system components
# kube-public       Active   1d    # Publicly readable resources
# kube-node-lease   Active   1d    # Node heartbeat leases
```

### When to Use Namespaces

| Use Case | Example |
|----------|---------|
| Environment separation | `dev`, `staging`, `prod` |
| Team isolation | `team-a`, `team-b` |
| Application grouping | `frontend`, `backend`, `database` |
| Resource quotas | Limit CPU/memory per namespace |

### Creating and Using Namespaces

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
```

```bash
# Create namespace
kubectl create namespace development
# Or
kubectl apply -f namespace.yaml

# List namespaces
kubectl get ns

# Deploy to specific namespace
kubectl apply -f deployment.yaml -n development

# Set default namespace for current context
kubectl config set-context --current --namespace=development

# View resources in all namespaces
kubectl get pods -A
kubectl get pods --all-namespaces
```

### Namespace-Scoped vs Cluster-Scoped Resources

**Namespace-scoped** (isolated per namespace):
- Pods, Deployments, Services, ConfigMaps, Secrets, ServiceAccounts

**Cluster-scoped** (global):
- Nodes, PersistentVolumes, Namespaces, ClusterRoles, StorageClasses

```bash
# Check if a resource is namespaced
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

---

## Labels and Selectors — The Glue of Kubernetes

Labels are **key-value pairs** attached to objects. Selectors query objects by labels.

### Adding Labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: web
    environment: production
    version: v1.2.3
    team: platform
spec:
  containers:
    - name: nginx
      image: nginx
```

```bash
# Add label to existing resource
kubectl label pod my-pod tier=frontend

# Update existing label
kubectl label pod my-pod version=v1.2.4 --overwrite

# Remove label
kubectl label pod my-pod tier-
```

### Querying with Selectors

```bash
# Equality-based selectors
kubectl get pods -l app=web
kubectl get pods -l environment=production
kubectl get pods -l 'environment!=production'

# Set-based selectors
kubectl get pods -l 'environment in (production, staging)'
kubectl get pods -l 'environment notin (development)'
kubectl get pods -l 'app,!version'  # Has app label, doesn't have version

# Multiple conditions (AND)
kubectl get pods -l app=web,environment=production
```

### Selectors in YAML

```yaml
# Deployment selector (must match pod template labels)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: web
    matchExpressions:
      - key: environment
        operator: In
        values:
          - production
          - staging
  template:
    metadata:
      labels:
        app: web
        environment: production
```

### Annotations — Non-Identifying Metadata

Annotations store **non-identifying** information (not used for selection).

```yaml
metadata:
  annotations:
    description: "Main web application"
    owner: "platform-team@company.com"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

**Common uses:**
- Build/release information
- Tool configuration (Prometheus, Istio)
- Rollout tracking
- Documentation

---

## ConfigMaps — Externalize Configuration

ConfigMaps store **non-sensitive** configuration data.

### Creating ConfigMaps

**From literal values:**
```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=postgres \
  --from-literal=DATABASE_PORT=5432 \
  --from-literal=LOG_LEVEL=info
```

**From file:**
```bash
# config.properties
DATABASE_HOST=postgres
DATABASE_PORT=5432
LOG_LEVEL=info

kubectl create configmap app-config --from-file=config.properties
```

**From YAML:**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
  LOG_LEVEL: info
  
  # Multi-line configuration file
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
```

### Using ConfigMaps in Pods

**As environment variables:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:1.0
      # Individual keys
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_PORT
      # All keys as env vars
      envFrom:
        - configMapRef:
            name: app-config
```

**As mounted files:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        # Optional: select specific keys
        items:
          - key: nginx.conf
            path: nginx.conf
```

**Result:** Files created at `/etc/config/DATABASE_HOST`, `/etc/config/nginx.conf`, etc.

### ConfigMap Updates

- **Environment variables**: Pod must be restarted to see changes
- **Mounted volumes**: Updated automatically (may take up to 1 minute)

---

## Secrets — Sensitive Data Management

Secrets store **sensitive** data like passwords, tokens, and keys.

### Important: Secrets Are Not Encrypted by Default!

Secrets are only **base64 encoded**, not encrypted. Anyone with API access can decode them.

**For production:**
- Enable encryption at rest in etcd
- Use external secret managers (Vault, AWS Secrets Manager)
- Use RBAC to restrict access

### Creating Secrets

**From literal values:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cr3tP@ss!'
```

**From files:**
```bash
# Create files
echo -n 'admin' > ./username.txt
echo -n 'S3cr3tP@ss!' > ./password.txt

kubectl create secret generic db-credentials \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt
```

**From YAML (values must be base64 encoded):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # echo -n 'admin' | base64
  username: YWRtaW4=
  # echo -n 'S3cr3tP@ss!' | base64
  password: UzNjcjN0UEBzcyE=
```

**Using stringData (auto-encodes):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: S3cr3tP@ss!
```

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/service-account-token` | Service account token |
| `kubernetes.io/basic-auth` | Basic authentication |

### Using Secrets in Pods

**As environment variables:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:1.0
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
      # All keys
      envFrom:
        - secretRef:
            name: db-credentials
```

**As mounted files:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
        defaultMode: 0400  # Read-only for owner
```

### ImagePullSecrets — Pulling Private Images

```bash
# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
    - name: app
      image: myregistry.com/private-app:1.0
  imagePullSecrets:
    - name: regcred
```

**Attach to ServiceAccount (applies to all pods using it):**
```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

---

## Resource Management

### Requests and Limits

**Requests**: Minimum guaranteed resources (used for scheduling)
**Limits**: Maximum allowed resources (enforced at runtime)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: my-app:1.0
      resources:
        requests:
          memory: "256Mi"   # 256 Mebibytes
          cpu: "250m"       # 250 millicores (0.25 CPU)
        limits:
          memory: "512Mi"
          cpu: "500m"
```

**CPU units:**
- `1` = 1 vCPU/Core
- `500m` = 0.5 CPU (500 millicores)
- `100m` = 0.1 CPU

**Memory units:**
- `Ki`, `Mi`, `Gi` (Kibibytes, Mebibytes, Gibibytes - powers of 2)
- `K`, `M`, `G` (Kilobytes, Megabytes, Gigabytes - powers of 10)

### What Happens When Limits Are Exceeded?

| Resource | Behavior |
|----------|----------|
| CPU | Throttled (slowed down) |
| Memory | OOMKilled (container killed) |

### ResourceQuotas — Namespace Limits

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
```

### LimitRanges — Default and Constraints

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:          # Default limits if not specified
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:   # Default requests if not specified
        cpu: "100m"
        memory: "128Mi"
      min:              # Minimum allowed
        cpu: "50m"
        memory: "64Mi"
      max:              # Maximum allowed
        cpu: "2"
        memory: "2Gi"
```

---

## Health Probes — Keeping Applications Healthy

Kubernetes uses probes to know if your application is healthy.

### Three Types of Probes

| Probe | Purpose | When It Runs |
|-------|---------|--------------|
| **Startup** | Is the app started? | Until success, then stops |
| **Liveness** | Is the app alive? | Continuously after startup |
| **Readiness** | Can the app serve traffic? | Continuously |

### Probe Actions

**HTTP GET:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: Custom-Header
        value: Awesome
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
  successThreshold: 1
```

**TCP Socket:**
```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10
```

**Exec Command:**
```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

**gRPC (Kubernetes 1.24+):**
```yaml
livenessProbe:
  grpc:
    port: 50051
    service: my-service  # Optional
  initialDelaySeconds: 10
```

### Complete Example with All Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-probes
spec:
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      
      # Startup probe - for slow-starting apps
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30    # 30 * 10s = 5 minutes to start
        periodSeconds: 10
      
      # Liveness probe - restart if unhealthy
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 0   # Starts after startup probe succeeds
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3      # Restart after 3 failures
      
      # Readiness probe - remove from service if not ready
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
        successThreshold: 1
```

### Probe Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `initialDelaySeconds` | Seconds before first probe | 0 |
| `periodSeconds` | How often to probe | 10 |
| `timeoutSeconds` | Probe timeout | 1 |
| `failureThreshold` | Failures before action | 3 |
| `successThreshold` | Successes to be healthy | 1 |

### Best Practices

1. **Always use readiness probes** for services
2. **Use startup probes** for slow-starting apps (instead of long `initialDelaySeconds`)
3. **Liveness probes should be simple** — don't check dependencies
4. **Readiness probes can check dependencies** — database, cache, etc.
5. **Don't make probes too aggressive** — avoid unnecessary restarts

---

## Init Containers — Pre-Start Tasks

Init containers run **before** app containers and must complete successfully.

**Use cases:**
- Wait for dependencies (database, service)
- Clone git repos
- Generate configuration
- Run database migrations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    # Wait for database to be ready
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting for db; sleep 2; done']
    
    # Clone configuration
    - name: clone-config
      image: alpine/git
      command: ['git', 'clone', 'https://github.com/org/config.git', '/config']
      volumeMounts:
        - name: config-volume
          mountPath: /config
  
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
  
  volumes:
    - name: config-volume
      emptyDir: {}
```

### Init Container Behavior

- Run sequentially (one after another)
- Each must complete successfully before the next starts
- If any fails, the pod restarts (respecting `restartPolicy`)
- App containers don't start until all init containers succeed

---

## Sidecar Pattern

Sidecars are **helper containers** that run alongside the main application.

**Common sidecars:**
- Log collectors (Fluent Bit)
- Proxies (Envoy, Istio)
- Monitoring agents
- Secret refreshers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
    # Main application
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    
    # Sidecar: Log collector
    - name: log-collector
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluent-config
          mountPath: /fluent-bit/etc
  
  volumes:
    - name: logs
      emptyDir: {}
    - name: fluent-config
      configMap:
        name: fluent-bit-config
```

---

## Hands-On Labs

### Lab 1: Namespaces and Resource Quotas

```bash
# Create namespace
kubectl create namespace lab-dev

# Create resource quota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: lab-dev
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "5"
EOF

# Try to create pods exceeding quota
kubectl run test1 --image=nginx -n lab-dev
kubectl run test2 --image=nginx -n lab-dev
# ... continue until quota is hit

# Check quota usage
kubectl describe resourcequota dev-quota -n lab-dev
```

### Lab 2: ConfigMaps and Secrets

```bash
# Create ConfigMap
kubectl create configmap app-settings \
  --from-literal=APP_ENV=development \
  --from-literal=LOG_LEVEL=debug

# Create Secret
kubectl create secret generic db-creds \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=secret123

# Deploy pod using both
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ['sh', '-c', 'env && sleep 3600']
      envFrom:
        - configMapRef:
            name: app-settings
        - secretRef:
            name: db-creds
EOF

# Verify environment variables
kubectl exec config-demo -- env | grep -E 'APP_|DB_|LOG_'
```

### Lab 3: Health Probes

```bash
# Deploy app with probes
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
    - name: app
      image: nginx
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
EOF

# Watch pod status
kubectl get pod probe-demo -w

# Simulate failure (delete index.html)
kubectl exec probe-demo -- rm /usr/share/nginx/html/index.html

# Watch pod become not ready, then restart
kubectl get pod probe-demo -w
```

---

## Key Takeaways

1. **Namespaces** provide logical isolation and resource boundaries
2. **Labels** are the primary way to organize and select resources
3. **ConfigMaps** for non-sensitive config, **Secrets** for sensitive data
4. **Resource requests** guarantee resources, **limits** cap usage
5. **Probes** keep your application healthy and traffic flowing correctly
6. **Init containers** handle pre-start tasks
7. **Sidecars** extend functionality without modifying the main app

---

## Next: HPA, Deployment Strategies, and Debugging

In the next section, we'll cover autoscaling, advanced deployment strategies, and troubleshooting techniques.
