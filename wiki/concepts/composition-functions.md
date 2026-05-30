---
tags: [concept, crossplane, functions, pipeline]
created: 2026-05-30
---

# Composition Functions

**Composition Functions** are serverless functions that run inside a [[Compositions|Composition]]'s pipeline. In Crossplane v2, they're the ONLY way to define Composition logic — native `patches:` were removed. Every resource transformation, validation, and readiness check happens through functions.

## Why Functions?

In Crossplane v1, Compositions had a built-in `resources:` array with native patching. This was simple but limited:
- No conditionals (hard to skip resources based on flags)
- No loops (can't generate N resources)
- No external data (can't query APIs)
- No custom validation

Functions solve all of these. They're OCI containers that:
1. Receive the current desired state
2. Transform/add/remove resources
3. Return the updated desired state

## The Pipeline

A Composition pipeline is an ordered sequence of function calls:

```yaml
spec:
  mode: Pipeline
  pipeline:
    - step: patch-and-transform        # Step 1
      functionRef:
        name: function-patch-and-transform
      input:
        # ... resource definitions and patches
    - step: auto-ready                  # Step 2 (ALWAYS LAST)
      functionRef:
        name: function-auto-ready
```

Each step receives the output of the previous step. The final output becomes the desired managed resources.

## Functions Used in FinEdge

### 1. `function-patch-and-transform`

The workhorse. This does what native Composition resources used to do — define resources and patch fields from the composite to managed resources.

```yaml
- step: patch-and-transform
  functionRef:
    name: function-patch-and-transform
  input:
    apiVersion: pt.fn.crossplane.io/v1beta1    # Note: pt.fn, not pt.crossplane.io
    kind: Resources
    resources:
      - name: s3-bucket
        base:
          apiVersion: s3.aws.upbound.io/v1beta1
          kind: Bucket
          spec:
            deletionPolicy: Delete
            forProvider:
              region: us-east-1
        patches:
          - type: CombineFromComposite
            combine:
              variables:
                - fromFieldPath: spec.parameters.teamName
                - fromFieldPath: spec.parameters.environment
              strategy: string
              string:
                fmt: "finedge-%s-%s-data"
            toFieldPath: metadata.annotations[crossplane.io/external-name]
```

### 2. `function-auto-ready`

Aggregates readiness of all child [[Managed Resources]]. Without it, the composite resource (XR) would NEVER show `Ready: True` in v2.

```yaml
- step: auto-ready
  functionRef:
    name: function-auto-ready
  # No input needed — it just checks all child resources
```

**Must be the last pipeline step.** It looks at all resources created by previous steps and checks their `Ready` condition. If ALL are Ready, the XR becomes Ready.

## Available Functions

| Function | Purpose | FinEdge? |
|----------|---------|---------|
| `function-patch-and-transform` | Resource mapping and field patching | ✅ |
| `function-auto-ready` | Readiness aggregation | ✅ |
| `function-go-templating` | Go templates for complex logic | ❌ (future) |
| `function-kcl` | KCL language for type-safe compositions | ❌ (future) |
| `function-cel-filter` | CEL expressions for filtering | ❌ (future) |

### When to Use Which

```
Simple resource mapping with patches?
  → function-patch-and-transform (FinEdge's current approach)

Complex conditionals, loops, or computed values?
  → function-go-templating or function-kcl

Need to aggregate readiness?
  → function-auto-ready (always needed in v2)
```

## Function Installation

Functions are installed as Crossplane packages:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.7.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.3.0
```

Each function runs as a gRPC server pod in `crossplane-system`:
```bash
# Check function pods
kubectl get functions
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=function-patch-and-transform --tail=20
```

## Pipeline Data Flow

```
                    ┌─────────────────────┐
                    │  Composite Resource  │
                    │  (XAppEnv spec)      │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
Step 1:             │ function-patch-and-  │
                    │ transform            │
                    │                      │
                    │ Input:  XR spec      │
                    │ Output: Desired MRs  │
                    │  - Bucket            │
                    │  - Queue             │
                    │  - RDS Instance      │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
Step 2:             │ function-auto-ready  │
                    │                      │
                    │ Input:  Desired MRs  │
                    │ Output: Same MRs +   │
                    │  Ready condition on  │
                    │  the XR              │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │ Crossplane applies   │
                    │ desired MRs to       │
                    │ cluster              │
                    └─────────────────────┘
```

## The v1 → v2 Migration

| v1 | v2 |
|----|----|
| `mode: Resources` (default) | Removed — `mode: Pipeline` only |
| `resources:` with native `patches:` | Use `function-patch-and-transform` |
| Built-in readiness check | Use `function-auto-ready` |
| `patchSets:` | Define in function input |

If migrating from v1, the `function-patch-and-transform` input format is nearly identical to the old native format — just wrapped in the function's `input:` block.

## Testing Functions

```bash
# Render locally — runs the full pipeline without a cluster
crossplane render \
  infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml

# This shows exactly what resources the pipeline would create
# Verify: names, patches, conditional resources, tags
```

## Related

- [[Crossplane]]
- [[Compositions]]
- [[Managed Resources]]
- [[Composite Resource Definitions]]
