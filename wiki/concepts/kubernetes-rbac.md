---
tags: [concept, kubernetes, security, rbac]
created: 2026-05-30
---

# Kubernetes RBAC

**Role-Based Access Control (RBAC)** is Kubernetes' authorization mechanism. It answers: "Can this user/service do this action on this resource?" In FinEdge, RBAC is how we ensure the payments team can't accidentally (or intentionally) mess with the analytics team's infrastructure.

## The Four RBAC Resources

### 1. Role (Namespaced)
Defines what actions are allowed within a **specific namespace**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: finedge-team-developer
  namespace: payments
rules:
  - apiGroups: ["platform.finedge.io"]
    resources: ["appenvs"]           # Claims
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
```

### 2. ClusterRole (Cluster-Wide)
Defines permissions that apply across ALL namespaces or for cluster-scoped resources:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: finedge-platform-admin
rules:
  - apiGroups: ["platform.finedge.io"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["apiextensions.crossplane.io"]
    resources: ["compositions", "compositeresourcedefinitions"]
    verbs: ["*"]
```

### 3. RoleBinding (Namespaced)
Binds a Role (or ClusterRole) to a subject within a namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-team-binding
  namespace: payments
subjects:
  - kind: Group
    name: payments-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: finedge-team-developer
  apiGroup: rbac.authorization.k8s.io
```

### 4. ClusterRoleBinding (Cluster-Wide)
Binds a ClusterRole to a subject across the entire cluster:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-admin-binding
subjects:
  - kind: Group
    name: platform-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: finedge-platform-admin
  apiGroup: rbac.authorization.k8s.io
```

## Subjects: Who Gets Access

| Subject Kind | Description | Example |
|-------------|-------------|---------|
| **User** | A human identity (usually from an identity provider) | `dev-lead@finedge.io` |
| **Group** | A collection of users | `payments-team`, `platform-admins` |
| **ServiceAccount** | A pod identity (machine user) | `system:serviceaccount:crossplane-system:crossplane` |

## The FinEdge RBAC Model

```
┌─────────────────────────────────────────────────────────┐
│                    Cluster Level                         │
│                                                         │
│  ClusterRole: finedge-platform-admin                    │
│    → Full access to XRDs, Compositions, Providers       │
│    → Bound to: platform-admins group                    │
│                                                         │
├────────────┬────────────┬──────────────────────────────┤
│ payments   │ analytics  │ notifications                 │
│ namespace  │ namespace  │ namespace                     │
│            │            │                               │
│ Role:      │ Role:      │ Role:                         │
│ team-dev   │ team-dev   │ team-dev                      │
│ → AppEnv   │ → AppEnv   │ → AppEnv                      │
│ → Secrets  │ → Secrets  │ → Secrets                     │
│ → Pods(RO) │ → Pods(RO) │ → Pods(RO)                   │
│            │            │                               │
│ Bound to:  │ Bound to:  │ Bound to:                     │
│ payments-  │ analytics- │ notifications-                │
│ team       │ team       │ team                          │
└────────────┴────────────┴──────────────────────────────┘
```

Key design decisions:
1. **No `cluster-admin` for developers** — only platform team gets cluster-wide access
2. **Namespace-scoped Roles** — teams can only access their own namespace
3. **Read-only for pods/services** — teams can observe but not manually delete workloads
4. **Full CRUD on Claims** — teams manage their own `AppEnv` resources
5. **Read-only for Secrets** — teams can read connection details but not create arbitrary secrets

## RBAC Verbs

| Verb | Meaning | Example |
|------|---------|---------|
| `get` | Read a single resource | `kubectl get appenv payments-dev` |
| `list` | Read all resources | `kubectl get appenvs` |
| `watch` | Stream changes | `kubectl get appenvs -w` |
| `create` | Create new resource | `kubectl apply -f claim.yaml` (first time) |
| `update` | Modify existing resource | `kubectl apply -f claim.yaml` (subsequent) |
| `patch` | Partial modification | `kubectl patch appenv ...` |
| `delete` | Remove resource | `kubectl delete appenv payments-dev` |

## Testing RBAC

```bash
# Can I create an AppEnv in the payments namespace?
kubectl auth can-i create appenvs.platform.finedge.io \
  -n payments --as=user@finedge.io

# Can I list secrets in analytics? (should be no for payments-team)
kubectl auth can-i list secrets \
  -n analytics --as-group=payments-team

# What can I do in this namespace?
kubectl auth can-i --list -n payments --as=user@finedge.io
```

## Common Mistakes

| Mistake | Why It's Dangerous |
|---------|-------------------|
| Using `cluster-admin` for everything | No isolation — a dev could delete Crossplane itself |
| Forgetting `watch` verb | Breaks `kubectl get -w` and controller patterns |
| Binding ClusterRole via RoleBinding | Limits ClusterRole to one namespace (sometimes intentional, sometimes a bug) |
| Not testing with `kubectl auth can-i` | You don't know if your RBAC works until you test it |

## Related

- [[Namespace Isolation]]
- [[Crossplane]]
- [[Pod Security Admission]]
- [[Policy as Code]]
- [[Network Policies]]
