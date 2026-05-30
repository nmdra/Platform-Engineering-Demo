---
tags: [concept, crossplane, core]
created: 2026-05-30
---

# Crossplane

**Crossplane** is an open-source Kubernetes add-on that turns your cluster into a universal control plane. Instead of using cloud consoles, CLIs, or Terraform, you define cloud infrastructure as Kubernetes resources — and Crossplane continuously reconciles them against reality.

## Why Crossplane?

The fundamental insight: **Kubernetes already has a powerful control plane** (API server, controllers, etcd, RBAC, admission webhooks). Crossplane reuses all of that for infrastructure management.

| Traditional IaC | Crossplane |
|-----------------|-----------|
| External state file (S3, Terraform Cloud) | Kubernetes etcd IS the state store |
| Run `apply` manually or in CI | Continuous [[Reconciliation Loop]] (~30s) |
| Separate auth system (IAM, API keys) | [[Kubernetes RBAC]] — same as your apps |
| Separate CLI (`terraform`, `pulumi`) | `kubectl` — you already know it |
| No self-healing | Automatic [[Drift Detection]] and repair |
| Flat resource model | [[Compositions]] for multi-resource abstractions |

## Core Architecture

```
┌─────────────────────────────────────────────────┐
│              Kubernetes Cluster                  │
│                                                  │
│  ┌─────────────┐  ┌───────────────────────────┐ │
│  │ Crossplane   │  │ Provider Pods              │ │
│  │ Core         │  │  ├─ provider-aws-s3        │ │
│  │              │  │  ├─ provider-aws-sqs       │ │
│  │ Reconciles:  │  │  ├─ provider-aws-rds       │ │
│  │ XRDs, Comps, │  │  └─ provider-aws-sns       │ │
│  │ XRs, Claims  │  │                            │ │
│  └──────┬───────┘  │ Reconciles:                │ │
│         │          │ Managed Resources           │ │
│         │          │ (Buckets, Queues, etc.)     │ │
│         │          └────────────┬────────────────┘ │
│         │                      │                   │
│  ┌──────┴──────────────────────┴──────────────┐   │
│  │              etcd                           │   │
│  │  Stores: XRDs, Compositions, Claims,        │   │
│  │          Managed Resources, status          │   │
│  └─────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────┘
                               │ API calls
                       ┌───────┴───────┐
                       │  Cloud APIs    │
                       │  (LocalStack   │
                       │   or AWS)      │
                       └───────────────┘
```

## Key Concepts

### [[Composite Resource Definitions]] (XRDs)
The API schema — defines what parameters teams can specify. Think of it as the "form" teams fill out.

### [[Compositions]]
The implementation logic — maps a Claim to multiple cloud resources. Written by the platform team.

### [[Claims]] (XRCs)
Namespaced requests — what teams actually submit. A 20-line YAML that triggers infrastructure creation.

### [[Managed Resources]] (MRs)
The 1:1 representation of a cloud resource in Kubernetes (e.g., a `Bucket` CR = an S3 bucket).

### [[Providers]]
Plugins that know how to talk to specific cloud APIs (AWS, GCP, Azure, etc.).

### [[Composition Functions]]
Serverless functions that run in the Composition pipeline to transform, validate, or generate resources.

## Crossplane v2 Changes

FinEdge uses Crossplane **v2.3.1**. Key changes from v1:

| Feature | v1 | v2 |
|---------|----|----|
| Composition mode | `mode: Resources` (default) or `mode: Pipeline` | **Pipeline only** — `mode: Resources` removed |
| Patch & Transform | Native `patches:` in Composition spec | Via `function-patch-and-transform` only |
| Usages | Alpha (behind flag) | **GA** — no flag needed |
| Namespaced resources | Limited | Full support (MRs can be namespaced) |
| `function-auto-ready` | Recommended | **Required** (no built-in readiness) |

### What This Means in Practice

Every Composition MUST use pipeline mode:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: appenv-localstack
spec:
  compositeTypeRef:
    apiVersion: platform.finedge.io/v1alpha1
    kind: XAppEnv
  mode: Pipeline    # ← Only option in v2
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          # ... resource definitions
    - step: auto-ready
      functionRef:
        name: function-auto-ready   # ← Required! No built-in readiness
```

## The Reconciliation Model

Crossplane follows the same [[Reconciliation Loop]] pattern as Kubernetes itself:

```
Every ~30 seconds:
  1. READ desired state (from etcd — the Claim/XR/MR specs)
  2. OBSERVE actual state (call cloud API — does the resource exist? what's its config?)
  3. COMPARE desired vs actual
  4. ACT if different:
     - Resource missing → CREATE
     - Resource config differs → UPDATE
     - Resource shouldn't exist → DELETE (if deletionPolicy: Delete)
  5. UPDATE status in Kubernetes (Ready: True/False, Synced: True/False)
```

This is fundamentally different from Terraform's "run once" model. See [[Drift Detection]] for why this matters.

## FinEdge Crossplane Setup

### Providers Installed
```yaml
# provider-family-aws (umbrella)
# Individual providers:
- provider-aws-s3       # S3 buckets, encryption, versioning
- provider-aws-sqs      # SQS queues (main + DLQ)
- provider-aws-rds      # RDS PostgreSQL instances
- provider-aws-sns      # SNS topics
```

### Functions Installed
```yaml
- function-patch-and-transform  # Resource mapping & patching
- function-auto-ready           # Readiness aggregation
```

### ProviderConfig
```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: localstack
spec:
  endpoint:
    url:
      type: Static
      static: "http://host.docker.internal:4566"  # LocalStack
  credentials:
    source: Secret
    secretRef:
      name: localstack-creds
      namespace: crossplane-system
      key: credentials
```

## Useful Debugging Commands

```bash
# See all Claims across namespaces
kubectl get appenv -A

# See all composite resources (cluster-scoped)
kubectl get xappenv

# See all managed resources
kubectl get managed

# Trace a Claim's full resource tree
kubectl crossplane beta trace appenv payments-dev -n payments

# Check provider pod logs
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=provider-aws-s3 --tail=40

# Render a Composition locally (no cluster needed)
crossplane render infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml
```

## Related

- [[Compositions]]
- [[Composite Resource Definitions]]
- [[Claims]]
- [[Managed Resources]]
- [[Providers]]
- [[Composition Functions]]
- [[Reconciliation Loop]]
- [[Drift Detection]]
- [[Infrastructure as Code]]
