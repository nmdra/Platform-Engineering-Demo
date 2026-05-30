---
tags: [concept, kubernetes, crd, api-extension]
created: 2026-05-30
---

# Custom Resource Definitions

**Custom Resource Definitions (CRDs)** are Kubernetes' mechanism for extending the API with your own resource types. Everything in [[Crossplane]] — XRDs, Compositions, Managed Resources, Providers — is built on top of CRDs. Understanding CRDs is essential to understanding how the platform works.

## What Is a CRD?

A CRD tells the Kubernetes API server: "I want a new resource type called X with this schema." After installing a CRD, you can use `kubectl` to create, read, update, and delete instances of that new type.

```
Standard Kubernetes:          With CRDs:
  Pods                          Pods
  Services                     Services
  Deployments                  Deployments
  ConfigMaps                   ConfigMaps
                                Buckets (CRD)        ← new!
                                Queues (CRD)         ← new!
                                AppEnvs (CRD)        ← new!
                                Compositions (CRD)   ← new!
```

## How Crossplane Uses CRDs

[[Crossplane]] installs CRDs at multiple levels:

### 1. Crossplane Core CRDs
Installed when you install Crossplane:
- `CompositeResourceDefinition` — for creating XRDs
- `Composition` — for creating Compositions
- `Provider` — for installing providers
- `Function` — for installing functions

### 2. Provider CRDs
Installed when you install a [[Providers|Provider]]:
- `Bucket` (s3.aws.upbound.io) — S3 buckets
- `Queue` (sqs.aws.upbound.io) — SQS queues
- `Instance` (rds.aws.upbound.io) — RDS instances

### 3. XRD-Generated CRDs
Installed when you create an [[Composite Resource Definitions|XRD]]:
- `XAppEnv` (platform.finedge.io) — composite resource
- `AppEnv` (platform.finedge.io) — claim

## CRD Anatomy

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: buckets.s3.aws.upbound.io      # plural.group
spec:
  group: s3.aws.upbound.io              # API group
  names:
    kind: Bucket                         # Resource kind
    plural: buckets                      # Plural name (for kubectl)
    singular: bucket
    shortNames: ["s3bucket"]             # kubectl get s3bucket
  scope: Cluster                         # or Namespaced
  versions:
    - name: v1beta1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                forProvider:
                  type: object
                  properties:
                    region:
                      type: string
                    tags:
                      type: object
                      additionalProperties:
                        type: string
```

## CRD + Controller = Operator Pattern

A CRD alone is just a data store — it doesn't DO anything. You need a **controller** to watch for CRD instances and act on them:

```
CRD (Schema)           + Controller (Logic)    = Operator (Full Feature)
─────────────            ──────────────          ─────────────────────────
"Bucket is a thing"      "When Bucket created,   "Bucket CRD + provider
                          call CreateBucket API"   that manages S3 buckets"
```

In Crossplane:
- **Provider pods** are the controllers for [[Managed Resources]] CRDs
- **Crossplane core** is the controller for XRD and Composition CRDs
- **ArgoCD** is the controller for Application CRDs

## Why CRDs Matter for Platform Engineering

1. **Your platform IS a set of CRDs** — the `AppEnv` CRD is your platform's API
2. **Self-documenting** — `kubectl explain appenv.spec.parameters` shows the API schema
3. **RBAC-compatible** — you can grant/deny access to specific CRDs via [[Kubernetes RBAC]]
4. **Validation built-in** — OpenAPI v3 schema validates input at admission time
5. **Standard tooling** — kubectl, GitOps, admission webhooks all work with CRDs

## Inspecting CRDs

```bash
# List all CRDs in the cluster
kubectl get crds

# Find Crossplane-related CRDs
kubectl get crds | grep crossplane
kubectl get crds | grep platform.finedge.io

# Find provider CRDs
kubectl get crds | grep aws.upbound.io

# Inspect a specific CRD
kubectl explain appenv
kubectl explain appenv.spec
kubectl explain appenv.spec.parameters

# See CRD details
kubectl get crd appenvs.platform.finedge.io -o yaml
```

## CRD Versions and Compatibility

CRDs support multiple versions for API evolution:

```yaml
versions:
  - name: v1alpha1    # Early, unstable API
    served: true
    storage: false
  - name: v1beta1     # Maturing API
    served: true
    storage: true      # This version is stored in etcd
```

Rules for evolving CRDs:
- **Never remove fields** from an existing version
- **Add new optional fields** with defaults
- **Create new versions** for breaking changes
- **Conversion webhooks** can translate between versions

## CRD Count in FinEdge

```
Crossplane core:    ~10 CRDs
provider-aws-s3:    ~10 CRDs (Bucket, BucketVersioning, BucketPolicy, etc.)
provider-aws-sqs:   ~3 CRDs  (Queue, QueuePolicy, etc.)
provider-aws-rds:   ~8 CRDs  (Instance, SubnetGroup, ParameterGroup, etc.)
provider-aws-sns:   ~5 CRDs  (Topic, TopicPolicy, Subscription, etc.)
XRD-generated:      ~2 CRDs  (XAppEnv, AppEnv)
ArgoCD:             ~5 CRDs  (Application, AppProject, etc.)
Kyverno:            ~8 CRDs  (ClusterPolicy, Policy, etc.)
──────────────────────────────
Total:              ~50+ CRDs
```

This is why individual [[Providers]] are preferred over the monolithic `provider-aws` (~900 CRDs!).

## Related

- [[Crossplane]]
- [[Composite Resource Definitions]]
- [[Managed Resources]]
- [[Providers]]
