# policies/ — Kyverno Guardrail Policies

This directory contains **Kyverno** admission control policies that enforce platform standards on all `AppEnv` claims.

> **⚠️ Platform Team Only:** Only the platform team should modify these files.

## Policies

| File | Policy | Action |
|------|--------|--------|
| `require-owner-annotation.yaml` | Every claim needs `platform.finedge.io/owner` (email) and `platform.finedge.io/ticket` (INFRA-NNN) | **Enforce** |
| `block-public-buckets.yaml` | Storage must have `encryption: true` (PCI-DSS) | **Enforce** |
| `restrict-prod-instance.yaml` | Prod databases can't use `db.t3.micro`; must have `storageGb >= 50` | **Enforce** |

## Testing Policies Locally

```bash
# Test a valid claim passes
kyverno apply policies/ --resource tests/policies/test-valid-claim.yaml

# Test an invalid claim is rejected
kyverno apply policies/ --resource tests/policies/test-invalid-claim.yaml
```

## Adding a New Policy

1. Create `policies/finedge-<description>.yaml`
2. Set `validationFailureAction: Enforce`
3. Add test fixtures in `tests/policies/`
4. Test locally with `kyverno apply`
5. Apply: `kubectl apply -f policies/`

## API Migration Note

These policies use the legacy `ClusterPolicy` API (`kyverno.io/v1`), which is deprecated and will be removed in Kyverno v1.20 (~October 2026). New policies should use the CEL-based `ValidatingPolicy` API. See [Docs/initial_plan.md](../Docs/initial_plan.md) Phase 5.3 for an example.
