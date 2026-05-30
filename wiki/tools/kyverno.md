---
tags: [tool, kyverno, policy, admission-control]
created: 2026-05-30
---

# Kyverno

**Kyverno** (Greek for "govern") is a Kubernetes-native policy engine. In FinEdge, it enforces organizational rules — every [[Claims|Claim]] must have an owner, every resource must have a ticket reference, and no one creates resources in the `default` namespace. It's the enforcement arm of [[Policy as Code]].

## Why Kyverno?

Unlike OPA/Gatekeeper which requires learning Rego (a specialized policy language), Kyverno policies are **just YAML**. Platform engineers already write YAML all day — adding policy rules feels natural.

## Policy Types

### 1. ClusterPolicy (Legacy — kyverno.io/v1)

The original policy format. Still functional but deprecated:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: finedge-appenv-require-owner
  annotations:
    policies.kyverno.io/title: "Require Owner Annotation"
    policies.kyverno.io/description: >-
      All AppEnv claims must have a platform.finedge.io/owner annotation
      to ensure accountability.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-owner
      match:
        any:
          - resources:
              kinds:
                - AppEnv
      validate:
        message: >-
          AppEnv claims must have annotation platform.finedge.io/owner.
          Example: platform.finedge.io/owner: "dev-lead@finedge.io"
        pattern:
          metadata:
            annotations:
              platform.finedge.io/owner: "?*"
```

### 2. ValidatingPolicy (Modern — CEL-based)

The new format using Common Expression Language (CEL). Kyverno v1.18+ supports this:

```yaml
apiVersion: kyverno.io/v2alpha1
kind: ValidatingPolicy
metadata:
  name: finedge-require-owner-cel
spec:
  failureAction: Enforce
  matchConstraints:
    resourceRules:
      - apiGroups: ["platform.finedge.io"]
        apiVersions: ["*"]
        resources: ["appenvs"]
        operations: ["CREATE", "UPDATE"]
  validations:
    - expression: >-
        has(object.metadata.annotations) &&
        'platform.finedge.io/owner' in object.metadata.annotations &&
        object.metadata.annotations['platform.finedge.io/owner'].size() > 0
      message: "AppEnv claims must have a non-empty platform.finedge.io/owner annotation"
```

### Migration Path

FinEdge currently uses `ClusterPolicy` (v1) but plans to migrate to `ValidatingPolicy` (CEL) before October 2026. When writing **new** policies, prefer the CEL format.

## FinEdge Policies

| Policy | Purpose | Enforcement |
|--------|---------|-------------|
| `finedge-appenv-require-owner` | Owner annotation on all Claims | Enforce |
| `finedge-appenv-require-ticket` | Ticket annotation on all Claims | Enforce |
| `finedge-block-default-namespace` | No resources in `default` | Enforce |
| `finedge-require-team-label` | Team label on all Claims | Enforce |

## Policy Capabilities

### Validation (Reject Bad Resources)
```yaml
validate:
  message: "S3 buckets must have encryption enabled"
  pattern:
    spec:
      forProvider:
        serverSideEncryptionConfiguration:
          rule:
            - applyServerSideEncryptionByDefault:
                sseAlgorithm: "aws:kms"
```

### Mutation (Auto-Fix Resources)
```yaml
mutate:
  patchStrategicMerge:
    metadata:
      labels:
        managed-by: platform-team
```

### Generation (Create Companion Resources)
```yaml
generate:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  name: default-deny
  namespace: "{{request.object.metadata.name}}"
  data:
    spec:
      podSelector: {}
      policyTypes:
        - Ingress
        - Egress
```

## Testing Policies

### CLI Testing (No Cluster Needed)

```bash
# Test a policy against a valid resource
kyverno apply policies/finedge-appenv-require-owner.yaml \
  --resource tests/policies/test-valid-claim.yaml

# Expected output:
# pass: 1, fail: 0, warn: 0, error: 0

# Test against an invalid resource
kyverno apply policies/finedge-appenv-require-owner.yaml \
  --resource tests/policies/test-invalid-claim.yaml

# Expected output:
# pass: 0, fail: 1, warn: 0, error: 0
```

### Test Fixtures

```yaml
# tests/policies/test-valid-claim.yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: test-valid
  namespace: payments
  annotations:
    platform.finedge.io/owner: "test@finedge.io"      # ✅ Present
    platform.finedge.io/ticket: "INFRA-TEST"           # ✅ Present
spec:
  parameters:
    teamName: payments
    environment: dev

# tests/policies/test-invalid-claim.yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: test-invalid
  namespace: payments
  # ❌ Missing owner and ticket annotations
spec:
  parameters:
    teamName: payments
    environment: dev
```

## Enforce vs. Audit

| Mode | `validationFailureAction` | Behavior |
|------|--------------------------|----------|
| **Enforce** | `Enforce` | Reject the resource (kubectl returns error) |
| **Audit** | `Audit` | Allow but create a PolicyReport |

**FinEdge default**: Always `Enforce`. Use `Audit` only when testing new policies.

## Debugging

```bash
# Check Kyverno pod status
kubectl get pods -n kyverno

# View policy reports
kubectl get policyreport -A

# Check admission webhook logs
kubectl logs -n kyverno -l app.kubernetes.io/component=admission-controller --tail=50

# Test why a resource was rejected
kubectl describe clusterPolicy finedge-appenv-require-owner
```

## Related

- [[Policy as Code]]
- [[Namespace Isolation]]
- [[Claims]]
- [[Pod Security Admission]]
