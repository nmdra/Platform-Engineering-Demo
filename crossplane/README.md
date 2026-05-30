# crossplane/ — Platform API Definitions

This directory contains the **Crossplane v2.3** configuration — the heart of the platform.

> **⚠️ Platform Team Only:** Only the platform team should modify files in this directory. Product teams interact via claims in `infra/`.

## Files

| File | Description |
|------|-------------|
| `functions.yaml` | Composition Functions (`function-patch-and-transform` + `function-auto-ready`) |
| `providers.yaml` | AWS provider family packages (S3, SQS, RDS, IAM, SNS) |
| `providerconfig-localstack.yaml` | Routes AWS API calls to LocalStack (`host.docker.internal:4566`) |
| `providerconfig-aws.yaml` | Routes AWS API calls to real AWS (Phase 8 — graduation) |
| `xrd-appenv.yaml` | `CompositeResourceDefinition` — the API schema teams use |
| `composition-appenv.yaml` | Composition for LocalStack — maps claims to AWS resources |
| `composition-appenv-aws.yaml` | Composition for real AWS — production-grade settings |

## Apply Order

```bash
# 1. Functions first (compositions depend on them)
kubectl apply -f crossplane/functions.yaml

# 2. Providers (need time to pull images)
kubectl apply -f crossplane/providers.yaml
kubectl get providers -w  # wait for HEALTHY=True

# 3. Provider config (needs providers to be healthy)
kubectl apply -f crossplane/providerconfig-localstack.yaml

# 4. XRD (defines the API)
kubectl apply -f crossplane/xrd-appenv.yaml

# 5. Composition (implements the API)
kubectl apply -f crossplane/composition-appenv.yaml
```

## Key Design Decisions

- **Pipeline mode only** — `mode: Resources` is removed in Crossplane v2
- **`function-auto-ready`** aggregates readiness from all child resources
- **`deletionPolicy: Delete`** on all resources (safe for LocalStack)
- **`deletionPolicy: Orphan`** on RDS in the production composition (prevents data loss)
- **`defaultCompositionRef`** set in XRD so teams don't need to specify it
