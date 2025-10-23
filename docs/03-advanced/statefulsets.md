# StatefulSets - Managing Stateful Applications

## What is a StatefulSet?

StatefulSets manage stateful applications that require:
- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

**Use Cases:**
- Databases (MySQL, PostgreSQL, MongoDB)
- Distributed systems (Kafka, ZooKeeper, etcd)
- Applications requiring stable hostnames

## StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (web-abc123) | Ordinal (web-0, web-1, web-2) |
| Network ID | Changes on restart | Stable hostname |
| Storage | Shared or random | Dedicated per pod |
| Scaling | Parallel | Ordered (0→1→2) |
| Updates | Can be random | Ordered |

## Basic StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None  # Headless service required!
  selector:
    app: nginx
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx-headless"  # Must match headless service
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
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**Creates:**
- Pods: web-0, web-1, web-2
- PVCs: www-web-0, www-web-1, www-web-2
- DNS: web-0.nginx-headless, web-1.nginx-headless, web-2.nginx-headless

## PostgreSQL StatefulSet Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
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
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 10Gi
```

## Scaling StatefulSets

**Scale up (ordered):**
```bash
kubectl scale statefulset web --replicas=5
# Creates: web-3, then web-4
```

**Scale down (reverse order):**
```bash
kubectl scale statefulset web --replicas=2
# Deletes: web-4, then web-3
```

## Update Strategies

### RollingUpdate (Default)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods
  # ...
```

**Partition for staged rollouts:**
```yaml
partition: 2  # Only update pods >= web-2 (web-2, web-3, ...)
```

### OnDelete

```yaml
updateStrategy:
  type: OnDelete
```

Manual deletion required for updates.

## Best Practices

1. **Always use headless service**
2. **Set pod management policy** for parallel operations
3. **Use init containers** for setup
4. **Configure persistent volumes** properly
5. **Test failure scenarios** (pod deletion, node failure)

## Summary

StatefulSets provide stable identity and storage for stateful workloads. Essential for databases and distributed systems requiring ordered deployment and stable network identities.
