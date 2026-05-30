---
tags: [concept, crossplane, drift, self-healing]
created: 2026-05-30
---

# Drift Detection

**Drift** is when the actual state of a cloud resource diverges from its declared desired state. Someone logs into the AWS console and changes a tag, deletes a bucket, or modifies a security group — that's drift. [[Crossplane]]'s superpower is that it **detects and fixes drift automatically**.

## How Drift Happens

```
Desired state (in Kubernetes):           Actual state (in AWS):
┌──────────────────────┐                ┌──────────────────────┐
│ Bucket                │                │ S3 Bucket             │
│   encryption: AES256  │    ≠           │   encryption: none    │ ← someone disabled it
│   tags:               │                │   tags:               │
│     Team: payments    │                │     Team: payments    │
│     ManagedBy: xplane │                │     ManagedBy: (gone) │ ← someone deleted the tag
└──────────────────────┘                └──────────────────────┘
```

Common causes of drift:
- Manual console changes ("I'll just fix this real quick...")
- Automation outside of Crossplane (scripts, other tools)
- Cloud provider auto-updates (security patches, deprecations)
- Another team's tool modifying shared resources

## The Detection-Correction Cycle

Every ~30 seconds, the [[Reconciliation Loop]]:

```
1. OBSERVE: Provider calls DescribeBucket API
   → Returns: encryption=none, tags={Team: payments}

2. COMPARE: Crossplane compares with MR spec
   → encryption: AES256 ≠ none          → DRIFT
   → tag ManagedBy: crossplane ≠ (none) → DRIFT
   → tag Team: payments = payments      → OK

3. ACT: Provider calls PutBucketEncryption and PutBucketTagging
   → Restores encryption to AES256
   → Restores ManagedBy tag

4. STATUS: MR status updated
   → Synced: True (back in sync)
   → Ready: True (resource operational)
```

## Self-Healing Demo

One of the most powerful demonstrations in this project:

```bash
# 1. Check the bucket exists and is Ready
kubectl get buckets.s3.aws.upbound.io
# NAME                          READY   SYNCED
# finedge-payments-dev-data     True    True

# 2. Delete the bucket directly in LocalStack
aws --endpoint-url=http://localhost:4566 s3 rb s3://finedge-payments-dev-data --force

# 3. Wait ~30 seconds...

# 4. Check again — it's back!
kubectl get buckets.s3.aws.upbound.io
# NAME                          READY   SYNCED
# finedge-payments-dev-data     True    True

# The bucket was recreated automatically by the reconciliation loop
```

## Drift vs. Intended Changes

Not all differences are "drift." Sometimes you WANT to change the desired state:

| Scenario | Is It Drift? | Correct Approach |
|----------|-------------|-----------------|
| Someone changes a tag in AWS console | ✅ Yes | Crossplane auto-fixes |
| Platform team updates the Composition | ❌ No | Update in Git → ArgoCD syncs → Crossplane applies |
| Developer updates their Claim | ❌ No | Update in Git → ArgoCD syncs → Crossplane applies |
| Cloud provider deprecates an API | ⚠️ Maybe | May need Composition update |

The rule: **All intentional changes go through Git** ([[GitOps]]). Anything changed outside Git is drift and gets reverted.

## Crossplane vs. Terraform Drift Handling

| Aspect | Terraform | Crossplane |
|--------|-----------|-----------|
| Detection | `terraform plan` (manual) | Automatic (~30s) |
| Correction | `terraform apply` (manual) | Automatic |
| Continuous? | Only when you run it | Always running |
| State refresh | `terraform refresh` (can be expensive) | Built into reconciliation |
| Notification | Only if you run plan | MR status changes (Ready/Synced) |

## Monitoring Drift

### Status Conditions
```bash
# Synced: False usually means drift was detected
kubectl get managed -o wide | grep False
```

### Events
```bash
# See reconciliation events
kubectl describe bucket finedge-payments-dev-data
# Events:
#   Warning  CannotObserve  ...  bucket encryption differs from desired
#   Normal   ExternalUpdate ...  updating bucket encryption
```

### Prometheus Metrics
With [[Prometheus and Grafana]], you can alert on drift:
- `crossplane_managed_resource_exists` — is the resource present?
- `crossplane_managed_resource_ready` — is it in the desired state?

## When Self-Healing Isn't Enough

Some scenarios require manual intervention:

1. **Resource deleted AND recreate fails** (e.g., name conflict, IAM permissions)
2. **Provider bug** (incorrect API call)
3. **Cloud API breaking change** (field renamed)
4. **Rate limiting** (too many reconciliations = API throttling)

In these cases, check provider logs:
```bash
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=provider-aws-s3 --tail=100
```

## Related

- [[Reconciliation Loop]]
- [[Crossplane]]
- [[Managed Resources]]
- [[GitOps]]
- [[Prometheus and Grafana]]
