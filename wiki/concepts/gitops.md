---
tags: [concept, gitops, operations]
created: 2026-05-30
---

# GitOps

**GitOps** is an operational model where Git is the single source of truth for your infrastructure and application state. Instead of running `kubectl apply` from your laptop, you push to Git — and a reconciliation tool ([[ArgoCD]]) automatically syncs the cluster to match.

## Core Principles

### 1. Declarative Desired State
Everything is described declaratively — Kubernetes manifests, [[Crossplane]] Claims, Kyverno policies. No imperative scripts.

### 2. Git as Single Source of Truth
The Git repository IS the desired state. What's in Git = what should be in the cluster.

### 3. Automated Reconciliation
A GitOps operator continuously compares Git ↔ cluster and applies differences. No manual `kubectl apply`.

### 4. Continuous Self-Healing
If someone manually changes something in the cluster, the operator detects the drift and reverts it.

## Pull-Based vs. Push-Based

| Model | How It Works | Example Tools | Security |
|-------|-------------|---------------|----------|
| **Push** | CI pushes changes to the cluster | Jenkins, GitHub Actions | CI needs cluster credentials |
| **Pull** | Agent IN the cluster pulls from Git | ArgoCD, Flux | Only the agent needs Git access |

FinEdge uses **pull-based GitOps** with [[ArgoCD]]:

```
Developer                 Git Repo              ArgoCD (in-cluster)          Kubernetes
   │                         │                        │                        │
   │── git push ──────────▶  │                        │                        │
   │                         │◀── poll (3 min) ──────│                        │
   │                         │── diff detected ──────▶│                        │
   │                         │                        │── kubectl apply ──────▶│
   │                         │                        │                        │
   │                         │                        │── health check ──────▶│
   │                         │                        │◀── status ────────────│
   │◀── Synced notification ─│                        │                        │
```

## The FinEdge GitOps Flow

### Infrastructure Changes (Crossplane)
```
1. Developer edits infra/payments/claim.yaml
2. Opens PR → code review
3. PR merged to main
4. ArgoCD detects change (polling interval: ~3 min)
5. ArgoCD applies the Claim to the cluster
6. Crossplane reconciles → provisions AWS resources
7. Resources Ready → Claim shows Ready: True
```

### Policy Changes (Kyverno)
```
1. Platform engineer edits policies/finedge-require-owner.yaml
2. Opens PR → review by another platform engineer
3. PR merged to main
4. ArgoCD applies the policy to the cluster
5. Kyverno starts enforcing the new policy immediately
```

### Platform Changes (Crossplane Compositions)
```
1. Platform engineer edits crossplane/composition-appenv.yaml
2. Opens PR → review
3. PR merged → ArgoCD applies
4. Existing Claims are re-reconciled with the updated Composition
5. New/changed resources are created/updated
```

## Why Git?

| Benefit | How |
|---------|-----|
| **Audit trail** | Git log shows who changed what, when, and why |
| **Rollback** | `git revert` to undo infrastructure changes |
| **Code review** | PRs ensure peer review of infrastructure changes |
| **Disaster recovery** | Cluster dies → reinstall ArgoCD → it rebuilds everything from Git |
| **Compliance** | Prove who authorized every infrastructure change |

## GitOps + Crossplane = Powerful Combo

Crossplane adds a second reconciliation loop ON TOP of ArgoCD's:

```
Layer 1 (ArgoCD):
  Git repo ──reconcile──▶ Kubernetes manifests (Claims, Compositions)

Layer 2 (Crossplane):
  Kubernetes manifests ──reconcile──▶ Cloud resources (S3, SQS, RDS)
```

ArgoCD ensures **Git → Cluster** consistency.
Crossplane ensures **Cluster → Cloud** consistency.

Together, they provide **Git → Cloud** consistency with two self-healing loops.

## ArgoCD Sync Policies in FinEdge

```yaml
# Root app-of-apps
syncPolicy:
  automated:
    prune: true       # Delete child Applications removed from Git
    selfHeal: true    # Fix manual changes to Application manifests

# Team infrastructure apps
syncPolicy:
  automated:
    prune: false      # DON'T delete Claims removed from Git (safety!)
    selfHeal: true    # Fix manual changes to Claims
```

Why `prune: false` for team apps? Because accidentally deleting a Claim file from Git would trigger [[Deletion Policy|deletion]] of cloud resources. Belt and suspenders — see [[ArgoCD]] for details.

## Anti-Patterns

| Anti-Pattern | Problem |
|-------------|---------|
| `kubectl apply` from laptops | Bypasses Git review, no audit trail |
| Secrets in Git | Security breach waiting to happen — use external-secrets or sealed-secrets |
| No branch protection | Anyone can push to main without review |
| Manual cluster changes | ArgoCD will revert them (if selfHeal is on) |
| Mixing imperative and declarative | Half the state in Git, half not — impossible to reason about |

## Related

- [[ArgoCD]]
- [[Reconciliation Loop]]
- [[Drift Detection]]
- [[Infrastructure as Code]]
- [[App of Apps Pattern]]
