---
tags: [concept, crossplane, providers]
created: 2026-05-30
---

# Providers

**Providers** are [[Crossplane]] plugins that know how to talk to specific cloud APIs. They're the translation layer between Kubernetes Custom Resources ([[Managed Resources]]) and actual cloud services. Without a provider, Crossplane can't manage anything — it's just a framework.

## Provider Architecture

```
┌──────────────────────────────────────┐
│         Crossplane Core              │
│  (Manages XRDs, Compositions, XRs)   │
└─────────────┬────────────────────────┘
              │ creates Managed Resources
┌─────────────┴────────────────────────┐
│        Provider Controller Pod        │
│  provider-aws-s3                      │
│                                       │
│  Watches: Bucket, BucketVersioning,   │
│           BucketEncryption CRDs       │
│                                       │
│  Does: CREATE/UPDATE/DELETE/OBSERVE   │
│        via AWS S3 API                 │
└─────────────┬────────────────────────┘
              │ API calls
┌─────────────┴────────────────────────┐
│  Cloud API (LocalStack or AWS)       │
└──────────────────────────────────────┘
```

## Upbound vs. Community Providers

| Aspect | Upbound (Official) | Community |
|--------|-------------------|-----------|
| Maintainer | Upbound (the company behind Crossplane) | Community contributors |
| Package prefix | `xpkg.upbound.io/upbound/` | `xpkg.upbound.io/crossplane-contrib/` |
| Coverage | Individual resource types | Monolithic (all resources in one provider) |
| FinEdge uses | ✅ Upbound provider family | ❌ |

## Provider Families

Upbound organizes AWS providers into a **family** model:

```yaml
# The umbrella provider (installs shared CRDs and auth)
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-family-aws
spec:
  package: xpkg.upbound.io/upbound/provider-family-aws:v1.17.0

# Individual service providers (only install what you need)
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-sqs
spec:
  package: xpkg.upbound.io/upbound/provider-aws-sqs:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-rds
spec:
  package: xpkg.upbound.io/upbound/provider-aws-rds:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-sns
spec:
  package: xpkg.upbound.io/upbound/provider-aws-sns:v1.17.0
```

### Why Individual Providers?

The monolithic `provider-aws` installs **900+ CRDs**. Individual providers install only what you need:
- `provider-aws-s3` → ~10 CRDs (Bucket, BucketVersioning, etc.)
- `provider-aws-sqs` → ~3 CRDs (Queue, QueuePolicy, etc.)
- Less memory, faster startup, smaller attack surface

## ProviderConfig

A ProviderConfig tells a provider WHERE and HOW to connect:

### LocalStack Configuration
```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: localstack
spec:
  endpoint:
    url:
      type: Static
      static: "http://host.docker.internal:4566"
  credentials:
    source: Secret
    secretRef:
      name: localstack-creds
      namespace: crossplane-system
      key: credentials
```

### Real AWS Configuration
```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-real
spec:
  credentials:
    source: Secret
    secretRef:
      name: aws-creds
      namespace: crossplane-system
      key: credentials
```

### The Credentials Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: localstack-creds
  namespace: crossplane-system
type: Opaque
stringData:
  credentials: |
    [default]
    aws_access_key_id = test
    aws_secret_access_key = test
```

## How Providers Work

When a [[Managed Resources|Managed Resource]] is created:

1. Provider controller watches for its CRDs (e.g., `Bucket`)
2. New MR appears → provider reads `providerConfigRef`
3. Provider authenticates using the ProviderConfig's credentials
4. Provider calls the cloud API (`CreateBucket`, `DescribeBucketEncryption`, etc.)
5. Provider updates the MR status (`Ready: True`, `atProvider: { arn: ... }`)
6. Every ~30 seconds, provider re-observes and reconciles ([[Reconciliation Loop]])

## Version Pinning

**Always pin provider versions.** Unpinned versions can break compositions when a provider update changes field names or behaviors:

```yaml
# ✅ Good — pinned version
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.17.0

# ❌ Bad — no version or latest
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3
```

## Debugging Providers

```bash
# Check provider pod status
kubectl get providers

# Check if provider is healthy
kubectl get providers provider-aws-s3 -o yaml
# Look at status.conditions: Installed, Healthy

# View provider logs
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=provider-aws-s3 --tail=50

# Check ProviderConfig
kubectl get providerconfigs.aws.upbound.io
```

### Common Provider Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Wrong endpoint URL | `connection refused` in logs | Check `host.docker.internal` vs `localhost` |
| Missing credentials | `NoCredentialProviders` | Verify Secret exists in `crossplane-system` |
| Version mismatch | CRD field changes break compositions | Pin all provider versions |
| Pod crash loop | `CrashLoopBackOff` | Check logs, usually auth or memory issues |
| Free-tier restrictions | Some APIs return errors | [[LocalStack]] emulates most services |

## Related

- [[Crossplane]]
- [[Managed Resources]]
- [[LocalStack]]
- [[Reconciliation Loop]]
- [[Compositions]]
