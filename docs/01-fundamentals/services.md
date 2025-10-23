# Kubernetes Services - Networking Deep Dive

## What is a Service?

A Service is an abstraction that defines a logical set of Pods and a policy to access them. Services provide stable networking for ephemeral pods.

### Why Services?

**Problem:** Pods are ephemeral - they get replaced, IP addresses change.

```
Before Service:
App → Pod IP (10.244.1.5) → Pod crashes → New Pod IP (10.244.2.3) → App breaks!

With Service:
App → Service IP (10.96.0.100) → Always routes to healthy pods
```

## Service Types

### 1. ClusterIP (Default)

Exposes service on an internal IP. Only accessible within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80          # Service port
    targetPort: 8080  # Pod port
    protocol: TCP
    name: http
```

**Use Cases:**
- Internal microservices communication
- Database services
- Cache services

**Access:**
```bash
# From within cluster
curl http://backend-service.default.svc.cluster.local
curl http://backend-service  # Same namespace
curl http://10.96.0.100      # ClusterIP
```

### 2. NodePort

Exposes service on each Node's IP at a static port (30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Optional, auto-assigned if omitted
```

**How it works:**
```
Internet → Node IP:30080 → Service → Pods
```

**Access:**
```bash
# External access
curl http://<node-ip>:30080
curl http://worker-1:30080
curl http://worker-2:30080  # Works on any node!
```

**Use Cases:**
- Development/testing
- When LoadBalancer not available
- Direct node access needed

### 3. LoadBalancer

Creates an external load balancer (cloud provider required).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  loadBalancerSourceRanges:  # IP whitelist
  - 203.0.113.0/24
```

**How it works:**
```
Internet → Cloud LB (34.123.45.67) → NodePort → Service → Pods
```

**Access:**
```bash
kubectl get svc public-web
# NAME         TYPE           EXTERNAL-IP      PORT(S)
# public-web   LoadBalancer   34.123.45.67     80:31234/TCP

curl http://34.123.45.67
```

**Use Cases:**
- Production web applications
- Public APIs
- Any service needing external access

### 4. ExternalName

Maps service to external DNS name (no proxy).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: prod-db.external.com
```

**Use Cases:**
- Accessing external services
- Migration from external to internal
- Multi-cluster communication

## Service Discovery

### DNS-Based Discovery

Kubernetes automatically creates DNS records:

```
<service-name>.<namespace>.svc.cluster.local
```

**Examples:**
```bash
# Same namespace
curl http://backend-service

# Different namespace
curl http://backend-service.production.svc.cluster.local

# Headless service (see below)
curl http://pod-1.backend-service.production.svc.cluster.local
```

### Environment Variables

Kubernetes injects service info into pods:

```bash
# For service "backend-service"
BACKEND_SERVICE_SERVICE_HOST=10.96.0.100
BACKEND_SERVICE_SERVICE_PORT=80
```

**Note:** Pods must be created after the service!

## Advanced Service Patterns

### Headless Service

No ClusterIP assigned. Returns pod IPs directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
spec:
  clusterIP: None  # Headless!
  selector:
    app: database
  ports:
  - port: 5432
```

**DNS returns pod IPs:**
```bash
nslookup stateful-service
# Server: 10.96.0.10
# Address: 10.244.1.5  (pod-1)
# Address: 10.244.2.3  (pod-2)
# Address: 10.244.3.7  (pod-3)
```

**Use Cases:**
- StatefulSets
- Client-side load balancing
- Direct pod communication needed

### Multi-Port Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

### Session Affinity

Route requests from same client to same pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: web
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
  ports:
  - port: 80
    targetPort: 8080
```

**Use Cases:**
- Shopping carts
- User sessions (if not using external session store)
- WebSocket connections

### External IPs

Map service to specific external IPs:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-ip-service
spec:
  selector:
    app: web
  ports:
  - port: 80
  externalIPs:
  - 203.0.113.10
  - 203.0.113.11
```

## Service Implementation (kube-proxy)

### iptables Mode (Default)

```bash
# kube-proxy creates iptables rules
sudo iptables-save | grep my-service
# -A KUBE-SERVICES -d 10.96.0.100/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-ABC123
# -A KUBE-SVC-ABC123 -m statistic --mode random --probability 0.33 -j KUBE-SEP-POD1
# -A KUBE-SVC-ABC123 -m statistic --mode random --probability 0.50 -j KUBE-SEP-POD2
# -A KUBE-SVC-ABC123 -j KUBE-SEP-POD3
```

**Pros:**
- Mature, stable
- Low CPU usage

**Cons:**
- Complex iptables rules
- Hard to debug
- Rule updates not atomic

### IPVS Mode (Recommended for Large Clusters)

```yaml
# ConfigMap for kube-proxy
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      scheduler: "rr"  # Round Robin
```

**Load Balancing Algorithms:**
- `rr` - Round Robin
- `lc` - Least Connection
- `wrr` - Weighted Round Robin
- `sh` - Source Hashing

**Pros:**
- Better performance
- More load-balancing algorithms
- Better scalability

## Endpoints and EndpointSlices

Services don't route directly to pods. They use Endpoints.

```yaml
# Automatically created
apiVersion: v1
kind: Endpoints
metadata:
  name: backend-service
subsets:
- addresses:
  - ip: 10.244.1.5
    nodeName: worker-1
  - ip: 10.244.2.3
    nodeName: worker-2
  ports:
  - port: 8080
    protocol: TCP
```

**Check endpoints:**
```bash
kubectl get endpoints backend-service
# NAME              ENDPOINTS                          AGE
# backend-service   10.244.1.5:8080,10.244.2.3:8080   5m

# Describe for more details
kubectl describe endpoints backend-service
```

**EndpointSlices** (newer, more scalable):
```bash
kubectl get endpointslices
```

## Service Without Selector

Manually control endpoints:

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  ports:
  - port: 5432
    targetPort: 5432
---
# Manual Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: external-database
subsets:
- addresses:
  - ip: 192.168.1.100  # External DB
  ports:
  - port: 5432
```

**Use Cases:**
- External databases
- Services in other clusters
- Migration scenarios

## Traffic Policies

### External Traffic Policy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # or Cluster (default)
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Cluster (default):**
- Load balanced across all pods
- May add extra hop
- Source IP is SNAT'd

**Local:**
- Only route to pods on same node
- Preserves source IP
- No extra hop
- May cause imbalance

### Internal Traffic Policy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  internalTrafficPolicy: Local  # or Cluster (default)
  selector:
    app: backend
  ports:
  - port: 80
```

## Troubleshooting Services

### Service Not Accessible

```bash
# 1. Check service exists
kubectl get svc my-service

# 2. Check selector matches pods
kubectl get pods --selector=app=myapp

# 3. Check endpoints
kubectl get endpoints my-service

# 4. Verify pod labels
kubectl get pods --show-labels

# 5. Test from within cluster
kubectl run -it --rm debug --image=alpine --restart=Never -- sh
/ # wget -qO- http://my-service
```

### No Endpoints

```bash
kubectl describe svc my-service
# Endpoints: <none>

# Causes:
# 1. No pods match selector
kubectl get pods -l app=myapp

# 2. Pods not ready
kubectl get pods
# NAME    READY   STATUS
# pod-1   0/1     Running  # Not ready!

# 3. Check readiness probe
kubectl describe pod pod-1
```

### Wrong Port

```bash
# Service port != Target port
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  ports:
  - port: 80        # Service listens on 80
    targetPort: 8080 # Pods listen on 8080 ✓

# Test from pod
kubectl exec -it my-pod -- curl http://web:80
```

## Best Practices

### 1. Use Descriptive Names

```yaml
# Good
name: user-api-service
name: postgres-primary-service

# Bad
name: service1
name: svc
```

### 2. Label Selectors

```yaml
selector:
  app: backend
  tier: api
  version: v2
```

### 3. Named Ports

```yaml
ports:
- name: http  # Named!
  port: 80
  targetPort: http  # Reference by name

# In pod:
containers:
- name: app
  ports:
  - name: http
    containerPort: 8080
```

### 4. Health Checks

Ensure readiness probes are configured:

```yaml
# Pod
containers:
- name: app
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
```

### 5. Resource Limits

Apply to pods, not services:

```yaml
containers:
- name: app
  resources:
    limits:
      cpu: "1"
      memory: "512Mi"
```

## Complete Example

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - name: http
          containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
    tier: frontend
  ports:
  - name: http
    port: 80
    targetPort: http
  sessionAffinity: None
```

## Summary

**Key Takeaways:**

- Services provide stable networking for pods
- ClusterIP for internal, LoadBalancer for external
- DNS-based service discovery
- kube-proxy implements service routing
- Use selectors to automatically create endpoints
- Session affinity for stateful apps
- External traffic policy affects load balancing

**Service Type Decision Tree:**

```
Need external access?
├─ No → ClusterIP
└─ Yes
   ├─ Cloud LB available? → LoadBalancer
   ├─ Development/testing? → NodePort
   └─ Ingress available? → ClusterIP + Ingress
```

**Next Steps:**

- [Ingress](../02-networking/ingress.md) - HTTP(S) routing
- [Network Policies](../02-networking/network-policies.md) - Traffic control
- [DNS](../02-networking/dns.md) - Service discovery details
