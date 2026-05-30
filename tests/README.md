# tests/ — Validation & Testing

This directory contains test fixtures for validating Crossplane compositions and Kyverno policies **before** they reach the cluster.

## Structure

```
tests/
├── compositions/
│   ├── test-appenv-render.yaml       # crossplane render test input
│   └── observed-resources.yaml       # Mock observed state for render
└── policies/
    ├── test-valid-claim.yaml         # Claim that should PASS all policies
    └── test-invalid-claim.yaml       # Claim that should FAIL policies
```

## Running Tests

### Composition Rendering

```bash
# Render a composition locally to see what resources it would create
crossplane render \
  infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml

# Render with observed state (for update scenarios)
crossplane render \
  infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml \
  --observed-resources tests/compositions/observed-resources.yaml
```

### Policy Validation

```bash
# Should pass — valid claim with proper annotations
kyverno apply policies/ --resource tests/policies/test-valid-claim.yaml

# Should fail — missing annotations, encryption off
kyverno apply policies/ --resource tests/policies/test-invalid-claim.yaml
```

### CI Integration

These tests are designed to run in CI pipelines. See [Docs/initial_plan.md](../Docs/initial_plan.md) Phase 7.3 for a GitHub Actions workflow example.

## Adding New Tests

1. Create test fixture YAML in the appropriate subdirectory
2. Follow existing naming: `test-<description>.yaml`
3. For policy tests, always create both valid and invalid variants
4. Document what each test validates in YAML comments
