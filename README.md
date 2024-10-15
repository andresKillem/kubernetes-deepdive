# Kubernetes Deep Dive

A comprehensive, hands-on guide to mastering Kubernetes from fundamentals to advanced production patterns.

## Overview

This repository is a complete learning path for Kubernetes, built from real-world production experience. Each concept includes theory, diagrams, and working examples.

## Learning Path

### Level 1: Fundamentals (Weeks 1-2)
- [x] [What is Kubernetes?](./docs/01-fundamentals/what-is-kubernetes.md)
- [x] [Architecture Overview](./docs/01-fundamentals/architecture.md)
- [x] [Pods: The Building Block](./docs/01-fundamentals/pods.md)
- [x] [Services and Networking](./docs/01-fundamentals/services.md)
- [x] [Deployments and ReplicaSets](./docs/01-fundamentals/deployments.md)

### Level 2: Core Concepts (Weeks 3-4)
- [ ] [ConfigMaps and Secrets](./docs/02-core-concepts/configmaps-secrets.md)
- [ ] [Persistent Volumes](./docs/02-core-concepts/storage.md)
- [ ] [Namespaces and Resource Quotas](./docs/02-core-concepts/namespaces.md)
- [ ] [Labels and Selectors](./docs/02-core-concepts/labels.md)
- [ ] [Ingress Controllers](./docs/02-core-concepts/ingress.md)

### Level 3: Advanced Workloads (Weeks 5-6)
- [ ] [StatefulSets](./docs/03-advanced/statefulsets.md)
- [ ] [DaemonSets](./docs/03-advanced/daemonsets.md)
- [ ] [Jobs and CronJobs](./docs/03-advanced/jobs.md)
- [ ] [Horizontal Pod Autoscaling](./docs/03-advanced/hpa.md)
- [ ] [Vertical Pod Autoscaling](./docs/03-advanced/vpa.md)

### Level 4: Security (Weeks 7-8)
- [ ] [RBAC Deep Dive](./docs/04-security/rbac.md)
- [ ] [Network Policies](./docs/04-security/network-policies.md)
- [ ] [Pod Security Standards](./docs/04-security/pod-security.md)
- [ ] [Secrets Management](./docs/04-security/secrets-management.md)
- [ ] [Security Best Practices](./docs/04-security/best-practices.md)

### Level 5: Production Patterns (Weeks 9-10)
- [ ] [GitOps with ArgoCD](./docs/05-production/gitops.md)
- [ ] [Service Mesh (Istio/Linkerd)](./docs/05-production/service-mesh.md)
- [ ] [Observability Stack](./docs/05-production/observability.md)
- [ ] [Multi-tenancy](./docs/05-production/multi-tenancy.md)
- [ ] [Disaster Recovery](./docs/05-production/disaster-recovery.md)

### Level 6: Advanced Topics (Weeks 11-12)
- [ ] [Custom Resource Definitions](./docs/06-advanced-topics/crds.md)
- [ ] [Operators and Controllers](./docs/06-advanced-topics/operators.md)
- [ ] [Cluster API](./docs/06-advanced-topics/cluster-api.md)
- [ ] [Multi-cluster Management](./docs/06-advanced-topics/multi-cluster.md)
- [ ] [FinOps for Kubernetes](./docs/06-advanced-topics/finops.md)

## Hands-On Labs

Each concept includes practical exercises:

```
labs/
├── 01-setup/              # Local cluster setup (kind, minikube)
├── 02-deployments/        # Deploy your first app
├── 03-networking/         # Service discovery and load balancing
├── 04-storage/            # Stateful applications
├── 05-security/           # RBAC and network policies
├── 06-monitoring/         # Prometheus and Grafana
├── 07-gitops/             # ArgoCD deployment
└── 08-troubleshooting/    # Common issues and solutions
```

## Quick Start

### Prerequisites
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"

# Install kind (Kubernetes in Docker)
brew install kind

# Install helm
brew install helm
```

### Create Your First Cluster
```bash
# Create a local cluster
kind create cluster --name learning-cluster --config configs/kind-config.yaml

# Verify
kubectl cluster-info
kubectl get nodes
```

### Deploy Your First Application
```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:latest

# Expose it as a service
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Check status
kubectl get deployments,pods,services
```

## Repository Structure

```
kubernetes-deepdive/
├── docs/                  # Comprehensive documentation
│   ├── 01-fundamentals/
│   ├── 02-core-concepts/
│   ├── 03-advanced/
│   ├── 04-security/
│   ├── 05-production/
│   └── 06-advanced-topics/
├── examples/              # YAML manifests and code
│   ├── basic/
│   ├── intermediate/
│   └── advanced/
├── labs/                  # Hands-on exercises
├── diagrams/              # Architecture diagrams
├── scripts/               # Helper scripts
└── troubleshooting/       # Common issues guide
```

## Key Concepts Visualized

### Kubernetes Architecture
```
┌─────────────────────────────────────────────────┐
│              Control Plane                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │   API    │  │Scheduler │  │Controller│     │
│  │  Server  │  │          │  │ Manager  │     │
│  └──────────┘  └──────────┘  └──────────┘     │
│  ┌──────────┐                                   │
│  │  etcd    │                                   │
│  └──────────┘                                   │
└─────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
┌───────▼──────┐ ┌───▼────────┐ ┌──▼────────┐
│   Node 1     │ │   Node 2   │ │  Node 3   │
│ ┌──────────┐ │ │┌──────────┐│ │┌──────────┐│
│ │  kubelet │ │ ││  kubelet ││ ││  kubelet ││
│ └──────────┘ │ │└──────────┘│ │└──────────┘│
│ ┌──────────┐ │ │┌──────────┐│ │┌──────────┐│
│ │Pod│Pod   │ │ ││Pod│Pod   ││ ││Pod│Pod   ││
│ └──────────┘ │ │└──────────┘│ │└──────────┘│
└──────────────┘ └────────────┘ └────────────┘
```

### Pod Lifecycle
```
Pending → ContainerCreating → Running → Succeeded/Failed
```

## Real-World Production Examples

- **Multi-tier Application**: [examples/real-world/microservices/](./examples/real-world/microservices/)
- **Database Cluster**: [examples/real-world/postgres-ha/](./examples/real-world/postgres-ha/)
- **CI/CD Pipeline**: [examples/real-world/tekton-pipelines/](./examples/real-world/tekton-pipelines/)
- **Monitoring Stack**: [examples/real-world/monitoring/](./examples/real-world/monitoring/)

## Common Patterns

### Blue-Green Deployment
### Canary Deployment
### Circuit Breaker
### Sidecar Pattern
### Ambassador Pattern
### Init Container Pattern

## Troubleshooting Guide

- [Pod is Pending](./troubleshooting/pod-pending.md)
- [CrashLoopBackOff](./troubleshooting/crashloop.md)
- [Image Pull Errors](./troubleshooting/image-pull.md)
- [Resource Limits](./troubleshooting/resource-limits.md)
- [Network Issues](./troubleshooting/networking.md)

## Tools Covered

- **kubectl**: Command-line tool
- **helm**: Package manager
- **kustomize**: Configuration management
- **ArgoCD**: GitOps continuous delivery
- **Prometheus**: Monitoring
- **Grafana**: Visualization
- **Istio**: Service mesh
- **cert-manager**: Certificate management
- **external-secrets**: Secrets management
- **kyverno**: Policy management

## Resources

- [Official Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes Patterns Book](https://www.oreilly.com/library/view/kubernetes-patterns/9781492050278/)

## Certifications Path

1. **CKAD** (Certified Kubernetes Application Developer)
2. **CKA** (Certified Kubernetes Administrator)
3. **CKS** (Certified Kubernetes Security Specialist)

## Contributing

This is a personal learning repository, but feedback and suggestions are welcome!

## Author

**Andrés Muñoz** - Principal DevOps Architect
- 10+ years in infrastructure and cloud platforms
- AWS, Kubernetes, Terraform specialist
- [GitHub](https://github.com/andresKillem)

---

*Last Updated: October 2025*
