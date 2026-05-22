# Week 5: Scaling, StatefulSets & Storage

## Overview

This week covers advanced scaling with Karpenter and KEDA, stateful workloads, and persistent storage patterns.

---

## Day 1-2: Karpenter for Node Autoscaling

### Karpenter vs Cluster Autoscaler

| Feature | Karpenter | Cluster Autoscaler |
|---------|-----------|-------------------|
| Speed | Seconds | Minutes |
| Flexibility | Any instance type | Predefined node groups |
| Cost optimization | Built-in Spot support | Limited |
| Consolidation | Automatic | Manual |
| Cloud support | AWS (GCP/Azure coming) | All major clouds |

### Karpenter Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        KARPENTER WORKFLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Pending Pod detected                                                    │
│     ┌─────────────┐                                                         │
│     │ Unscheduled │                                                         │
│     │    Pod      │                                                         │
│     └──────┬──────┘                                                         │
│            │                                                                 │
│            ▼                                                                 │
│  2. Karpenter evaluates NodePool constraints                                │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │  NodePool: general                                               │    │
│     │  - Instance types: c5.*, m5.*, r5.*                             │    │
│     │  - Capacity: spot, on-demand                                     │    │
│     │  - Zones: us-west-2a, us-west-2b                                │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│            │                                                                 │
│            ▼                                                                 │
│  3. Launches optimal EC2 instance                                           │
│     ┌─────────────┐                                                         │
│     │  New Node   │ ◄── Right-sized for pending pods                       │
│     │  (c5.xlarge)│                                                         │
│     └──────┬──────┘                                                         │
│            │                                                                 │
│            ▼                                                                 │
│  4. Pod scheduled on new node                                               │
│     ┌─────────────┐                                                         │
│     │  Running    │                                                         │
│     │    Pod      │                                                         │
│     └─────────────┘                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Installing Karpenter

```bash
# Set environment variables
export KARPENTER_VERSION=v0.32.0
export CLUSTER_NAME=my-cluster
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Install Karpenter
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --namespace karpenter --create-namespace \
  --set settings.clusterName=${CLUSTER_NAME} \
  --set settings.interruptionQueue=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi
```

### NodePool Configuration

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["medium", "large", "xlarge", "2xlarge"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  role: KarpenterNodeRole-${CLUSTER_NAME}
  tags:
    Environment: production
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
```

### Spot Instance Handling

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule
```

```yaml
# Deployment tolerating spot nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  template:
    spec:
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: karpenter.sh/capacity-type
                    operator: In
                    values: ["spot"]
```

---

## Day 3-4: KEDA for Event-Driven Autoscaling

### KEDA Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          KEDA WORKFLOW                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  External Metrics Sources                                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                          │
│  │   SQS   │ │  Kafka  │ │Prometheus│ │  Cron   │                          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘                          │
│       │           │           │           │                                 │
│       └───────────┴─────┬─────┴───────────┘                                │
│                         │                                                   │
│                         ▼                                                   │
│                  ┌─────────────┐                                           │
│                  │    KEDA     │                                           │
│                  │  Operator   │                                           │
│                  └──────┬──────┘                                           │
│                         │                                                   │
│                         ▼                                                   │
│                  ┌─────────────┐                                           │
│                  │     HPA     │                                           │
│                  └──────┬──────┘                                           │
│                         │                                                   │
│                         ▼                                                   │
│                  ┌─────────────┐                                           │
│                  │ Deployment  │ Scale 0 ↔ N                               │
│                  └─────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Installing KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

### ScaledObject Examples

```yaml
# SQS Queue Scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-west-2.amazonaws.com/123456789/orders
        queueLength: "5"
        awsRegion: us-west-2
      authenticationRef:
        name: keda-aws-credentials
---
# Prometheus Scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-server
spec:
  scaleTargetRef:
    name: api-server
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_per_second
        query: sum(rate(http_requests_total{service="api"}[2m]))
        threshold: "100"
---
# Cron Scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: batch-job
spec:
  scaleTargetRef:
    name: batch-processor
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: cron
      metadata:
        timezone: America/Los_Angeles
        start: 0 6 * * *
        end: 0 20 * * *
        desiredReplicas: "5"
---
# Kafka Scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 1
  maxReplicaCount: 100
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-group
        topic: events
        lagThreshold: "100"
```

### KEDA Authentication

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-aws-credentials
  namespace: production
spec:
  podIdentity:
    provider: aws-eks
---
# Or using secrets
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-kafka-auth
spec:
  secretTargetRef:
    - parameter: sasl
      name: kafka-credentials
      key: sasl
    - parameter: username
      name: kafka-credentials
      key: username
    - parameter: password
      name: kafka-credentials
      key: password
```

---

## Day 5-6: StatefulSets & Persistent Storage

### StatefulSet Fundamentals

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2
              memory: 4Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi
---
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

### StatefulSet DNS

```
# Pod DNS names
postgres-0.postgres-headless.production.svc.cluster.local
postgres-1.postgres-headless.production.svc.cluster.local
postgres-2.postgres-headless.production.svc.cluster.local
```

### Storage Classes

```yaml
# GP3 StorageClass for EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# EFS StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxx
  directoryPerms: "700"
```

### EBS vs EFS

| Feature | EBS | EFS |
|---------|-----|-----|
| Access Mode | ReadWriteOnce | ReadWriteMany |
| Performance | High IOPS | Scalable throughput |
| Use Case | Databases | Shared storage |
| Cost | Lower | Higher |
| Multi-AZ | No (snapshots) | Yes |

---

## Day 7-8: Backup & Disaster Recovery

### Velero for Backup

```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero-backups \
  --backup-location-config region=us-west-2 \
  --snapshot-location-config region=us-west-2 \
  --secret-file ./credentials-velero
```

```yaml
# Scheduled backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - production
    includedResources:
      - persistentvolumeclaims
      - persistentvolumes
      - secrets
      - configmaps
      - deployments
      - statefulsets
      - services
    snapshotVolumes: true
    ttl: 720h  # 30 days
---
# On-demand backup
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: pre-upgrade-backup
  namespace: velero
spec:
  includedNamespaces:
    - production
  snapshotVolumes: true
```

### Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: data-postgres-0
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
spec:
  storageClassName: gp3
  dataSource:
    name: postgres-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

---

## Week 5 Capstone

Enhanced microservices with:
- Karpenter for Spot node scaling
- KEDA for SQS-based pod autoscaling
- PostgreSQL StatefulSet with replication
- Kafka StatefulSet for event streaming
- Velero backup and disaster recovery

---

## Key Takeaways

1. **Karpenter** provides fast, flexible node provisioning
2. **KEDA** enables scale-to-zero and event-driven scaling
3. **StatefulSets** provide stable identity for stateful apps
4. **Storage Classes** abstract storage provisioning
5. **Velero** provides backup and disaster recovery
6. **Volume Snapshots** enable point-in-time recovery

---

## Next Week: DevSecOps on Kubernetes

Week 6 covers security scanning, policy enforcement, and compliance.
