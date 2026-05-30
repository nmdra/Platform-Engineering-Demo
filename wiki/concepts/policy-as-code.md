---
tags: [concept, policy, kyverno, admission-control]
created: 2026-05-30
---

# Policy as Code

**Policy as Code** is the practice of expressing organizational rules, compliance requirements, and security guardrails as machine-readable definitions that are automatically enforced. In FinEdge, policies ensure that every [[Claims|Claim]] has an owner, every resource follows naming conventions, and nobody creates resources in the wrong namespace.

## Why Policy as Code?

Without it:
- "Please remember to add the owner annotation" → forgotten 50% of the time
- "Don't create resources in the default namespace" → someone does it anyway
- "All S3 buckets must be encrypted" → one unencrypted bucket in production

With it:
- `kubectl apply` of a Claim WITHOUT owner annotation → **rejected instantly**
- `kubectl apply` of a resource in `default` namespace → **rejected instantly**
- These rules are version-controlled, tested, and self-documenting

## Kubernetes Admission Control

Policy engines work via Kubernetes **admission webhooks** — they intercept API requests BEFORE they're persisted to etcd:

```
User/ArgoCD                  API Server                 Policy Engine
    │                            │                           │
    │── kubectl apply ──────────▶│                           │
    │                            │── admission webhook ────▶│
    │                            │                           │── evaluate rules
    │                            │                           │
    │                            │◀── allow/deny ──────────│
    │                            │                           │
    │◀── Created / Rejected ────│                           │
```

## FinEdge Policy Engine: Kyverno

FinEdge uses [[Kyverno]] for policy enforcement. See the [[Kyverno]] page for implementation details.

### Example: Require Owner Annotation

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: finedge-appenv-require-owner
spec:
  validationFailureAction: Enforce    # ← Reject invalid resources
  rules:
    - name: require-owner-annotation
      match:
        any:
          - resources:
              kinds:
                - AppEnv
      validate:
        message: "AppEnv claims must have a platform.finedge.io/owner annotation"
        pattern:
          metadata:
            annotations:
              platform.finedge.io/owner: "?*"
```

### Example: Block Default Namespace

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: finedge-block-default-namespace
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-default
      match:
        any:
          - resources:
              kinds:
                - AppEnv
              namespaces:
                - default
      validate:
        deny: {}
        message: "Resources must not be created in the default namespace"
```

## Kyverno vs. OPA/Gatekeeper

| Feature | Kyverno | OPA Gatekeeper |
|---------|---------|---------------|
| Policy language | YAML (familiar) | Rego (new language to learn) |
| Validation | Built-in patterns | Constraint Templates |
| Mutation | Yes (modify resources) | Limited |
| Generation | Yes (create resources) | No |
| CEL support | Yes (v1.18+ `ValidatingPolicy`) | Yes (via K8s ValidatingAdmissionPolicy) |
| Learning curve | Low | High (Rego is powerful but complex) |
| FinEdge choice | ✅ | ❌ |

FinEdge chose Kyverno because:
1. YAML-native — no new language to learn
2. Can validate, mutate, AND generate resources
3. Lower barrier for platform engineers to write policies
4. CEL migration path for modern validation

## Policy Types

### 1. Validation (Block Bad Resources)
```yaml
# "AppEnv must have owner annotation"
validate:
  pattern:
    metadata:
      annotations:
        platform.finedge.io/owner: "?*"
```

### 2. Mutation (Auto-Fix Resources)
```yaml
# "Add ManagedBy label if missing"
mutate:
  patchStrategicMerge:
    metadata:
      labels:
        ManagedBy: crossplane
```

### 3. Generation (Create Companion Resources)
```yaml
# "When a namespace is created, generate a NetworkPolicy"
generate:
  kind: NetworkPolicy
  name: default-deny
  namespace: "{{request.object.metadata.name}}"
```

## Enforce vs. Audit

| Mode | Behavior | FinEdge Default |
|------|----------|-----------------|
| **Enforce** | Reject invalid resources | ✅ Used for all policies |
| **Audit** | Allow but log violation | Used during policy testing |

Always use `Enforce` in production. Use `Audit` only when testing new policies.

## Testing Policies

```bash
# Test a policy against a valid resource
kyverno apply policies/ --resource tests/policies/test-valid-claim.yaml
# → Pass ✅

# Test against an invalid resource
kyverno apply policies/ --resource tests/policies/test-invalid-claim.yaml
# → Fail ❌ (policy violation detected)
```

## Related

- [[Kyverno]]
- [[Namespace Isolation]]
- [[Kubernetes RBAC]]
- [[Pod Security Admission]]
- [[Claims]]
