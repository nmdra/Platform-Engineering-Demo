---
tags: [concept, crossplane, compositions]
created: 2026-05-30
---

# Compositions

A **Composition** is the heart of [[Crossplane]]'s abstraction model. It's the platform team's implementation that maps a simple [[Claims|Claim]] to multiple [[Managed Resources]]. Think of it as a "recipe" — the Claim says *what* the team wants, the Composition says *how* to build it.

## The Big Picture

```
Claim (20 lines)          Composition (200+ lines)         Cloud Resources
┌────────────┐            ┌──────────────────────┐         ┌─────────────┐
│ teamName   │            │ Pipeline:            │         │ S3 Bucket   │
│ environment│──matches──▶│  patch-and-transform │──creates▶│ SQS Queue   │
│ storage:   │            │  auto-ready          │         │ SQS DLQ     │
│   enabled  │            │                      │         │ RDS Instance│
│ queue:     │            │ Resources:           │         │ SNS Topic   │
│   enabled  │            │  S3 + encryption     │         │ Encryption  │
└────────────┘            │  SQS + DLQ           │         │ Versioning  │
                          │  RDS + protection    │         └─────────────┘
                          │  SNS                 │
                          └──────────────────────┘
```

## Pipeline Mode (v2 Only)

In Crossplane v2, Compositions MUST use `mode: Pipeline`. The pipeline is a sequence of [[Composition Functions]] that process the Claim:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: appenv-localstack
  labels:
    provider: localstack              # Used for compositionSelector
spec:
  compositeTypeRef:
    apiVersion: platform.finedge.io/v1alpha1
    kind: XAppEnv                     # Links to the XRD
  mode: Pipeline
  pipeline:
    - step: patch-and-transform       # Step 1: Map claim → resources
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: s3-bucket
            # ... resource definition
          - name: sqs-main-queue
            # ... resource definition
    - step: auto-ready                # Step 2: Aggregate readiness
      functionRef:
        name: function-auto-ready     # ← ALWAYS last
```

### Step Ordering Matters

1. **patch-and-transform** — Creates/patches resources based on Claim fields
2. Any custom functions (validation, transformation)
3. **auto-ready** — MUST be last; it checks if all child resources are Ready

## Resource Definition Anatomy

Each resource in the Composition has three parts:

### 1. Base (The Template)
```yaml
resources:
  - name: s3-bucket
    base:
      apiVersion: s3.aws.upbound.io/v1beta1
      kind: Bucket
      spec:
        deletionPolicy: Delete         # ← Explicit [[Deletion Policy]]
        forProvider:
          region: us-east-1
          tags:
            ManagedBy: crossplane
            Platform: finedge
```

### 2. Patches (Dynamic Values)
```yaml
    patches:
      # Construct the bucket name: finedge-<team>-<env>-data
      - type: CombineFromComposite
        combine:
          variables:
            - fromFieldPath: spec.parameters.teamName
            - fromFieldPath: spec.parameters.environment
          strategy: string
          string:
            fmt: "finedge-%s-%s-data"
        toFieldPath: metadata.annotations[crossplane.io/external-name]
      
      # Copy team name into tags
      - type: FromCompositeFieldPath
        fromFieldPath: spec.parameters.teamName
        toFieldPath: spec.forProvider.tags.Team
      
      # Copy environment into tags
      - type: FromCompositeFieldPath
        fromFieldPath: spec.parameters.environment
        toFieldPath: spec.forProvider.tags.Environment
```

### 3. Conditionals (Optional Resources)
```yaml
      # Only create this resource if storage is enabled
      - type: FromCompositeFieldPath
        fromFieldPath: spec.parameters.storage.enabled
        toFieldPath: spec.skipIfTrue    # Skip if false
```

## Patch Types

| Patch Type | Direction | Use Case |
|-----------|-----------|----------|
| `FromCompositeFieldPath` | Claim → Resource | Copy teamName into tags |
| `ToCompositeFieldPath` | Resource → Claim | Expose bucket ARN in status |
| `CombineFromComposite` | Multiple Claim fields → Resource | Build `finedge-<team>-<env>-data` |
| `CombineToComposite` | Multiple Resource fields → Claim | Build connection string |

## Composition Selection

When a team creates a Claim, Crossplane needs to know WHICH Composition to use. FinEdge uses label-based selection:

```yaml
# In the Claim
spec:
  compositionSelector:
    matchLabels:
      provider: localstack    # Use the LocalStack composition

# Matches this Composition
metadata:
  labels:
    provider: localstack      # ← Label match
```

This lets you have multiple Compositions for the same XRD:
- `appenv-localstack` (dev, uses [[LocalStack]])
- `appenv-aws` (prod, uses real AWS)

## FinEdge Composition Resources

The FinEdge `appenv-localstack` Composition creates up to 7 resources:

| Resource | Condition | Naming Pattern |
|----------|-----------|----------------|
| S3 Bucket | `storage.enabled` | `finedge-<team>-<env>-data` |
| BucketVersioning | `storage.enabled` | (attached to bucket) |
| BucketEncryption | `storage.enabled` | (attached to bucket) |
| SQS DLQ | `queue.enabled` | `finedge-<team>-<env>-events-dlq` |
| SQS Main Queue | `queue.enabled` | `finedge-<team>-<env>-events` |
| RDS Instance | `database.enabled` | `finedge-<team>-<env>-db` |
| SNS Topic | `notifications.enabled` | `finedge-<team>-<env>-notify` |

## Testing Compositions

Always test locally before applying:

```bash
# Render the composition (no cluster needed!)
crossplane render \
  infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml

# Output shows exactly what resources would be created
# Check: names, tags, patches, resource count
```

## Design Principles

1. **Teams specify WHAT** (boolean flags, team name, environment)
2. **Platform encodes HOW** (encryption config, naming, DLQ wiring, tagging)
3. **One Composition per environment class** (LocalStack vs AWS)
4. **Explicit [[Deletion Policy]]** on every resource
5. **[[Composition Functions|function-auto-ready]]** always last

## Anti-Patterns

| Anti-Pattern | Why It's Bad |
|-------------|-------------|
| Hardcoded resource names | Can't have multiple claims (name collisions) |
| No `function-auto-ready` | XR shows Ready before children are Ready |
| Exposing cloud fields in [[Composite Resource Definitions|XRD]] | Defeats the abstraction |
| Missing `deletionPolicy` | Unknown behavior on claim deletion |
| Monolithic Composition for all envs | Can't have different policies per environment |

## Related

- [[Crossplane]]
- [[Composite Resource Definitions]]
- [[Claims]]
- [[Composition Functions]]
- [[Managed Resources]]
- [[Deletion Policy]]
- [[Composition Pattern]]
