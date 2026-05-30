---
tags: [tool, argocd, gitops, continuous-delivery]
created: 2026-05-30
---

# ArgoCD

**ArgoCD** is a declarative, GitOps continuous delivery tool for Kubernetes. In FinEdge, it ensures that the cluster state always matches what's declared in the Git repository — syncing [[Claims]], [[Compositions]], policies, and everything else automatically.

## How It Works

ArgoCD runs inside the Kubernetes cluster and watches a Git repository. When it detects a difference between Git and the cluster, it synchronizes:

```
Git Repository                    ArgoCD                         Cluster
┌──────────────┐                 ┌──────────────┐              ┌──────────────┐
│ infra/        │  poll (3 min)  │ Application   │  sync        │ AppEnv       │
│ policies/     │◀──────────────│ Controller    │─────────────▶│ Policies     │
│ crossplane/   │                │              │              │ Compositions │
│ gitops/       │  detect diff   │ Repo Server  │  apply       │              │
└──────────────┘                 └──────────────┘              └──────────────┘
```

## Core Concepts

### Application CRD

An ArgoCD `Application` defines:
- **Source**: Where to find the manifests (Git repo + path)
- **Destination**: Where to deploy them (cluster + namespace)
- **Sync Policy**: How to handle differences

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-infra
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/finedge/platform-engineering-demo
    targetRevision: main
    path: infra/payments          # Only sync this directory
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    automated:
      prune: false                # Don't delete resources removed from Git
      selfHeal: true              # Fix manual changes
    syncOptions:
      - CreateNamespace=false     # Namespace must already exist
```

### Sync Policies

| Setting | Value | Effect |
|---------|-------|--------|
| `automated` | enabled | Auto-sync on Git changes |
| `prune` | `true` | Delete resources removed from Git |
| `prune` | `false` | Keep resources even if removed from Git (safe!) |
| `selfHeal` | `true` | Revert manual cluster changes to match Git |
| `selfHeal` | `false` | Allow manual overrides (not recommended) |

### FinEdge Sync Strategy

```yaml
# Root app-of-apps: prune: true
# → If a team Application manifest is removed from Git, remove the ArgoCD Application

# Team apps: prune: false
# → If a Claim file is removed from Git, DON'T delete the Claim
# → This prevents accidental infrastructure deletion
# → Combined with Crossplane's deletionPolicy for safety
```

See [[Deletion Policy]] and [[App of Apps Pattern]] for why this matters.

## The App-of-Apps Pattern

FinEdge uses the [[App of Apps Pattern]] to manage multiple Applications:

```
Root Application (gitops/root-app.yaml)
    │
    ├── watches: gitops/apps/ directory
    │
    ├── creates: Application for payments    (→ infra/payments/)
    ├── creates: Application for analytics   (→ infra/analytics/)
    ├── creates: Application for notifications (→ infra/notifications/)
    ├── creates: Application for crossplane  (→ crossplane/)
    ├── creates: Application for policies    (→ policies/)
    └── creates: Application for security    (→ security/)
```

One root Application manages all child Applications. Add a new team? Just add a new Application manifest to `gitops/apps/`.

## Sync Waves

ArgoCD supports **sync waves** to control the order of resource deployment:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"    # Deploy first (lower = earlier)
```

FinEdge order:
```
Wave 0: Namespaces, RBAC, NetworkPolicies
Wave 1: Crossplane XRDs, Functions, Providers
Wave 2: Compositions, ProviderConfigs
Wave 3: Team Claims
Wave 4: Kyverno Policies
```

## Health Checks

ArgoCD knows how to check the health of standard Kubernetes resources. For Crossplane resources, it uses custom health checks:

```
AppEnv Claim:
  Healthy:   conditions.Ready = True AND conditions.Synced = True
  Degraded:  conditions.Ready = False
  Progressing: conditions.Synced = False
```

## Useful Commands

```bash
# Login to ArgoCD CLI
argocd login localhost:8080

# List applications
argocd app list

# Sync an application manually
argocd app sync payments-infra

# Check app status
argocd app get payments-infra

# View app diff (what would change)
argocd app diff payments-infra

# Hard refresh (force re-read from Git)
argocd app get payments-infra --hard-refresh

# Access the UI (port-forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
```

## Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| App stuck in `OutOfSync` | Resources don't match Git | `argocd app sync <app>` or check for schema changes |
| App `Degraded` | Health check failing | Check underlying resource status (`kubectl describe`) |
| `ComparisonError` | Can't parse manifests | YAML syntax error in Git |
| `InvalidSpecError` | Bad Application spec | Check source path and destination |

## Related

- [[GitOps]]
- [[Reconciliation Loop]]
- [[Drift Detection]]
- [[App of Apps Pattern]]
- [[Deletion Policy]]
