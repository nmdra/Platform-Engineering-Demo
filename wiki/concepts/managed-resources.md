---
tags: [concept, crossplane, managed-resources]
created: 2026-05-30
---

# Managed Resources

**Managed Resources (MRs)** are [[Crossplane]]'s 1:1 representation of external cloud resources as Kubernetes objects. Every S3 bucket, SQS queue, or RDS instance that Crossplane manages has a corresponding MR in the cluster. They're the lowest-level Crossplane abstraction — the bridge between Kubernetes and the cloud.

## What Is a Managed Resource?

A Managed Resource is a Kubernetes Custom Resource that maps directly to a single cloud resource:

| MR Kind (Kubernetes) | Cloud Resource | API Group |
|---------------------|----------------|-----------|
| `Bucket` | AWS S3 Bucket | `s3.aws.upbound.io` |
| `Queue` | AWS SQS Queue | `sqs.aws.upbound.io` |
| `Instance` | AWS RDS Instance | `rds.aws.upbound.io` |
| `Topic` | AWS SNS Topic | `sns.aws.upbound.io` |
| `BucketVersioning` | S3 Bucket Versioning Config | `s3.aws.upbound.io` |
| `ServerSideEncryptionConfiguration` | S3 Bucket Encryption Config | `s3.aws.upbound.io` |

## Anatomy of a Managed Resource

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: finedge-payments-dev-data        # Kubernetes name
  annotations:
    crossplane.io/external-name: finedge-payments-dev-data  # Cloud resource name
  labels:
    crossplane.io/claim-name: payments-dev
    crossplane.io/claim-namespace: payments
spec:
  deletionPolicy: Delete                 # What happens when MR is deleted
  forProvider:
    region: us-east-1
    tags:
      ManagedBy: crossplane
      Platform: finedge
      Team: payments
      Environment: dev
  providerConfigRef:
    name: localstack                     # Which ProviderConfig to use
status:
  conditions:
    - type: Ready
      status: "True"
      reason: Available
    - type: Synced
      status: "True"
      reason: ReconcileSuccess
  atProvider:
    arn: "arn:aws:s3:::finedge-payments-dev-data"
    id: finedge-payments-dev-data
```

## Key Fields

### `forProvider`
The desired state specification. This is provider-specific — it maps to the cloud API's parameters:
```yaml
forProvider:
  region: us-east-1
  tags:
    ManagedBy: crossplane
```

### `providerConfigRef`
Which [[Providers|ProviderConfig]] to use for API calls:
```yaml
providerConfigRef:
  name: localstack      # Routes to LocalStack (dev)
  # OR
  name: aws-real        # Routes to real AWS (prod)
```

### `deletionPolicy`
See [[Deletion Policy]] for details:
```yaml
deletionPolicy: Delete   # Delete cloud resource when MR is deleted
# OR
deletionPolicy: Orphan   # Keep cloud resource, only delete MR
```

### `crossplane.io/external-name`
The name of the resource in the cloud provider. This annotation is critical — it's how Crossplane finds the existing resource:
```yaml
annotations:
  crossplane.io/external-name: finedge-payments-dev-data
```

## Status Conditions

Every MR has two key status conditions:

| Condition | Meaning |
|-----------|---------|
| **Ready** | The cloud resource exists and is operational |
| **Synced** | The desired spec matches the actual cloud state |

### Status Combinations

| Ready | Synced | What It Means |
|-------|--------|---------------|
| True | True | ✅ Everything is good |
| False | True | ⏳ Resource is being created or is unhealthy |
| True | False | ⚠️ Drift detected — Crossplane will fix it |
| False | False | ❌ Something is wrong — check provider logs |

### `atProvider`
Read-only fields reflecting the actual cloud state:
```yaml
status:
  atProvider:
    arn: "arn:aws:s3:::finedge-payments-dev-data"
    id: finedge-payments-dev-data
    hostedZoneId: "Z3AQBSTGFYJSTF"
```

## How MRs Are Created

Teams never create MRs directly. The flow is:

```
1. Team creates Claim (AppEnv)
2. Crossplane creates Composite Resource (XAppEnv)
3. Composition's pipeline function creates Managed Resources
4. Provider controller sees new MR
5. Provider calls cloud API (e.g., CreateBucket)
6. Provider updates MR status (Ready: True)
7. function-auto-ready aggregates readiness → XR → Claim
```

## Observing Managed Resources

```bash
# List all managed resources in the cluster
kubectl get managed

# List specific types
kubectl get buckets.s3.aws.upbound.io
kubectl get queues.sqs.aws.upbound.io

# Describe a specific MR (see status, events, conditions)
kubectl describe bucket finedge-payments-dev-data

# Trace from Claim to MRs
kubectl crossplane beta trace appenv payments-dev -n payments
```

## The Reconciliation Loop

Every ~30 seconds, the [[Providers|Provider]] controller:

1. **Observes** the cloud resource (API call)
2. **Compares** with the MR spec (`forProvider`)
3. **Updates** the cloud resource if there's drift
4. **Updates** the MR status (`Ready`, `Synced`, `atProvider`)

This is the [[Reconciliation Loop]] in action — and it's what enables [[Drift Detection]].

## Common Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Provider not installed | MR stuck in `Synced: False` | Install the correct provider package |
| Wrong ProviderConfig | Auth errors in provider logs | Check `providerConfigRef.name` |
| External name conflict | `AlreadyExists` error | The cloud resource already exists with that name |
| Rate limiting | Intermittent sync failures | Reduce reconciliation frequency or add retries |

## Related

- [[Crossplane]]
- [[Providers]]
- [[Reconciliation Loop]]
- [[Drift Detection]]
- [[Deletion Policy]]
- [[Compositions]]
