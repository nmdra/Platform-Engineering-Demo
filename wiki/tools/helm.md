---
tags: [tool, helm, kubernetes, package-manager]
created: 2026-05-30
---

# Helm

**Helm** is the package manager for Kubernetes. It bundles Kubernetes manifests into reusable **charts** — versioned, configurable templates that can be installed with a single command. In FinEdge, Helm installs [[Crossplane]], [[Kyverno]], [[ArgoCD]], and [[Prometheus and Grafana]].

## Why Helm?

Without Helm, installing Crossplane would require:
- Downloading 20+ YAML manifests
- Editing each one for your configuration
- Applying them in the right order
- Tracking what version you installed

With Helm:
```bash
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace \
  --version 2.3.1
# Done. Everything deployed, configured, and versioned.
```

## Core Concepts

### Charts
A chart is a package of Kubernetes templates:
```
crossplane-chart/
├── Chart.yaml           # Name, version, description
├── values.yaml          # Default configuration
├── templates/
│   ├── deployment.yaml  # Deployment template
│   ├── service.yaml     # Service template
│   ├── rbac.yaml        # RBAC template
│   └── ...
```

### Values
Values customize a chart's behavior without editing templates:
```yaml
# Custom values.yaml
replicas: 2
args:
  - --poll-interval=30s
  - --max-reconcile-rate=10
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

### Repositories
Charts are stored in repositories:
```bash
# Add a repository
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add kyverno https://kyverno.github.io/kyverno
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update local cache
helm repo update

# Search for charts
helm search repo crossplane
```

## FinEdge Helm Installations

### Crossplane
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 2.3.1 \
  --set args='{"--poll-interval=30s"}'
```

### ArgoCD
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.x.x \
  --set server.service.type=NodePort \
  --set server.service.nodePortHttps=30080
```

### Kyverno
```bash
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --version 3.x.x
```

### Prometheus + Grafana
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30300
```

## Common Operations

```bash
# List installed releases
helm list -A

# Check release status
helm status crossplane -n crossplane-system

# See installed values
helm get values crossplane -n crossplane-system

# Upgrade with new values
helm upgrade crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --version 2.3.1 \
  --set args='{"--poll-interval=15s"}'

# Rollback to previous version
helm rollback crossplane 1 -n crossplane-system

# Uninstall
helm uninstall crossplane -n crossplane-system

# Dry-run (see what would be deployed)
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --dry-run --debug
```

## Version Pinning

**Always pin chart versions.** Unpinned versions can break your platform:

```bash
# ✅ Good — pinned version
helm install crossplane crossplane-stable/crossplane --version 2.3.1

# ❌ Bad — uses latest
helm install crossplane crossplane-stable/crossplane
```

## Helm vs. kubectl apply

| Aspect | Helm | kubectl apply |
|--------|------|---------------|
| Versioning | Built-in (release history) | Manual |
| Rollback | `helm rollback` | Manual `kubectl apply` of old files |
| Configuration | `values.yaml` overrides | Edit YAML directly |
| Dependencies | Charts can depend on other charts | Manual |
| Templating | Go templates | None (or Kustomize) |
| FinEdge use | Install infrastructure components | Apply project-specific resources |

## Related

- [[Crossplane]]
- [[Kyverno]]
- [[Prometheus and Grafana]]
- [[ArgoCD]]
