---
tags: [tool, kind, kubernetes, development]
created: 2026-05-30
---

# kind

**kind** (Kubernetes IN Docker) is a tool for running local Kubernetes clusters using Docker containers as "nodes." In FinEdge, it provides the development cluster where [[Crossplane]], [[ArgoCD]], [[Kyverno]], and all team workloads run — no cloud account needed.

## Why kind?

| Feature | kind | minikube | k3s |
|---------|------|----------|-----|
| Multi-node | ✅ | ❌ (single VM) | ✅ |
| Docker-based | ✅ | VM-based | Binary |
| CI-friendly | ✅ (no VM) | ❌ | ✅ |
| Speed | Fast | Slow (VM boot) | Fast |
| Feature parity | Full K8s | Full K8s | Lightweight K8s |
| FinEdge choice | ✅ | ❌ | ❌ |

## FinEdge Cluster Configuration

```yaml
# kind/cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      # ArgoCD UI
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
      # Grafana
      - containerPort: 30300
        hostPort: 30300
        protocol: TCP
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
  - role: worker
    labels:
      workload: team-apps
  - role: worker
    labels:
      workload: platform
```

### Key Configuration Points

#### `extraPortMappings`
Maps container ports to host ports so you can access services from your browser:
```yaml
extraPortMappings:
  - containerPort: 30080    # NodePort inside kind
    hostPort: 30080          # Port on your laptop
```

#### Multi-Node
Running multiple nodes tests realistic scheduling:
```yaml
nodes:
  - role: control-plane    # Runs control plane + etcd
  - role: worker           # Runs team workloads
  - role: worker           # Runs platform components
```

#### `kubeadmConfigPatches`
Customize kubelet and API server configuration:
```yaml
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        # Enable admission plugins
        enable-admission-plugins: NodeRestriction,PodSecurity
```

## Common Operations

```bash
# Create cluster with config
kind create cluster --name finedge --config kind/cluster-config.yaml

# Create cluster with specific Kubernetes version
kind create cluster --name finedge --image kindest/node:v1.31.0

# List clusters
kind get clusters

# Get kubeconfig
kind get kubeconfig --name finedge

# Delete cluster
kind delete cluster --name finedge

# Load a local Docker image into kind
kind load docker-image my-app:latest --name finedge
```

## Networking: `host.docker.internal` on Linux

This is the **most common issue** on Linux. kind nodes need to reach [[LocalStack]] running on the host:

### The Problem
On macOS/Windows, Docker Desktop provides `host.docker.internal` automatically. On Linux with Docker Engine, it doesn't exist by default.

### Solution 1: Docker's `--add-host` (for LocalStack container)
```bash
docker run -d \
  --name localstack \
  --add-host=host.docker.internal:host-gateway \
  -p 4566:4566 \
  localstack/localstack
```

### Solution 2: Use Docker bridge IP
```bash
# Find the bridge IP
ip addr show docker0 | grep inet
# Usually 172.17.0.1

# Use this in ProviderConfig instead of host.docker.internal
```

### Solution 3: Docker network
```bash
# Create a shared network
docker network create finedge-net

# Run LocalStack on it
docker run -d --name localstack --network finedge-net -p 4566:4566 localstack/localstack

# Create kind cluster on the same network
kind create cluster --name finedge --config kind/cluster-config.yaml
docker network connect finedge-net finedge-control-plane

# Use LocalStack's container name as hostname
# endpoint: "http://localstack:4566"
```

## Resource Requirements

kind runs Kubernetes inside Docker containers. For FinEdge (with Crossplane + ArgoCD + Kyverno + Prometheus):

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 6+ cores |
| Memory | 8 GB | 16 GB |
| Disk | 20 GB | 40 GB |

### If kind is slow or OOM-killing pods:
```bash
# Check Docker resource limits
docker system info | grep -i memory

# Increase Docker memory limit (Docker Desktop settings)
# Or for Linux, check cgroup limits
```

## Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| Cluster won't start | `ERROR: failed to create cluster` | Check Docker is running, increase memory |
| Pods stuck in `Pending` | No schedulable nodes | Check node status, resource quotas |
| Can't reach NodePort | Connection refused on host | Check `extraPortMappings` configuration |
| Image pull failures | `ErrImagePull` | Load images with `kind load docker-image` |
| DNS not resolving | Pods can't reach external URLs | Check CoreDNS pods in kube-system |

## Related

- [[LocalStack]]
- [[Crossplane]]
- [[Helm]]
