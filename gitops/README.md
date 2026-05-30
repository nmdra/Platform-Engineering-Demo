# gitops/ — ArgoCD Application Definitions

This directory contains **ArgoCD** Application manifests that enable GitOps — the cluster is continuously reconciled to match the state in Git.

## Structure

```
gitops/
├── argocd-apps.yaml      # Root app-of-apps (manages child Applications)
└── apps/
    ├── payments.yaml      # Points ArgoCD at infra/payments/
    ├── analytics.yaml     # Points ArgoCD at infra/analytics/
    └── notifications.yaml # Points ArgoCD at infra/notifications/
```

## How It Works

```
Git repo ──► ArgoCD (root app) ──► ArgoCD (team apps) ──► kubectl apply (claims)
                                                              │
                                                              ▼
                                                     Crossplane reconciles
                                                     AWS resources in LocalStack
```

1. **Root Application** watches `gitops/apps/` for child Application manifests
2. **Team Applications** each watch their `infra/<team>/` directory for claims
3. When a PR merges, ArgoCD detects the change within ~3 minutes
4. ArgoCD applies the updated claim → Crossplane reconciles the resources

## Safety Settings

| Setting | Root App | Team Apps | Why |
|---------|----------|-----------|-----|
| `prune` | `true` | **`false`** | Removing a team's Application is intentional; removing a claim.yaml should NOT auto-delete infrastructure |
| `selfHeal` | `true` | `true` | Drift correction — if someone manually edits a claim, ArgoCD reverts it |
| `CreateNamespace` | `true` | — | Auto-create namespaces if they don't exist |

## Applying

```bash
kubectl apply -f gitops/argocd-apps.yaml

# Watch sync status
kubectl get applications -n argocd
```
