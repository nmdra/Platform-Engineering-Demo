---
tags: [concept, crossplane, xrd, api-design]
created: 2026-05-30
---

# Composite Resource Definitions

A **Composite Resource Definition (XRD)** is [[Crossplane]]'s way of defining a new API. It's the schema — the "form" that teams fill out when they submit a [[Claims|Claim]]. The XRD defines what parameters are available, what's required, what has defaults, and what the status looks like.

## XRD = Your Platform's API Contract

Think of an XRD like a database schema or a TypeScript interface:

```
XRD (Schema)          →  Claim (Instance)
──────────────            ──────────────
"teamName: string"        "teamName: payments"
"environment: enum"       "environment: dev"
"storage.enabled: bool"   "storage.enabled: true"
```

When you `kubectl apply` an XRD, Crossplane registers a new [[Custom Resource Definitions|CRD]] in the Kubernetes API. Teams can then discover it with `kubectl explain`.

## Anatomy of the FinEdge XRD

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xappenvs.platform.finedge.io    # Must match group + plural
spec:
  group: platform.finedge.io             # API group
  names:
    kind: XAppEnv                        # Composite Resource name (cluster-scoped)
    plural: xappenvs
  claimNames:
    kind: AppEnv                         # Claim name (namespaced!)
    plural: appenvs

  defaultCompositionRef:
    name: appenv-localstack              # Default Composition if none selected

  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  required:
                    - teamName
                    - environment
                  properties:
                    teamName:
                      type: string
                      description: "Team that owns this environment"
                    environment:
                      type: string
                      enum: [dev, staging, prod]
                      description: "Target environment"
                    storage:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                    queue:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: true
                    database:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: false
                        size:
                          type: string
                          enum: [small, medium, large]
                          default: small
                    notifications:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: false
```

## Key Fields Explained

### `group` and `names`
```yaml
group: platform.finedge.io
names:
  kind: XAppEnv        # The Composite Resource (XR) — cluster-scoped
  plural: xappenvs
claimNames:
  kind: AppEnv         # The Claim (XRC) — namespaced
  plural: appenvs
```

Two resources get created from one XRD:
- **XAppEnv** (cluster-scoped) — the "real" composite resource, managed by Crossplane
- **AppEnv** (namespaced) — the Claim that teams interact with

### `defaultCompositionRef`
```yaml
defaultCompositionRef:
  name: appenv-localstack
```
If a Claim doesn't specify `compositionSelector`, use this [[Compositions|Composition]].

### Schema Validation (openAPIV3Schema)
The schema enforces:
- **Required fields**: `teamName` and `environment` must be present
- **Enums**: `environment` must be `dev`, `staging`, or `prod`
- **Defaults**: `storage.enabled` defaults to `true`
- **Types**: Boolean flags, string names

```bash
# Teams can discover the schema!
kubectl explain appenv.spec.parameters
kubectl explain appenv.spec.parameters.database
```

## XRD → Claim → Composition Flow

```
1. Platform team applies XRD
   └─ Kubernetes registers AppEnv CRD
   └─ kubectl explain appenv now works

2. Team applies Claim (AppEnv)
   └─ Kubernetes validates against XRD schema
   └─ Crossplane creates XAppEnv (composite)
   └─ Crossplane matches to Composition (via labels or default)
   └─ Composition creates Managed Resources

3. Managed Resources reconcile
   └─ Provider calls cloud APIs
   └─ Status propagates: MR → XR → Claim
```

## Versioning

XRDs support multiple API versions for backward compatibility:

```yaml
versions:
  - name: v1alpha1
    served: true          # Accept requests for this version
    referenceable: true   # Compositions can reference this version
  - name: v1beta1
    served: true
    referenceable: true
```

When evolving the API:
1. Add a new version (e.g., `v1beta1`) with new fields
2. Keep the old version `served: true` for backward compatibility
3. Eventually deprecate old versions

## Design Guidelines

### DO
- Use **business-domain names** (`teamName`, not `s3BucketPrefix`)
- Set **defaults** for everything that has a reasonable default
- Use **enums** to constrain choices (`dev|staging|prod`)
- Mark **truly required** fields only
- Add **descriptions** to every field

### DON'T
- Expose cloud-specific details (`instanceClass`, `aclPolicy`)
- Make everything required (use defaults)
- Use deeply nested schemas (keep it flat where possible)
- Remove fields from existing versions (breaks backward compatibility)

## Testing XRD Changes

```bash
# Validate the XRD schema
kubectl apply --dry-run=server -f crossplane/xrd-appenv.yaml

# Check if a claim validates against the schema
kubectl apply --dry-run=server -f infra/payments/claim.yaml

# Render the full composition output
crossplane render infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml
```

## Related

- [[Crossplane]]
- [[Compositions]]
- [[Claims]]
- [[Custom Resource Definitions]]
- [[Managed Resources]]
