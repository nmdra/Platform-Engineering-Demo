---
tags: [concept, crossplane, deletion-policy, safety]
created: 2026-05-30
---

# Deletion Policy

**Deletion Policy** controls what happens to a cloud resource when its [[Managed Resources|Managed Resource]] is deleted from Kubernetes. It's one of the most critical safety mechanisms in [[Crossplane]] — getting it wrong can mean accidentally deleting a production database or leaking orphaned resources.

## The Two Policies

### `Delete` (Default in v2)
When the MR is deleted from Kubernetes, **also delete the cloud resource**.

```yaml
spec:
  deletionPolicy: Delete
```

Use when:
- Development/testing environments
- Resources backed by [[LocalStack]] (nothing real to lose)
- Stateless resources (S3 buckets with replaceable data)
- You want clean teardowns

### `Orphan`
When the MR is deleted from Kubernetes, **keep the cloud resource alive**.

```yaml
spec:
  deletionPolicy: Orphan
```

Use when:
- Production databases (RDS, DynamoDB)
- Resources with irreplaceable data
- Migrating from one Composition to another
- You'd rather have orphaned resources than data loss

## FinEdge Strategy

FinEdge uses an environment-based approach:

| Environment | Policy | Why |
|-------------|--------|-----|
| dev (LocalStack) | `Delete` | No real resources — clean teardown is preferred |
| staging | `Delete` | Staging data is reproducible |
| prod (RDS only) | `Orphan` | Database data is irreplaceable |
| prod (S3, SQS, SNS) | `Delete` | Stateless resources, can be recreated |

### In the Composition

```yaml
# LocalStack Composition — everything is Delete
resources:
  - name: s3-bucket
    base:
      spec:
        deletionPolicy: Delete

  - name: rds-instance
    base:
      spec:
        deletionPolicy: Delete    # Safe — it's just LocalStack

# AWS Composition — RDS is Orphan
resources:
  - name: rds-instance
    base:
      spec:
        deletionPolicy: Orphan    # NEVER delete prod databases

  - name: s3-bucket
    base:
      spec:
        deletionPolicy: Delete    # Stateless, can recreate
```

## The Danger Zone

### Scenario: Accidental Production Data Loss

```
1. Developer deletes their Claim (or ArgoCD prunes it)
2. Crossplane deletes the Composite Resource
3. Composition deletes all Managed Resources
4. Provider calls DeleteDBInstance
5. RDS instance is gone — data is gone
6. 😱
```

With `deletionPolicy: Orphan`:
```
1. Developer deletes their Claim
2. Crossplane deletes the Composite Resource
3. Composition deletes all Managed Resources from Kubernetes
4. Provider sees Orphan policy — does NOT call DeleteDBInstance
5. RDS instance still exists in AWS (just unmanaged now)
6. Platform team can re-adopt it later ✅
```

## Interaction with ArgoCD

This is why FinEdge uses `prune: false` for team [[ArgoCD]] applications:

```yaml
# ArgoCD Application for team apps
spec:
  syncPolicy:
    automated:
      prune: false      # ← Don't delete resources removed from Git
      selfHeal: true     # ← Fix drift
```

Even with `deletionPolicy: Delete`, ArgoCD won't remove the Claim if someone deletes the file from Git. Belt and suspenders.

See the [[App of Apps Pattern]] for how this is structured.

## Additional Protection: `deletionProtection`

For RDS instances, add AWS-level deletion protection:

```yaml
resources:
  - name: rds-instance
    base:
      spec:
        deletionPolicy: Orphan
        forProvider:
          deletionProtection: true    # AWS-level protection
```

With this, even if someone bypasses Crossplane and tries to delete the RDS instance via the AWS console or CLI, AWS will reject the request.

### Triple Protection Stack

```
Layer 1: ArgoCD prune: false     → Don't delete Claims from Git changes
Layer 2: deletionPolicy: Orphan  → Don't delete cloud resource on MR deletion
Layer 3: deletionProtection: true → Don't delete resource even if asked directly
```

## Always Be Explicit

**Never rely on defaults** for deletion policy. Always set it explicitly:

```yaml
# ✅ Good — explicit
spec:
  deletionPolicy: Delete

# ❌ Bad — relying on default (which may change between versions)
spec:
  # deletionPolicy not set
```

The default in Crossplane v2 is `Delete`, but relying on defaults is fragile — a version change or migration could change behavior silently.

## Related

- [[Crossplane]]
- [[Managed Resources]]
- [[Compositions]]
- [[ArgoCD]]
- [[App of Apps Pattern]]
