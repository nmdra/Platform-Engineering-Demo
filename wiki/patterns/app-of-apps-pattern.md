---
tags: [pattern, argocd, gitops]
created: 2026-05-29
---

# App of Apps Pattern

The **App of Apps** pattern is an [[ArgoCD]] management strategy where a single "root" Application manages multiple child Application resources.

## The Problem

Without this pattern, you'd need to manually create and manage an ArgoCD Application for every team, environment, or service. This doesn't scale and defeats the purpose of [[GitOps]].

## How It Works

```
┌─────────────────────────────┐
│  finedge-infra-root         │ ◄── Root Application
│  (watches: gitops/apps/)    │     Created once, manages children
└────────┬────────────────────┘
         │ creates & manages
         ├──► payments-infra      (watches: infra/payments/)
         ├──► analytics-infra     (watches: infra/analytics/)
         └──► notifications-infra (watches: infra/notifications/)
```

1. **Root Application** points at a directory (`gitops/apps/`) containing child Application YAMLs
2. ArgoCD syncs the root app → discovers child Application manifests
3. ArgoCD creates each child Application → each watches its own `infra/<team>/` directory
4. Adding a new team = adding a new YAML file in `gitops/apps/`

## FinEdge Implementation

```yaml
# The root app — manages all children
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finedge-infra-root
  namespace: argocd
spec:
  source:
    path: gitops/apps       # ← directory of child Application YAMLs
  syncPolicy:
    automated:
      prune: true            # ← OK to prune child Applications (intentional)
      selfHeal: true
```

```yaml
# A child app — one per team
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-infra
spec:
  source:
    path: infra/payments     # ← directory of team's claims
  syncPolicy:
    automated:
      prune: false           # ← NEVER auto-prune infrastructure claims
      selfHeal: true
```

## Why `prune: false` on Children?

This is a critical safety decision. If someone removes `infra/payments/claim.yaml` from Git:

| Setting | What happens |
|---------|-------------|
| `prune: true` | ArgoCD deletes the claim → [[Crossplane]] deletes all AWS resources 💥 |
| `prune: false` | ArgoCD shows "OutOfSync" but doesn't delete anything ✅ |

With `prune: false`, deleting infrastructure requires an explicit action in the ArgoCD UI or CLI — providing a safety net.

## When to Use This Pattern

- ✅ Managing infrastructure for multiple teams
- ✅ Multi-environment setups (dev, staging, prod)
- ✅ Any scenario where you have many Applications following a consistent structure

## Alternatives

- **ApplicationSets** — ArgoCD's built-in generator for creating Applications from templates (Git directories, cluster lists, etc.). More powerful but more complex.
- **Manual Applications** — Fine for 1-3 apps, doesn't scale beyond that.

## Related

- [[ArgoCD]]
- [[GitOps]]
- [[Drift Detection]]
- [[Reconciliation Loop]]
