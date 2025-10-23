# Kubernetes Pods - Deep Dive

## What is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster and can contain one or more containers.

### Key Characteristics

- **Atomic Unit**: Pods are created and destroyed as a unit
- **Shared Network**: Containers in a pod share the same IP address and port space
- **Shared Storage**: Containers can share volumes
- **Ephemeral**: Pods are disposable and replaceable
- **Scheduled Together**: All containers in a pod run on the same node

## Pod Anatomy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  namespace: production
  labels:
    app: my-app
    tier: backend
    version: "1.0"
  annotations:
    description: "My application pod"
    created-by: "team-backend"
spec:
  # Container specifications
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
    env:
    - name: ENV_VAR
      value: "production"
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    volumeMounts:
    - name: config
      mountPath: /etc/config
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5

  # Volume specifications
  volumes:
  - name: config
    configMap:
      name: app-config

  # Pod scheduling
  nodeSelector:
    disktype: ssd
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
```

## Pod Lifecycle

### 1. Pending

Pod has been accepted but container images are being downloaded or pod is waiting for scheduling.

```bash
kubectl get pods
# NAME        READY   STATUS    RESTARTS   AGE
# my-app-pod  0/1     Pending   0          10s
```

**Common Reasons:**
- No nodes with sufficient resources
- Image pull in progress
- Waiting for volumes to attach
- Node selector constraints not met

### 2. Running

Pod has been bound to a node and all containers have been created. At least one container is running.

```bash
# NAME        READY   STATUS    RESTARTS   AGE
# my-app-pod  1/1     Running   0          1m
```

### 3. Succeeded

All containers in the pod have terminated successfully (exit code 0).

Used for batch jobs and one-time tasks.

### 4. Failed

All containers have terminated and at least one container terminated with failure (non-zero exit code).

```bash
# NAME        READY   STATUS    RESTARTS   AGE
# my-app-pod  0/1     Error     0          2m
```

### 5. Unknown

The state of the pod could not be obtained (network issues, node problems).

## Multi-Container Patterns

### 1. Sidecar Pattern

Helper container that enhances the main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
  # Main application
  - name: web-app
    image: nginx:1.21
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  # Sidecar: Log shipping
  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
      readOnly: true
    env:
    - name: FLUENT_ELASTICSEARCH_HOST
      value: "elasticsearch.logging.svc.cluster.local"

  volumes:
  - name: logs
    emptyDir: {}
```

**Use Cases:**
- Log aggregation
- Monitoring agents
- Service mesh proxies (Istio, Linkerd)
- Configuration synchronization

### 2. Ambassador Pattern

Proxy that simplifies access to external services.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-proxy
spec:
  containers:
  # Main application
  - name: app
    image: my-app:1.0
    env:
    - name: DB_HOST
      value: "localhost"  # Connect to localhost
    - name: DB_PORT
      value: "5432"

  # Ambassador: Database proxy
  - name: db-proxy
    image: postgres-proxy:latest
    env:
    - name: REAL_DB_HOST
      value: "production-db.aws.com"
    - name: REAL_DB_PORT
      value: "5432"
```

**Use Cases:**
- Database connection pooling
- Service discovery
- Circuit breaking
- Rate limiting

### 3. Adapter Pattern

Standardizes output from the main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  # Main application (legacy format)
  - name: legacy-app
    image: old-app:1.0
    volumeMounts:
    - name: metrics
      mountPath: /var/metrics

  # Adapter: Convert metrics to Prometheus format
  - name: prometheus-adapter
    image: metrics-adapter:latest
    volumeMounts:
    - name: metrics
      mountPath: /var/metrics
      readOnly: true
    ports:
    - containerPort: 9090
      name: prometheus

  volumes:
  - name: metrics
    emptyDir: {}
```

**Use Cases:**
- Log format conversion
- Metric format standardization
- API protocol translation

## Init Containers

Run before app containers start. Used for setup tasks.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  # Wait for database to be ready
  - name: wait-for-db
    image: busybox:1.34
    command:
    - sh
    - -c
    - |
      until nc -z postgres-service 5432; do
        echo "Waiting for database..."
        sleep 2
      done
      echo "Database is ready!"

  # Download configuration
  - name: fetch-config
    image: alpine/git:latest
    command:
    - git
    - clone
    - https://github.com/company/config.git
    - /config
    volumeMounts:
    - name: config
      mountPath: /config

  containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
    - name: config
      mountPath: /etc/app-config

  volumes:
  - name: config
    emptyDir: {}
```

**Common Init Container Uses:**
- Database migration
- Configuration download
- Waiting for dependencies
- Security setup
- Certificate generation

## Resource Management

### Requests vs Limits

```yaml
resources:
  requests:
    memory: "128Mi"  # Guaranteed allocation
    cpu: "250m"      # 0.25 CPU cores
  limits:
    memory: "256Mi"  # Max allowed
    cpu: "500m"      # Max allowed
```

**CPU:**
- 1 CPU = 1000m (millicores)
- Request: Guaranteed CPU time
- Limit: Throttled if exceeded

**Memory:**
- Request: Guaranteed memory
- Limit: Pod killed (OOMKilled) if exceeded

### QoS Classes

Kubernetes assigns QoS based on resources:

**1. Guaranteed** (Highest priority)
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"  # Same as request
    cpu: "500m"      # Same as request
```

**2. Burstable**
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"
  limits:
    memory: "256Mi"  # Different from request
    cpu: "500m"
```

**3. BestEffort** (Lowest priority)
```yaml
# No resources specified
```

## Health Checks

### Liveness Probe

Checks if container is alive. Restarts if failing.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: LivenessCheck
  initialDelaySeconds: 30  # Wait before first check
  periodSeconds: 10        # Check every 10s
  timeoutSeconds: 5        # Timeout after 5s
  successThreshold: 1      # 1 success = healthy
  failureThreshold: 3      # 3 failures = restart
```

**Other Probe Types:**

```yaml
# TCP Socket
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20

# Exec Command
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Readiness Probe

Checks if container is ready to serve traffic. Removes from service if failing.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**Difference from Liveness:**
- Liveness: Restart container if failing
- Readiness: Remove from load balancer if failing

### Startup Probe

For slow-starting containers.

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes to start (30 * 10s)
```

## Pod Security

### Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: my-app:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
```

### Best Practices

1. **Run as non-root**
2. **Read-only root filesystem**
3. **Drop all capabilities**
4. **Use seccomp profile**
5. **Scan images for vulnerabilities**
6. **Use specific image tags** (not :latest)

## Pod Networking

### Pod IP Address

Each pod gets its own IP address:

```bash
kubectl get pods -o wide
# NAME     READY   STATUS    IP           NODE
# web-1    1/1     Running   10.244.1.5   worker-1
# web-2    1/1     Running   10.244.2.3   worker-2
```

### Container Communication

**Within Pod:** localhost

```bash
# Container 1 listens on :8080
# Container 2 connects to localhost:8080
```

**Between Pods:** Pod IP

```bash
# Use Service for stable DNS
curl http://my-service.default.svc.cluster.local
```

## Pod Troubleshooting

### Check Pod Status

```bash
# List pods
kubectl get pods

# Detailed info
kubectl describe pod my-app-pod

# Pod logs
kubectl logs my-app-pod

# Logs for specific container
kubectl logs my-app-pod -c container-name

# Follow logs
kubectl logs -f my-app-pod

# Previous container logs (after crash)
kubectl logs my-app-pod --previous
```

### Common Issues

**ImagePullBackOff:**
```bash
kubectl describe pod my-app-pod
# Events:
#   Failed to pull image "my-app:1.0": rpc error: code = Unknown desc = Error response from daemon: pull access denied

# Solutions:
# - Check image name and tag
# - Verify image registry credentials
# - Check imagePullSecrets
```

**CrashLoopBackOff:**
```bash
kubectl logs my-app-pod --previous

# Common causes:
# - Application error at startup
# - Missing dependencies
# - Invalid configuration
# - Resource limits too low
```

**Pending:**
```bash
kubectl describe pod my-app-pod
# Events:
#   0/3 nodes are available: insufficient cpu

# Solutions:
# - Scale cluster (add nodes)
# - Reduce resource requests
# - Check node selectors/taints
```

## Practical Examples

### Simple Web Server

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
```

### Database Pod with Persistence

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
spec:
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    ports:
    - containerPort: 5432
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"
        cpu: "1000m"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc
```

## Summary

**Key Takeaways:**

- Pods are the atomic unit in Kubernetes
- Pods can contain multiple containers (sidecar pattern)
- Init containers run before app containers
- Use probes for health checking
- Manage resources with requests/limits
- Security context for hardening
- Pods are ephemeral - use controllers for production

**Next Steps:**

- [Deployments](./deployments.md) - Managing pod replicas
- [Services](./services.md) - Exposing pods
- [ConfigMaps & Secrets](./config.md) - Configuration management
