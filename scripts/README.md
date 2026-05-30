# scripts/ — Automation Scripts

This directory contains helper scripts for bootstrapping, tearing down, and validating the FinEdge platform.

## Scripts

| Script | Description |
|--------|-------------|
| `bootstrap.sh` | Full platform setup — creates cluster, installs all components |
| `teardown.sh` | Clean teardown — deletes claims, then cluster |
| `validate.sh` | Runs all validation tests (composition render + policy checks) |

## Usage

```bash
# Full bootstrap (kind + LocalStack + Crossplane + providers + XRD + composition)
./scripts/bootstrap.sh

# Tear everything down
./scripts/teardown.sh

# Run validation checks
./scripts/validate.sh
```

> **Note:** Scripts should be run from the project root directory.
