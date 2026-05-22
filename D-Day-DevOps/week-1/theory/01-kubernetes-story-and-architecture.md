# Week 1, Day 1-2: The Story Behind Kubernetes & Architecture Deep Dive

## The Story Behind Kubernetes — The Why Before the How

### The Problem Before Kubernetes

Before Kubernetes, deploying applications was painful:

**Traditional Deployment (Pre-2000s)**
- Applications ran directly on physical servers
- No isolation between apps — one app could starve others of resources
- Scaling meant buying more hardware (weeks/months)
- Disaster recovery was manual and error-prone

**Virtualization Era (2000s-2010s)**
- VMs provided isolation but were heavy (each VM = full OS)
- Better resource utilization but still slow to provision
- VM sprawl became a management nightmare
- Still tied to infrastructure teams for provisioning

**Container Era (2013+)**
- Docker made containers accessible (2013)
- Lightweight, fast startup, consistent environments
- But managing hundreds of containers manually? Chaos.

### Enter Kubernetes (2014)

Google had been running containers at scale internally using a system called **Borg** since 2003. They ran everything — Gmail, Search, YouTube — on Borg.

In 2014, Google open-sourced the lessons learned from Borg as **Kubernetes** (Greek for "helmsman" or "pilot").

**What Kubernetes Solves:**
1. **Container Orchestration** — Automatically schedules containers across machines
2. **Self-Healing** — Restarts failed containers, replaces unhealthy nodes
3. **Scaling** — Scale up/down based on demand (manual or automatic)
4. **Service Discovery** — Containers find each other without hardcoded IPs
5. **Rolling Updates** — Deploy new versions without downtime
6. **Configuration Management** — Separate config from code
7. **Secret Management** — Securely store sensitive data

### Why Companies Use Kubernetes Today

| Company | Scale |
|---------|-------|
| Google | Runs billions of containers weekly |
| Spotify | 10+ million containers |
| Pinterest | 250,000+ pods |
| Airbnb | Migrated entire infrastructure to K8s |

**Key Insight**: Kubernetes is not just about containers — it's about **declaring your desired state** and letting the system figure out how to achieve it.

---

## Kubernetes Architecture Deep Dive

Kubernetes follows a **master-worker** architecture (now called **control plane** and **data plane**).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              KUBERNETES CLUSTER                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         CONTROL PLANE (Master)                         │ │
│  │                                                                        │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │ │
│  │  │  API Server  │  │  Scheduler   │  │  Controller  │  │    etcd    │ │ │
│  │  │              │  │              │  │   Manager    │  │            │ │ │
│  │  │ - REST API   │  │ - Assigns    │  │              │  │ - Key-value│ │ │
│  │  │ - Auth/Authz │  │   pods to    │  │ - Node       │  │   store    │ │ │
│  │  │ - Validation │  │   nodes      │  │ - Replication│  │ - Cluster  │ │ │
│  │  │ - Gateway    │  │ - Resource   │  │ - Endpoint   │  │   state    │ │ │
│  │  │   to cluster │  │   aware      │  │ - Service    │  │ - Raft     │ │ │
│  │  │              │  │              │  │   Account    │  │   consensus│ │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │ │
│  │                                                                        │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │                    Cloud Controller Manager                       │ │ │
│  │  │         (Only in cloud environments - AWS, GCP, Azure)           │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      DATA PLANE (Worker Nodes)                         │ │
│  │                                                                        │ │
│  │  ┌─────────────────────────────┐    ┌─────────────────────────────┐   │ │
│  │  │        Worker Node 1        │    │        Worker Node 2        │   │ │
│  │  │                             │    │                             │   │ │
│  │  │  ┌─────────┐ ┌─────────┐   │    │  ┌─────────┐ ┌─────────┐   │   │ │
│  │  │  │ kubelet │ │kube-proxy│  │    │  │ kubelet │ │kube-proxy│  │   │ │
│  │  │  └─────────┘ └─────────┘   │    │  └─────────┘ └─────────┘   │   │ │
│  │  │                             │    │                             │   │ │
│  │  │  ┌───────────────────────┐ │    │  ┌───────────────────────┐ │   │ │
│  │  │  │  Container Runtime    │ │    │  │  Container Runtime    │ │   │ │
│  │  │  │  (containerd/CRI-O)   │ │    │  │  (containerd/CRI-O)   │ │   │ │
│  │  │  └───────────────────────┘ │    │  └───────────────────────┘ │   │ │
│  │  │                             │    │                             │   │ │
│  │  │  ┌─────┐ ┌─────┐ ┌─────┐  │    │  ┌─────┐ ┌─────┐ ┌─────┐  │   │ │
│  │  │  │ Pod │ │ Pod │ │ Pod │  │    │  │ Pod │ │ Pod │ │ Pod │  │   │ │
│  │  │  └─────┘ └─────┘ └─────┘  │    │  └─────┘ └─────┘ └─────┘  │   │ │
│  │  └─────────────────────────────┘    └─────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

#### 1. API Server (kube-apiserver)
The **front door** to your cluster. Every interaction goes through it.

**Responsibilities:**
- Exposes the Kubernetes REST API
- Authenticates and authorizes requests
- Validates and processes API objects
- The only component that talks to etcd

**Real-world analogy**: The receptionist at a company — all requests go through them first.

```bash
# Every kubectl command talks to the API server
kubectl get pods  # GET request to /api/v1/namespaces/default/pods
kubectl apply -f deployment.yaml  # POST/PUT request
```

#### 2. etcd
A distributed **key-value store** that holds all cluster state.

**Responsibilities:**
- Stores all cluster data (pods, services, secrets, configs)
- Provides consistency using Raft consensus algorithm
- Only the API server reads/writes to etcd

**Critical insight**: If etcd dies and you have no backup, your cluster state is gone. Always backup etcd in production!

```bash
# etcd stores data like:
/registry/pods/default/nginx-pod
/registry/services/default/my-service
/registry/secrets/default/my-secret
```

#### 3. Scheduler (kube-scheduler)
Decides **which node** should run a new pod.

**Scheduling factors:**
- Resource requirements (CPU, memory)
- Node affinity/anti-affinity rules
- Taints and tolerations
- Pod topology spread constraints

**Real-world analogy**: A hotel booking system that assigns guests to rooms based on preferences and availability.

```
Pod needs: 2 CPU, 4GB RAM
Node A: 1 CPU available ❌
Node B: 4 CPU, 8GB available ✅ → Pod scheduled here
```

#### 4. Controller Manager (kube-controller-manager)
Runs **control loops** that watch cluster state and make changes.

**Key controllers:**
- **Node Controller**: Monitors node health, marks unhealthy nodes
- **Replication Controller**: Ensures correct number of pod replicas
- **Endpoint Controller**: Populates endpoint objects (joins Services & Pods)
- **Service Account Controller**: Creates default service accounts

**The reconciliation loop:**
```
while true:
    current_state = observe()
    desired_state = read_from_api()
    if current_state != desired_state:
        take_action_to_reconcile()
```

#### 5. Cloud Controller Manager
Integrates with cloud provider APIs (AWS, GCP, Azure).

**Responsibilities:**
- Node Controller: Checks if cloud node still exists
- Route Controller: Sets up routes in cloud infrastructure
- Service Controller: Creates cloud load balancers

---

### Worker Node Components

#### 1. kubelet
The **agent** running on every worker node.

**Responsibilities:**
- Registers node with the cluster
- Watches for pod assignments from API server
- Ensures containers are running and healthy
- Reports node and pod status back to control plane

**Key insight**: kubelet doesn't manage containers not created by Kubernetes.

```bash
# kubelet runs as a systemd service
systemctl status kubelet
```

#### 2. kube-proxy
Maintains **network rules** on nodes for Service abstraction.

**Modes:**
- **iptables mode** (default): Uses iptables rules for routing
- **IPVS mode**: Uses Linux IPVS for better performance at scale
- **userspace mode** (legacy): Proxies through userspace process

**What it does:**
```
Service: my-app (ClusterIP: 10.96.0.1)
  → Pod 1: 10.244.1.5
  → Pod 2: 10.244.2.8
  → Pod 3: 10.244.1.9

kube-proxy creates rules so traffic to 10.96.0.1 
gets load-balanced across the three pod IPs
```

#### 3. Container Runtime
Actually runs the containers.

**Options:**
- **containerd** (most common, Docker's runtime extracted)
- **CRI-O** (lightweight, designed for Kubernetes)
- Docker (deprecated in K8s 1.24+, but images still work)

---

## How Components Work Together

Let's trace what happens when you run `kubectl apply -f deployment.yaml`:

```
1. kubectl → API Server
   "Create a Deployment with 3 replicas"

2. API Server → etcd
   Validates and stores the Deployment object

3. Controller Manager (Deployment Controller)
   Sees new Deployment, creates ReplicaSet

4. Controller Manager (ReplicaSet Controller)
   Sees ReplicaSet needs 3 pods, creates 3 Pod objects

5. Scheduler
   Sees 3 unscheduled pods, assigns each to a node

6. kubelet (on each assigned node)
   Sees pod assigned to its node, tells container runtime to start containers

7. Container Runtime
   Pulls image, creates and starts container

8. kubelet → API Server
   Reports pod status as "Running"
```

---

## Key Concepts to Remember

### Desired State vs Current State
Kubernetes is a **declarative** system:
- You declare what you want (desired state)
- Kubernetes continuously works to match current state to desired state
- If something fails, K8s automatically tries to fix it

### Everything is an API Object
Pods, Services, Deployments — all are objects stored in etcd and managed via the API.

### Labels are Everything
Labels are key-value pairs that connect objects:
```yaml
# Deployment selects pods with this label
selector:
  matchLabels:
    app: nginx

# Service routes traffic to pods with this label
selector:
  app: nginx
```

---

## Practice Questions

1. What happens if etcd becomes unavailable?
2. Which component decides where a pod runs?
3. What's the difference between kubelet and kube-proxy?
4. Why is the API server called the "gateway" to the cluster?
5. What does the Controller Manager's reconciliation loop do?

---

## Next: Setting Up Kind and Your First Cluster

In the next section, we'll set up a multi-node Kubernetes cluster using Kind and start deploying workloads.
