# Kubernetes Architecture

## Overview

Kubernetes follows a master-worker architecture with a declarative API. You describe the desired state, and Kubernetes works to make it reality.

## Control Plane Components

The control plane makes global decisions about the cluster and detects/responds to cluster events.

### 1. API Server (kube-apiserver)

**Purpose**: Front-end for the Kubernetes control plane

**What it does:**
- Exposes the Kubernetes API (REST interface)
- Validates and processes REST requests
- Updates etcd with the cluster state
- Only component that talks to etcd

**Example Interaction:**
```bash
# When you run this command:
kubectl create deployment nginx --image=nginx

# Behind the scenes:
1. kubectl sends HTTPS POST to API server
2. API server authenticates and authorizes the request
3. API server validates the deployment spec
4. API server writes to etcd
5. API server returns response to kubectl
```

**API Groups:**
```yaml
# Core group (legacy)
apiVersion: v1
kind: Pod

# Named groups (modern)
apiVersion: apps/v1
kind: Deployment

apiVersion: batch/v1
kind: Job

apiVersion: networking.k8s.io/v1
kind: Ingress
```

### 2. etcd

**Purpose**: Distributed key-value store for cluster data

**What it stores:**
- Cluster state and configuration
- All resource definitions (Pods, Services, etc.)
- Secrets and ConfigMaps
- Network configuration

**Data Structure:**
```
/registry/
├── pods/
│   ├── default/nginx-abc123
│   └── kube-system/coredns-xyz789
├── services/
│   └── default/nginx-service
├── deployments/
│   └── default/nginx
└── configmaps/
    └── kube-system/kubeadm-config
```

**Critical Points:**
- Single source of truth
- Must be backed up regularly
- Uses Raft consensus algorithm
- Requires odd number of instances (3 or 5 in production)

### 3. Scheduler (kube-scheduler)

**Purpose**: Assigns pods to nodes

**Decision Process:**
```
New Pod Created
    ↓
1. Filtering (which nodes CAN run this pod?)
    ├── Node has enough CPU/memory?
    ├── Node matches nodeSelector?
    ├── Node satisfies tolerations?
    └── Port available on node?
    ↓
2. Scoring (which node is BEST?)
    ├── Resource balance
    ├── Pod affinity/anti-affinity
    ├── Data locality
    └── Taints and tolerations
    ↓
3. Binding
    └── Update Pod.spec.nodeName in etcd
```

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  # Before scheduling:
  # nodeName: <empty>

  # After scheduling:
  # nodeName: node-1

  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"  # Scheduler uses this!
```

### 4. Controller Manager (kube-controller-manager)

**Purpose**: Runs controller processes that regulate cluster state

**Key Controllers:**

**Node Controller**
- Monitors node health
- Marks nodes as NotReady after 40s of no heartbeat
- Evicts pods after 5 minutes

**Replication Controller**
- Ensures correct number of pod replicas
- Creates/deletes pods to match desired count

**Endpoints Controller**
- Populates Endpoints object (joins Services & Pods)

**Service Account Controller**
- Creates default ServiceAccounts for namespaces
- Creates API access tokens

**Control Loop Pattern:**
```python
# Pseudocode for a controller

while True:
    desired_state = get_from_etcd()
    current_state = observe_cluster()

    if current_state != desired_state:
        make_changes_to_match(desired_state)

    sleep(reconciliation_interval)
```

**Example: ReplicaSet Controller**
```yaml
# You declare:
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3  # Desired state

# Controller ensures:
# - If 2 pods exist → Create 1 more
# - If 4 pods exist → Delete 1
# - If pod crashes → Create replacement
```

### 5. Cloud Controller Manager (optional)

**Purpose**: Integrates with cloud provider APIs

**Responsibilities:**
- Node Controller: Check if deleted nodes are removed from cloud
- Route Controller: Set up routes in cloud infrastructure
- Service Controller: Create cloud load balancers for LoadBalancer services

## Node Components

Components that run on every node in the cluster.

### 1. Kubelet

**Purpose**: Agent that ensures containers are running in pods

**Responsibilities:**
- Registers node with API server
- Watches API server for pod assignments
- Pulls container images
- Starts/stops containers (via container runtime)
- Reports pod and node status
- Runs liveness/readiness probes

**Pod Lifecycle Management:**
```
API Server: "Run pod nginx on this node"
    ↓
Kubelet:
1. Pull image nginx:latest
2. Create container with specified config
3. Start container
4. Monitor container health
5. Report status back to API server
    ↓
API Server: Updates pod status in etcd
```

**Health Checks:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: myapp:1.0

    # Kubelet runs these probes:
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 2. Kube-proxy

**Purpose**: Network proxy that implements Kubernetes Service concept

**What it does:**
- Maintains network rules on nodes
- Implements virtual IPs for Services
- Handles load balancing between pod backends

**Modes:**

**iptables mode** (default):
```bash
# Creates iptables rules like:
-A KUBE-SERVICES -d 10.96.0.1/32 -p tcp -m tcp --dport 443 -j KUBE-SVC-NGINX

# Translates to:
Traffic to Service IP:Port → Random Pod IP:Port
```

**IPVS mode** (recommended for large clusters):
- Uses Linux IP Virtual Server
- Better performance
- More load-balancing algorithms

**Example:**
```yaml
# You create a service:
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080

# kube-proxy creates rules:
# ClusterIP: 10.96.0.100 (virtual IP)
#   ↓
# Routes to pod IPs: 192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080
```

### 3. Container Runtime

**Purpose**: Software responsible for running containers

**Supported Runtimes:**
- **containerd** (default in most clusters)
- **CRI-O**
- **Docker** (deprecated in K8s 1.24+, use containerd)

**Container Runtime Interface (CRI):**
```
Kubelet → CRI → Container Runtime → Containers
```

## Add-ons

Optional components that provide cluster features.

### DNS (CoreDNS)

Provides DNS-based service discovery:
```bash
# Pod can reach service by name:
curl http://nginx-service

# Fully qualified:
curl http://nginx-service.default.svc.cluster.local
```

### Dashboard

Web-based UI for cluster management

### Ingress Controller

Manages external access to services (nginx, traefik, etc.)

### Metrics Server

Collects resource metrics for autoscaling

## Communication Flow

### Creating a Deployment (Complete Flow)

```
1. User: kubectl create deployment nginx --image=nginx
    ↓
2. kubectl → API Server (HTTPS POST)
    ↓
3. API Server:
   - Authenticates request
   - Authorizes (RBAC check)
   - Validates Deployment spec
   - Writes to etcd
    ↓
4. Deployment Controller (watching API):
   - Sees new Deployment
   - Creates ReplicaSet
    ↓
5. ReplicaSet Controller (watching API):
   - Sees new ReplicaSet with replicas: 1
   - Creates Pod
    ↓
6. Scheduler (watching API):
   - Sees unscheduled Pod
   - Selects best node (node-1)
   - Updates Pod.spec.nodeName
    ↓
7. Kubelet on node-1 (watching API):
   - Sees Pod assigned to it
   - Pulls image nginx
   - Creates container
   - Reports status to API Server
    ↓
8. API Server:
   - Updates Pod status in etcd
   - Pod status: Running
```

## High Availability Architecture

Production setup with 3 control plane nodes:

```
┌─────────────────────────────────────────────┐
│          Load Balancer (keepalived)         │
│              VIP: 192.168.1.100             │
└────────────┬───────────┬───────────────────┘
             │           │
    ┌────────┴─────┐  ┌──┴───────────┐  ┌──────────────┐
    │   Master 1   │  │   Master 2   │  │   Master 3   │
    │ API│Sched│CM │  │ API│Sched│CM │  │ API│Sched│CM │
    └───┬──────────┘  └──┬───────────┘  └──┬───────────┘
        │                │                  │
    ┌───┴────────────────┴──────────────────┴───┐
    │           etcd cluster (3 nodes)           │
    │        (Raft consensus, quorum=2)          │
    └────────────────────────────────────────────┘
```

## Summary

| Component | Runs On | Purpose |
|-----------|---------|---------|
| API Server | Control Plane | Entry point for all REST commands |
| etcd | Control Plane | Stores cluster state |
| Scheduler | Control Plane | Assigns pods to nodes |
| Controller Manager | Control Plane | Reconciles desired vs actual state |
| Kubelet | Every Node | Runs containers |
| Kube-proxy | Every Node | Network proxy for Services |
| Container Runtime | Every Node | Actually runs containers |

## Key Takeaways

1. **Declarative Model**: You declare desired state, controllers work to achieve it
2. **Watch Pattern**: Controllers watch API server for changes
3. **Reconciliation Loop**: Controllers constantly reconcile current vs desired state
4. **API-Centric**: Everything goes through API server
5. **Distributed**: Multiple components work together autonomously
6. **Self-Healing**: Controllers automatically fix issues

## Next Steps

- [Understanding Pods](./pods.md)
- [Services and Networking](./services.md)
- [Deployments Deep Dive](./deployments.md)
