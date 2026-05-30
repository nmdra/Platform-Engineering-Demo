---
tags: [concept, kubernetes, security, multi-tenancy]
created: 2026-05-30
---

# Namespace Isolation

**Namespace Isolation** is the strategy of using Kubernetes namespaces as security boundaries between teams. In FinEdge, each product team gets its own namespace with RBAC, NetworkPolicies, ResourceQuotas, and Pod Security Admission labels — creating a multi-tenant environment from a single cluster.

## Why Namespaces?

A Kubernetes namespace is a logical partition within a cluster. Think of it as a "virtual cluster" that:

- Scopes [[Kubernetes RBAC]] — a RoleBinding in `payments` doesn't apply to `analytics`
- Scopes [[Network Policies]] — rules apply within namespace boundaries
- Scopes ResourceQuotas — each team gets a CPU/memory budget
- Scopes [[Claims]] — an `AppEnv` in `payments` belongs to the payments team

## The FinEdge Namespace Model

```
Cluster
├── crossplane-system    # Crossplane core (platform team only)
├── argocd               # ArgoCD (platform team only)
├── kyverno              # Policy engine (platform team only)
├── monitoring           # Prometheus + Grafana (platform team only)
├── payments             # Payments team ← isolated
├── analytics            # Analytics team ← isolated
└── notifications        # Notifications team ← isolated
```

### Namespace Creation with Security Labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    # Pod Security Admission labels
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
    # FinEdge labels
    platform.finedge.io/team: payments
    platform.finedge.io/managed-by: platform-team
```

The [[Pod Security Admission]] labels are critical — they restrict what kinds of pods can run in the namespace.

## Defense in Depth: Four Layers

### Layer 1: RBAC (Who Can Do What)
See [[Kubernetes RBAC]] for details.
```yaml
# Payments team can ONLY access their namespace
kind: RoleBinding
metadata:
  namespace: payments     # Scoped to payments only
subjects:
  - kind: Group
    name: payments-team
```

### Layer 2: Network Policies (Who Can Talk to Whom)
See [[Network Policies]] for details.
```yaml
# Default deny all traffic in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}          # Applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
```
Then selectively allow what's needed (DNS, specific services).

### Layer 3: Resource Quotas (How Much Can They Use)
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
```

Without quotas, one team could consume all cluster resources (the "noisy neighbor" problem).

### Layer 4: LimitRanges (Defaults for Pods)
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: payments
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

LimitRanges set **default** resource requests/limits so pods without explicit values don't run unbounded.

## What Namespaces DON'T Isolate

Important: namespaces are NOT a security boundary on their own:

| What They Isolate | What They DON'T Isolate |
|-------------------|------------------------|
| RBAC scoping | Kernel-level isolation (use [[Pod Security Admission]]) |
| Resource quotas | Node-level attacks |
| Network policies | Shared storage without access control |
| Secret scoping | Cluster-scoped resources (CRDs, ClusterRoles) |

That's why FinEdge combines namespace isolation with RBAC, NetworkPolicies, PSA, and [[Policy as Code|Kyverno policies]] — defense in depth.

## Namespace Anti-Patterns

| Anti-Pattern | Problem | FinEdge Solution |
|-------------|---------|-----------------|
| Using `default` namespace | No isolation, hard to manage | [[Kyverno]] policy rejects resources in `default` |
| One namespace for everything | No team isolation | One namespace per team |
| No ResourceQuotas | Noisy neighbor problem | Quotas on every team namespace |
| No NetworkPolicies | Pods can reach anything in the cluster | Default-deny + explicit allow |
| Hardcoding namespace in manifests | Can't reuse across teams | Parametrize via [[Compositions]] patches |

## Related

- [[Kubernetes RBAC]]
- [[Network Policies]]
- [[Pod Security Admission]]
- [[Policy as Code]]
- [[Claims]]
