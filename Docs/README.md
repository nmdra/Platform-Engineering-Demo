# Docs/ — Project Documentation

This directory contains the project documentation, implementation guides, and architectural decision records.

## Files

| File | Description |
|------|-------------|
| `initial_plan.md` | **Full implementation guide** — 8 phases from bootstrap to production graduation |

## Implementation Phases

| # | Phase | Description |
|---|-------|-------------|
| 1 | Bootstrap | kind + LocalStack 2026.x + Crossplane v2.3 + Functions |
| 2 | Platform API | XRD + Composition with auto-ready & deletionPolicy |
| 3 | Security + Claims | RBAC, NetworkPolicy, Quotas, PSA + team claims |
| 4 | GitOps | ArgoCD app-of-apps with safe prune settings |
| 5 | Guardrails | Kyverno policies (owner, encryption, prod sizing) |
| 6 | Observability | Prometheus rules + Grafana + drift demo |
| 7 | Testing | Composition render tests + policy validation + CI |
| 8 | Graduation | One-label flip from LocalStack to real AWS |
