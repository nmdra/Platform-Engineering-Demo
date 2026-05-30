# kind/ — Cluster Configuration

This directory contains the **kind** (Kubernetes IN Docker) cluster configuration for the FinEdge platform.

## Files

| File | Description |
|------|-------------|
| `cluster.yaml` | Multi-node kind cluster spec (1 control-plane + 2 workers) |

## Usage

```bash
# Create the cluster
kind create cluster --config kind/cluster.yaml

# Delete the cluster
kind delete cluster --name finedge-platform

# Check nodes
kubectl get nodes
```

## Configuration Details

- **Cluster name:** `finedge-platform`
- **Nodes:** 1 control-plane + 2 workers
- **Worker labels:** `workload: platform` and `workload: apps`
- **Port mappings:** 8080→80, 8443→443 on control-plane
- **Pod subnet:** `10.244.0.0/16`
- **Service subnet:** `10.96.0.0/12`

## Linux Note

On Linux, `host.docker.internal` isn't automatically available. After creating the cluster, patch CoreDNS — see [Docs/initial_plan.md](../Docs/initial_plan.md) Phase 1.6.
