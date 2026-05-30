# security/ — Security Hardening

This directory contains security configurations applied **before** any team claims are submitted.

> **⚠️ Platform Team Only:** Only the platform team should modify these files.

## Structure

```
security/
├── rbac/
│   ├── platform-admin-role.yaml     # ClusterRole for platform engineers
│   ├── team-developer-role.yaml     # ClusterRole for team developers (AppEnv only)
│   └── team-rolebindings.yaml       # RoleBindings per team namespace
├── network-policies/
│   ├── default-deny.yaml            # Default deny all ingress/egress
│   └── allow-crossplane.yaml        # Allow crossplane-system outbound
└── resource-quotas/
    └── team-quotas.yaml             # Max 5 AppEnv claims per namespace + LimitRange
```

## RBAC Model

| Role | Scope | Can Do |
|------|-------|--------|
| `finedge-platform-admin` | Cluster | Manage XRDs, Compositions, Providers, Policies |
| `finedge-team-developer` | Namespace | CRUD on `AppEnv` claims only |

Team developers **cannot**:
- See other teams' claims
- Access Compositions, XRDs, or ProviderConfigs
- Create resources in other namespaces
- Bypass Kyverno policies

## Network Isolation

- **Default deny** all ingress/egress in team namespaces
- **Explicit allow** for `crossplane-system` to reach cloud APIs and K8s API server
- Teams are isolated from each other at the network level

## Resource Quotas

- **5 AppEnv claims maximum** per team namespace
- **Container defaults:** 100m CPU request, 128Mi memory request
- **Container limits:** 500m CPU, 512Mi memory

## Applying

```bash
# Apply to all team namespaces
kubectl apply -f security/rbac/
for ns in payments analytics notifications; do
  kubectl apply -f security/network-policies/default-deny.yaml -n $ns
  kubectl apply -f security/resource-quotas/team-quotas.yaml -n $ns
done
kubectl apply -f security/network-policies/allow-crossplane.yaml
```
