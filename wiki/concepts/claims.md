---
tags: [concept, crossplane, claims, developer-experience]
created: 2026-05-30
---

# Claims

**Claims** (also called Composite Resource Claims or XRCs) are the developer-facing API of [[Crossplane]]. They're the simple, namespaced YAML resources that product teams submit to request infrastructure. A Claim is the answer to: "How do I get an S3 bucket without learning AWS?"

## Claims vs. Composite Resources

| Aspect | Claim (XRC) | Composite Resource (XR) |
|--------|-------------|------------------------|
| **Kind** | `AppEnv` | `XAppEnv` |
| **Scope** | Namespaced | Cluster-scoped |
| **Who creates it** | Product team | Crossplane (automatically) |
| **Visibility** | Team can see their own | Platform team sees all |
| **RBAC** | Scoped to namespace | Requires cluster-level access |

When a team creates a Claim, Crossplane automatically creates a corresponding Composite Resource (XR). The XR is what actually triggers the [[Compositions|Composition]].

```
Team creates:              Crossplane creates:          Composition creates:
┌──────────────┐           ┌──────────────┐            ┌─────────────────┐
│ AppEnv       │──────────▶│ XAppEnv      │───────────▶│ Managed         │
│ (namespaced) │           │ (cluster)    │            │ Resources       │
│ in: payments │           │              │            │ (S3, SQS, etc.) │
└──────────────┘           └──────────────┘            └─────────────────┘
```

## The FinEdge Claim

```yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: payments-dev
  namespace: payments                    # Team's namespace
  annotations:
    platform.finedge.io/owner: "dev-lead@finedge.io"    # Required by Kyverno
    platform.finedge.io/ticket: "INFRA-042"             # Required by Kyverno
spec:
  parameters:
    teamName: payments
    environment: dev
    storage:
      enabled: true       # → S3 Bucket (encrypted, versioned)
    queue:
      enabled: true        # → SQS Main Queue + DLQ
    database:
      enabled: false       # → No RDS instance
    notifications:
      enabled: true        # → SNS Topic
  compositionSelector:
    matchLabels:
      provider: localstack # → Use LocalStack Composition
```

That's it. **20 lines**. The team doesn't need to know about:
- S3 server-side encryption configuration
- SQS dead-letter queue policies and redrive configuration
- Bucket naming conventions
- Tagging standards
- [[Deletion Policy]] settings

All of that is in the [[Compositions|Composition]].

## Status Propagation

When a Claim is created, Crossplane updates its status as resources are provisioned:

```yaml
# kubectl get appenv payments-dev -n payments -o yaml
status:
  conditions:
    - type: Ready
      status: "True"           # All child resources are ready
      reason: Available
    - type: Synced
      status: "True"           # Desired state matches actual state
      reason: ReconcileSuccess
```

Status flows upward:
```
Managed Resource (Bucket) → Ready: True
Managed Resource (Queue)  → Ready: True
        ↓ aggregated by function-auto-ready
Composite Resource (XAppEnv) → Ready: True
        ↓ propagated
Claim (AppEnv) → Ready: True ✅
```

If ANY child resource isn't Ready, the Claim shows `Ready: False`.

## Connection Secrets

When a Composition creates resources with connection details (passwords, endpoints, ARNs), they flow back to the Claim's namespace as a Kubernetes Secret:

```yaml
# In the Claim
spec:
  writeConnectionSecretToRef:
    name: payments-dev-conn       # Secret name
    # namespace is automatically the Claim's namespace

# Resulting Secret in the payments namespace
apiVersion: v1
kind: Secret
metadata:
  name: payments-dev-conn
  namespace: payments
data:
  endpoint: aHR0cDovLy4uLg==     # Base64 encoded
  password: c3VwZXJzZWNyZXQ=
```

Teams can mount this secret in their pods — no manual credential passing.

## Lifecycle Operations

### Create
```bash
kubectl apply -f infra/payments/claim.yaml
# Crossplane creates XR → Composition creates MRs → Providers create cloud resources
```

### Update
```bash
# Edit claim.yaml: set queue.enabled: false
kubectl apply -f infra/payments/claim.yaml
# Crossplane deletes the SQS Queue MR → Provider deletes the cloud queue
```

### Delete
```bash
kubectl delete appenv payments-dev -n payments
# Crossplane deletes XR → MRs → cloud resources
# UNLESS deletionPolicy: Orphan (then cloud resources stay)
```

### Observe
```bash
# Quick status
kubectl get appenv -n payments
# NAME           SYNCED   READY   AGE
# payments-dev   True     True    5m

# Detailed trace
kubectl crossplane beta trace appenv payments-dev -n payments
# Shows the full tree: Claim → XR → all MRs with their statuses
```

## Policy Enforcement

Before a Claim is accepted, [[Kyverno]] policies validate it:

1. **Owner annotation required** — `platform.finedge.io/owner` must be present
2. **Ticket annotation required** — `platform.finedge.io/ticket` must be present
3. **Namespace validation** — Claims can only be created in team namespaces (not `default`)

```bash
# This will be REJECTED by Kyverno:
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: payments-dev
  namespace: payments
  # ❌ Missing owner and ticket annotations!
spec:
  parameters:
    teamName: payments
    environment: dev
```

## GitOps Flow

In the FinEdge model, Claims live in Git and are synced by [[ArgoCD]]:

```
Developer edits infra/payments/claim.yaml
    → git push
    → ArgoCD detects change
    → ArgoCD applies Claim to cluster
    → Crossplane reconciles
    → Infrastructure created/updated
```

This means infrastructure changes go through the same PR review process as code changes.

## Related

- [[Crossplane]]
- [[Composite Resource Definitions]]
- [[Compositions]]
- [[Managed Resources]]
- [[Deletion Policy]]
- [[ArgoCD]]
