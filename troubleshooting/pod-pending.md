# Troubleshooting: Pod Stuck in Pending State

## Overview

When a pod remains in "Pending" status, it means the pod has been accepted by the cluster but cannot be scheduled to run on any node.

## Quick Diagnosis

```bash
# Check pod status
kubectl get pods

# Get detailed information
kubectl describe pod <pod-name>

# Check events
kubectl get events --sort-by='.metadata.creationTimestamp'
```

## Common Causes and Solutions

### 1. Insufficient Resources

**Symptom:**
```
Events:
  Warning  FailedScheduling  0/3 nodes are available: insufficient cpu
```

**Diagnosis:**
```bash
# Check node resources
kubectl top nodes

# Check pod resource requests
kubectl describe pod <pod-name> | grep -A 5 "Requests"

# See available resources per node
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Solutions:**

1. **Reduce resource requests:**
```yaml
resources:
  requests:
    memory: "128Mi"  # Reduced from 256Mi
    cpu: "250m"      # Reduced from 500m
```

2. **Add more nodes to cluster:**
```bash
# For cloud providers (example: GKE)
gcloud container clusters resize my-cluster --num-nodes=5

# For managed Kubernetes
# Use your cloud provider's console/CLI
```

3. **Scale down other deployments:**
```bash
kubectl scale deployment <deployment-name> --replicas=1
```

###  2. Node Selector Constraints Not Met

**Symptom:**
```
Events:
  Warning  FailedScheduling  0/3 nodes are available: node(s) didn't match node selector
```

**Diagnosis:**
```bash
# Check pod's node selector
kubectl get pod <pod-name> -o yaml | grep -A 5 nodeSelector

# Check node labels
kubectl get nodes --show-labels
```

**Solutions:**

1. **Remove or fix nodeSelector:**
```yaml
# Before (problematic)
spec:
  nodeSelector:
    disktype: ssd  # No nodes have this label!

# After (fixed)
spec:
  nodeSelector:
    disktype: standard  # Or remove entirely
```

2. **Add required label to nodes:**
```bash
kubectl label nodes <node-name> disktype=ssd
```

### 3. Pod Tolerations and Node Taints

**Symptom:**
```
Events:
  Warning  FailedScheduling  0/3 nodes are available: node(s) had taints that the pod didn't tolerate
```

**Diagnosis:**
```bash
# Check node taints
kubectl describe nodes | grep Taints

# Check pod tolerations
kubectl get pod <pod-name> -o yaml | grep -A 5 tolerations
```

**Solutions:**

1. **Add toleration to pod:**
```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
```

2. **Remove taint from node:**
```bash
kubectl taint nodes <node-name> dedicated=backend:NoSchedule-
```

### 4. PersistentVolumeClaim Not Bound

**Symptom:**
```
Events:
  Warning  FailedScheduling  persistentvolumeclaim "pvc-name" not found
```

**Diagnosis:**
```bash
# Check PVC status
kubectl get pvc

# Describe PVC for details
kubectl describe pvc <pvc-name>

# Check available PVs
kubectl get pv
```

**Solutions:**

1. **Wait for dynamic provisioning:**
```bash
# Check if StorageClass exists
kubectl get storageclass

# Verify PVC will be dynamically provisioned
kubectl describe pvc <pvc-name> | grep "StorageClass"
```

2. **Create PersistentVolume manually (if needed):**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
```

3. **Fix PVC definition:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard  # Make sure this exists!
```

### 5. Image Pull Errors

**Symptom:**
```
Events:
  Warning  Failed  Failed to pull image "myapp:1.0": rpc error: code = Unknown
```

**Diagnosis:**
```bash
# Check image name and tag
kubectl describe pod <pod-name> | grep Image

# Test image pull manually on node
docker pull myapp:1.0
```

**Solutions:**

1. **Fix image name/tag:**
```yaml
containers:
- name: app
  image: docker.io/myorg/myapp:1.0  # Full path
```

2. **Add imagePullSecrets:**
```yaml
spec:
  imagePullSecrets:
  - name: registry-credentials
  containers:
  - name: app
    image: myregistry.com/myapp:1.0
```

Create the secret:
```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com
```

### 6. Pod Security Policy Violations

**Symptom:**
```
Events:
  Warning  FailedCreate  Error creating: pods "pod-name" is forbidden: unable to validate against any pod security policy
```

**Solutions:**

1. **Modify pod security context:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

2. **Create appropriate PSP or use Pod Security Admission:**
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'secret'
```

### 7. Scheduler Not Running

**Symptom:**
No scheduling events at all

**Diagnosis:**
```bash
# Check scheduler pod
kubectl get pods -n kube-system | grep scheduler

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-<node-name>
```

**Solutions:**

1. **Restart scheduler (if managed cluster, contact support)**
2. **Check scheduler configuration**
3. **Verify control plane health**

## Complete Troubleshooting Workflow

```bash
# 1. Get pod details
kubectl describe pod <pod-name>

# 2. Check recent events
kubectl get events --field-selector involvedObject.name=<pod-name> --sort-by='.lastTimestamp'

# 3. Check node resources
kubectl describe nodes

# 4. Check if it's a scheduling issue
kubectl get pods -o wide

# 5. Try to manually schedule (debugging)
kubectl get pod <pod-name> -o yaml > pod.yaml
# Edit pod.yaml to add nodeName: <specific-node>
kubectl delete pod <pod-name>
kubectl create -f pod.yaml

# 6. Check control plane
kubectl get componentstatuses
```

## Prevention Best Practices

1. **Always set resource requests and limits**
2. **Use Pod Disruption Budgets**
3. **Monitor cluster capacity**
4. **Use Cluster Autoscaler**
5. **Test pod definitions before deploying**
6. **Use admission controllers for validation**

## Quick Reference Commands

```bash
# View all pending pods
kubectl get pods --field-selector=status.phase=Pending

# Get detailed events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check scheduler
kubectl get events -n kube-system --field-selector involvedObject.name=kube-scheduler

# Force delete stuck pod
kubectl delete pod <pod-name> --force --grace-period=0
```

## Summary

Most pending pod issues fall into these categories:
1. Resource constraints (most common)
2. Node selection issues
3. Storage problems
4. Image pull failures
5. Security policy violations

Always start with `kubectl describe pod` and read the Events section carefully.
