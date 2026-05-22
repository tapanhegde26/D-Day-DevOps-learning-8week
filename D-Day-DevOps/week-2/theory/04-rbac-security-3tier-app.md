# Week 2, Day 7-8: RBAC, Security & 3-Tier Application

## Kubernetes RBAC Deep Dive

RBAC (Role-Based Access Control) controls who can do what in your cluster.

### RBAC Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           RBAC MODEL                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  WHO (Subjects)              WHAT (Rules)              WHERE (Scope)        │
│  ┌─────────────┐            ┌─────────────┐          ┌─────────────┐       │
│  │   Users     │            │  Verbs      │          │  Namespace  │       │
│  │   Groups    │◄──────────►│  Resources  │◄────────►│  (Role)     │       │
│  │   SAs       │            │  API Groups │          │             │       │
│  └─────────────┘            └─────────────┘          │  Cluster    │       │
│                                                       │  (ClusterRole)│     │
│                                                       └─────────────┘       │
│                                                                              │
│  Bindings connect Subjects to Roles:                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  RoleBinding         → Role (namespace-scoped)                       │   │
│  │  ClusterRoleBinding  → ClusterRole (cluster-scoped)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Role and ClusterRole

```yaml
# Namespace-scoped Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]           # "" = core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
# Cluster-scoped ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

### Common Verbs

| Verb | Description |
|------|-------------|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create resources |
| `update` | Update existing resources |
| `patch` | Partially update resources |
| `delete` | Delete resources |
| `deletecollection` | Delete multiple resources |

### RoleBinding and ClusterRoleBinding

```yaml
# Bind Role to User in namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: developer@company.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: ci-bot
    namespace: ci-cd
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
# Bind ClusterRole cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: admin@company.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Aggregated ClusterRoles

```yaml
# Base role with aggregation label
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
---
# Aggregated role that combines all matching roles
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # Rules are automatically filled in
```

---

## EKS Access Management

### aws-auth ConfigMap

```yaml
# aws-auth ConfigMap (managed by EKS module)
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/AdminRole
      username: admin
      groups:
        - system:masters
    - rolearn: arn:aws:iam::123456789012:role/DeveloperRole
      username: developer
      groups:
        - developers
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/admin
      username: admin
      groups:
        - system:masters
```

### EKS Access Entries (New Method)

```hcl
# Terraform - EKS Access Entries
resource "aws_eks_access_entry" "admin" {
  cluster_name  = module.eks.cluster_name
  principal_arn = "arn:aws:iam::123456789012:role/AdminRole"
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "admin" {
  cluster_name  = module.eks.cluster_name
  principal_arn = aws_eks_access_entry.admin.principal_arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"

  access_scope {
    type = "cluster"
  }
}

resource "aws_eks_access_entry" "developer" {
  cluster_name  = module.eks.cluster_name
  principal_arn = "arn:aws:iam::123456789012:role/DeveloperRole"
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "developer" {
  cluster_name  = module.eks.cluster_name
  principal_arn = aws_eks_access_entry.developer.principal_arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type       = "namespace"
    namespaces = ["development", "staging"]
  }
}
```

---

## Production RBAC Patterns

### Developer Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
  # Read-only access to most resources
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  # Can manage deployments in their namespace
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Can view logs and exec into pods
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "create"]
  # Can port-forward
  - apiGroups: [""]
    resources: ["pods/portforward"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### CI/CD ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: ci-cd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ci-deployer
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-production
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: ci-cd
roleRef:
  kind: ClusterRole
  name: ci-deployer
  apiGroup: rbac.authorization.k8s.io
```

---

## Database Migrations with Kubernetes Jobs

### Migration Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v1-2-0
  namespace: production
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400  # Clean up after 24 hours
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: migration-sa
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z $DATABASE_HOST $DATABASE_PORT; do
                echo "Waiting for database..."
                sleep 2
              done
              echo "Database is ready!"
          env:
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: host
            - name: DATABASE_PORT
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: port
      containers:
        - name: migration
          image: ghcr.io/company/backend:1.2.0
          command: ["npm", "run", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

### Migration CronJob for Scheduled Tasks

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-cleanup
  namespace: production
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleanup
              image: ghcr.io/company/backend:1.2.0
              command: ["npm", "run", "cleanup-old-data"]
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: url
              resources:
                requests:
                  cpu: 100m
                  memory: 256Mi
```

---

## RDS Module for Terraform

```hcl
# modules/rds/main.tf

resource "aws_db_subnet_group" "main" {
  name       = "${var.identifier}-subnet-group"
  subnet_ids = var.subnet_ids

  tags = {
    Name        = "${var.identifier}-subnet-group"
    Environment = var.environment
  }
}

resource "aws_security_group" "rds" {
  name        = "${var.identifier}-rds-sg"
  description = "Security group for RDS"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.eks_security_group_id]
    description     = "PostgreSQL from EKS"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.identifier}-rds-sg"
    Environment = var.environment
  }
}

resource "aws_db_instance" "main" {
  identifier = var.identifier

  engine               = "postgres"
  engine_version       = var.engine_version
  instance_class       = var.instance_class
  allocated_storage    = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type         = "gp3"
  storage_encrypted    = true

  db_name  = var.database_name
  username = var.master_username
  password = var.master_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az               = var.environment == "prod" ? true : false
  publicly_accessible    = false
  deletion_protection    = var.environment == "prod" ? true : false
  skip_final_snapshot    = var.environment == "prod" ? false : true
  final_snapshot_identifier = var.environment == "prod" ? "${var.identifier}-final-snapshot" : null

  backup_retention_period = var.environment == "prod" ? 7 : 1
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  performance_insights_enabled = true
  monitoring_interval         = 60
  monitoring_role_arn        = aws_iam_role.rds_monitoring.arn

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  tags = {
    Name        = var.identifier
    Environment = var.environment
  }
}

# Store credentials in Secrets Manager
resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.environment}/database/credentials"
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = var.master_username
    password = var.master_password
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    database = var.database_name
    url      = "postgresql://${var.master_username}:${var.master_password}@${aws_db_instance.main.address}:${aws_db_instance.main.port}/${var.database_name}"
  })
}
```

---

## Week 2 Capstone: 3-Tier E-Commerce on EKS

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS EKS                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Internet ──► ALB ──► Ingress                                               │
│                          │                                                   │
│              ┌───────────┴───────────┐                                      │
│              │                       │                                      │
│              ▼                       ▼                                      │
│       ┌─────────────┐         ┌─────────────┐                              │
│       │  Frontend   │         │   Backend   │                              │
│       │  (React)    │────────►│  (Node.js)  │                              │
│       │  3 replicas │         │  5 replicas │                              │
│       └─────────────┘         └──────┬──────┘                              │
│                                      │                                      │
│                                      │ IRSA                                 │
│                                      ▼                                      │
│                               ┌─────────────┐                              │
│                               │    RDS      │                              │
│                               │ PostgreSQL  │                              │
│                               └─────────────┘                              │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Supporting Services:                                                │   │
│  │  - AWS Load Balancer Controller                                      │   │
│  │  - ExternalDNS                                                       │   │
│  │  - External Secrets Operator                                         │   │
│  │  - Prometheus + Grafana                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Deployment Steps

```bash
# 1. Deploy infrastructure with Terraform
cd terraform/environments/dev
terraform init
terraform apply

# 2. Configure kubectl
aws eks update-kubeconfig --name ecommerce-dev --region us-west-2

# 3. Install supporting services
helm upgrade --install aws-lb-controller aws-load-balancer-controller \
  -n kube-system -f values/aws-lb-controller.yaml

helm upgrade --install external-dns external-dns \
  -n kube-system -f values/external-dns.yaml

helm upgrade --install external-secrets external-secrets \
  -n external-secrets --create-namespace

# 4. Run database migration
kubectl apply -f k8s/jobs/migration.yaml
kubectl wait --for=condition=complete job/db-migration -n production

# 5. Deploy application with Helm
helm upgrade --install ecommerce ./charts/ecommerce-platform \
  -n production --create-namespace \
  -f values/production.yaml

# 6. Verify deployment
kubectl get pods -n production
kubectl get ingress -n production
kubectl get svc -n production
```

### Production values.yaml

```yaml
# values/production.yaml
global:
  environment: production
  domain: example.com

frontend:
  replicaCount: 3
  image:
    repository: ghcr.io/company/frontend
    tag: "1.0.0"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
  ingress:
    enabled: true
    annotations:
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    hosts:
      - host: www.example.com
        paths:
          - path: /
            pathType: Prefix

backend:
  replicaCount: 5
  image:
    repository: ghcr.io/company/backend
    tag: "1.0.0"
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 512Mi
  autoscaling:
    enabled: true
    minReplicas: 5
    maxReplicas: 20
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/backend-role
  externalSecrets:
    enabled: true
    secretName: production/database
  ingress:
    enabled: true
    hosts:
      - host: api.example.com
        paths:
          - path: /
            pathType: Prefix

migrations:
  enabled: true
```

---

## Week 2 Summary

### What You've Learned

1. **Terraform for EKS** — Modular infrastructure as code
2. **VPC Design** — Production networking for Kubernetes
3. **EKS Configuration** — Node groups, add-ons, OIDC
4. **Helm Charts** — Packaging and templating applications
5. **IRSA** — Secure AWS access for pods
6. **AWS Load Balancer Controller** — ALB/NLB management
7. **ExternalDNS** — Automatic DNS management
8. **RBAC** — Access control and security
9. **Database Migrations** — Jobs and CronJobs
10. **3-Tier Architecture** — Production deployment patterns

### Key Commands

```bash
# Terraform
terraform init
terraform plan
terraform apply
terraform workspace list/select/new

# Helm
helm create my-chart
helm template my-release ./my-chart
helm install/upgrade my-release ./my-chart
helm list -A
helm rollback my-release 1

# EKS
aws eks update-kubeconfig --name cluster --region region
eksctl get cluster

# RBAC
kubectl auth can-i create pods --as developer
kubectl get roles,rolebindings -A
```

### Interview Questions

1. Explain IRSA and how it provides secure AWS access.
2. What's the difference between Role and ClusterRole?
3. How do you handle database migrations in Kubernetes?
4. Explain the AWS Load Balancer Controller annotations.
5. How do you manage multiple environments with Terraform?
6. What are Helm hooks and when would you use them?

---

## Next Week: Microservices, Service Mesh, mTLS

In Week 3, we'll dive into:
- Istio service mesh
- mTLS for zero-trust security
- Canary deployments
- Gateway API
- Network Policies
