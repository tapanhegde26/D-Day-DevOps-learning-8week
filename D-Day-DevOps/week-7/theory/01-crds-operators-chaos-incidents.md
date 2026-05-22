# Week 7: CRDs, Operators, Chaos Engineering & Production Incidents

## Overview

This week covers extending Kubernetes with CRDs and Operators, chaos engineering practices, and handling production incidents.

---

## Day 1-2: Custom Resource Definitions (CRDs)

### Understanding CRDs

CRDs extend the Kubernetes API with custom resources.

```yaml
# Define a custom resource
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com
spec:
  group: mycompany.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - engine
                - version
                - storage
              properties:
                engine:
                  type: string
                  enum: ["postgres", "mysql", "mongodb"]
                version:
                  type: string
                storage:
                  type: string
                  pattern: "^[0-9]+Gi$"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 5
                  default: 1
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    schedule:
                      type: string
                      default: "0 2 * * *"
            status:
              type: object
              properties:
                phase:
                  type: string
                endpoint:
                  type: string
                replicas:
                  type: integer
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Version
          type: string
          jsonPath: .spec.version
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
```

### Using Custom Resources

```yaml
# Create a database instance
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: orders-db
  namespace: production
spec:
  engine: postgres
  version: "15"
  storage: 100Gi
  replicas: 3
  backup:
    enabled: true
    schedule: "0 */6 * * *"
```

```bash
# Interact with custom resources
kubectl get databases
kubectl get db orders-db -o yaml
kubectl describe db orders-db
```

### Real-World CRD Examples

Karpenter uses CRDs:
```yaml
# NodePool CRD
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
```

---

## Day 3-4: Kubernetes Operators

### Operator Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OPERATOR PATTERN                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    RECONCILIATION LOOP                               │   │
│  │                                                                      │   │
│  │    ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐  │   │
│  │    │  Watch  │ ───► │ Compare │ ───► │  Act    │ ───► │ Update  │  │   │
│  │    │ Events  │      │ Desired │      │ Create/ │      │ Status  │  │   │
│  │    │         │      │ vs      │      │ Update/ │      │         │  │   │
│  │    │         │      │ Current │      │ Delete  │      │         │  │   │
│  │    └─────────┘      └─────────┘      └─────────┘      └─────────┘  │   │
│  │         ▲                                                    │      │   │
│  │         └────────────────────────────────────────────────────┘      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Operator manages:                                                          │
│  - Custom Resources (CRDs)                                                  │
│  - Native Resources (Deployments, Services, etc.)                          │
│  - External Resources (Cloud services, databases)                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Building an Operator with Operator SDK

```bash
# Install Operator SDK
brew install operator-sdk

# Create new operator project
operator-sdk init --domain mycompany.com --repo github.com/mycompany/database-operator

# Create API and controller
operator-sdk create api --group database --version v1 --kind Database --resource --controller
```

### Operator Controller Logic

```go
// controllers/database_controller.go
package controllers

import (
    "context"
    "fmt"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    databasev1 "github.com/mycompany/database-operator/api/v1"
)

type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    // Fetch the Database instance
    database := &databasev1.Database{}
    err := r.Get(ctx, req.NamespacedName, database)
    if err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    // Check if StatefulSet exists
    found := &appsv1.StatefulSet{}
    err = r.Get(ctx, req.NamespacedName, found)
    if err != nil && errors.IsNotFound(err) {
        // Create StatefulSet
        sts := r.statefulSetForDatabase(database)
        log.Info("Creating StatefulSet", "Name", sts.Name)
        err = r.Create(ctx, sts)
        if err != nil {
            return ctrl.Result{}, err
        }
        
        // Update status
        database.Status.Phase = "Creating"
        r.Status().Update(ctx, database)
        
        return ctrl.Result{Requeue: true}, nil
    }
    
    // Update status based on StatefulSet
    if found.Status.ReadyReplicas == *found.Spec.Replicas {
        database.Status.Phase = "Running"
        database.Status.Replicas = found.Status.ReadyReplicas
        database.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local", 
            database.Name, database.Namespace)
    } else {
        database.Status.Phase = "Pending"
    }
    r.Status().Update(ctx, database)
    
    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) statefulSetForDatabase(db *databasev1.Database) *appsv1.StatefulSet {
    replicas := int32(db.Spec.Replicas)
    
    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
            OwnerReferences: []metav1.OwnerReference{
                *metav1.NewControllerRef(db, databasev1.GroupVersion.WithKind("Database")),
            },
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app": db.Name},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{"app": db.Name},
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "database",
                        Image: fmt.Sprintf("%s:%s", db.Spec.Engine, db.Spec.Version),
                        Ports: []corev1.ContainerPort{{
                            ContainerPort: 5432,
                        }},
                    }},
                },
            },
        },
    }
}

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Complete(r)
}
```

---

## Day 5-6: Chaos Engineering

### LitmusChaos

```bash
# Install LitmusChaos
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm install chaos litmuschaos/litmus \
  --namespace litmus --create-namespace
```

### Chaos Experiments

```yaml
# Pod delete experiment
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=backend"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
---
# Network latency experiment
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=backend"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: NETWORK_LATENCY
              value: "200"
            - name: TOTAL_CHAOS_DURATION
              value: "120"
---
# CPU stress experiment
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: cpu-stress
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=backend"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: CPU_CORES
              value: "2"
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CPU_LOAD
              value: "100"
```

### Chaos Workflow

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: chaos-workflow
  namespace: litmus
spec:
  entrypoint: chaos-test
  templates:
    - name: chaos-test
      steps:
        - - name: install-experiment
            template: install-chaos
        - - name: run-chaos
            template: pod-delete
        - - name: verify-resilience
            template: verify
        - - name: cleanup
            template: cleanup
    
    - name: install-chaos
      container:
        image: litmuschaos/k8s:latest
        command: ["/bin/sh", "-c"]
        args:
          - kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/pod-delete/experiment.yaml
    
    - name: pod-delete
      container:
        image: litmuschaos/litmus-checker:latest
        args:
          - -file=/tmp/chaosengine.yaml
    
    - name: verify
      container:
        image: curlimages/curl
        command: ["/bin/sh", "-c"]
        args:
          - |
            for i in $(seq 1 10); do
              curl -f http://backend.production/health || exit 1
              sleep 5
            done
```

---

## Day 7-8: Production Incidents & War Rooms

### Incident Response Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INCIDENT RESPONSE LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. DETECT        2. TRIAGE         3. MITIGATE       4. RESOLVE           │
│  ┌─────────┐     ┌─────────┐       ┌─────────┐       ┌─────────┐          │
│  │ Alert   │ ──► │ Assess  │ ──►   │ Stop    │ ──►   │ Fix     │          │
│  │ fires   │     │ severity│       │ bleeding│       │ root    │          │
│  │         │     │ & impact│       │         │       │ cause   │          │
│  └─────────┘     └─────────┘       └─────────┘       └─────────┘          │
│                                                                              │
│  5. POSTMORTEM                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ - Timeline reconstruction                                            │   │
│  │ - Root cause analysis                                                │   │
│  │ - Action items                                                       │   │
│  │ - Lessons learned                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### War Room Simulation 1: OOMKill Cascade

**Scenario**: Order service pods are being OOMKilled, causing cascading failures.

```bash
# Investigation steps
# 1. Check pod status
kubectl get pods -n production -l app=orders
kubectl describe pod orders-xxx -n production | grep -A 10 "Last State"

# 2. Check resource usage
kubectl top pods -n production -l app=orders

# 3. Check events
kubectl get events -n production --sort-by='.lastTimestamp' | grep orders

# 4. Check metrics
# In Grafana: container_memory_usage_bytes{pod=~"orders.*"}

# 5. Check logs before crash
kubectl logs orders-xxx -n production --previous | tail -100

# Mitigation
# 1. Increase memory limits
kubectl patch deployment orders -n production -p '{"spec":{"template":{"spec":{"containers":[{"name":"orders","resources":{"limits":{"memory":"1Gi"}}}]}}}}'

# 2. Scale up to distribute load
kubectl scale deployment orders -n production --replicas=10

# 3. Add circuit breaker to prevent cascade
```

### War Room Simulation 2: DB Connection Pool Exhaustion

**Scenario**: Backend can't connect to database under load.

```bash
# Investigation
# 1. Check backend logs
kubectl logs -l app=backend -n production | grep -i "connection"

# 2. Check database connections
kubectl exec -it postgres-0 -n production -- psql -c "SELECT count(*) FROM pg_stat_activity;"

# 3. Check HPA status
kubectl get hpa backend-hpa -n production

# 4. Check if pods are ready
kubectl get pods -l app=backend -n production -o wide

# Mitigation
# 1. Increase connection pool size
kubectl set env deployment/backend -n production DB_POOL_SIZE=50

# 2. Add connection timeout
kubectl set env deployment/backend -n production DB_CONNECT_TIMEOUT=5000

# 3. Scale database read replicas
kubectl scale statefulset postgres -n production --replicas=3
```

### RCA Template

```markdown
# Incident Report: [Title]

## Summary
- **Date**: YYYY-MM-DD
- **Duration**: X hours Y minutes
- **Severity**: P1/P2/P3
- **Impact**: X% of users affected

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 14:00 | Alert fired: High error rate |
| 14:05 | On-call acknowledged |
| 14:15 | Root cause identified |
| 14:30 | Mitigation applied |
| 15:00 | Service recovered |

## Root Cause
[Detailed explanation of what caused the incident]

## Impact
- X requests failed
- Y users affected
- Z revenue impact

## Mitigation
[What was done to stop the bleeding]

## Resolution
[What was done to fix the root cause]

## Action Items
| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| Increase memory limits | @engineer | 2024-01-15 | Done |
| Add circuit breaker | @engineer | 2024-01-20 | In Progress |
| Improve monitoring | @sre | 2024-01-25 | Pending |

## Lessons Learned
1. [Lesson 1]
2. [Lesson 2]
```

---

## Interview Preparation

### System Design Questions

1. **Design a Kubernetes deployment for a high-traffic e-commerce site**
   - Multi-AZ deployment
   - HPA and Karpenter for scaling
   - Database with read replicas
   - CDN for static assets
   - Circuit breakers and retries

2. **How would you handle a database migration with zero downtime?**
   - Blue-green deployment
   - Feature flags
   - Backward-compatible schema changes
   - Rollback plan

3. **Design a CI/CD pipeline for microservices**
   - GitOps with ArgoCD
   - Canary deployments
   - Automated testing
   - Security scanning

---

## Week 7 Capstone

- Build custom Database Operator
- Run LitmusChaos experiments
- Conduct war room simulations
- Write RCA documents
- Create resume-ready documentation

---

## Key Takeaways

1. **CRDs** extend Kubernetes API
2. **Operators** encode operational knowledge
3. **Chaos engineering** validates resilience
4. **Incident response** requires process
5. **Postmortems** drive improvement
6. **Documentation** is essential

---

## Next Week: AIOps on Kubernetes

Week 8 covers AI workloads and intelligent infrastructure.
