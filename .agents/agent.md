# Agent Instructions — FinEdge Platform Engineering Demo

> **This file provides context and rules for AI agents (Gemini, Claude, Copilot, Cursor, etc.)
> working on this codebase. Read this before making any changes.**

---

## Project Overview

This is a **Platform Engineering demo** that builds a self-service Internal Developer Platform (IDP) for a fictional fintech company called **FinEdge**. The platform uses **Crossplane v2.3** with **LocalStack** to let product teams provision AWS infrastructure via simple YAML claims — without touching AWS directly.

### Tech Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| Crossplane | v2.3.1 | Infrastructure orchestration via Kubernetes CRDs |
| LocalStack | >= 2026.01 (CalVer) | Local AWS emulation — no real account needed |
| ArgoCD | v3.4.x | GitOps — cluster state synced from Git |
| Kyverno | v1.18.x | Policy engine — admission control for claims |
| kind | >= 0.31 | Local Kubernetes cluster (Docker-based) |
| Helm | >= 3.14 | Package manager for K8s components |
| kube-prometheus-stack | latest | Monitoring — Prometheus + Grafana |

### Key Abstractions

- **XRD (`xappenvs.platform.finedge.io`)** — the schema/API that teams interact with
- **Composition** — the platform team's implementation that maps claims to AWS resources
- **Claim (`AppEnv`)** — a team's 20-line request for infrastructure
- **ProviderConfig** — the wiring that routes API calls to LocalStack or real AWS
- **Composition Functions** — `function-patch-and-transform` (resource mapping) + `function-auto-ready` (readiness aggregation)

---

## Directory Structure & Ownership

```
finedge-platform/
├── Docs/                        # Documentation (plan, guides, ADRs)
├── kind/                        # kind cluster configuration
├── crossplane/                  # ⚠️ PLATFORM TEAM ONLY — XRD, Compositions, Providers
├── infra/                       # Team claims — one subdirectory per team
│   ├── payments/
│   ├── analytics/
│   └── notifications/
├── security/                    # ⚠️ PLATFORM TEAM ONLY — RBAC, NetworkPolicies, Quotas
│   ├── rbac/
│   ├── network-policies/
│   └── resource-quotas/
├── policies/                    # ⚠️ PLATFORM TEAM ONLY — Kyverno policies
├── gitops/                      # ArgoCD application definitions
│   └── apps/                    # Per-team ArgoCD Application manifests
├── observability/               # Prometheus rules, Grafana dashboards
├── tests/                       # Validation tests
│   ├── compositions/            # crossplane render test fixtures
│   └── policies/                # Kyverno policy test fixtures (valid + invalid)
├── scripts/                     # Automation scripts (bootstrap, teardown, validate)
├── wiki/                        # Obsidian-style learning knowledge base
│   ├── concepts/                # Core concept pages
│   ├── tools/                   # Tool guides
│   ├── patterns/                # Architecture patterns
│   ├── troubleshooting/         # Issue resolution guides
│   └── journal/                 # Daily learning notes
├── .agents/
│   └── agent.md                 # THIS FILE — AI agent instructions
└── README.md                    # Project overview and quickstart
```

---

## Wiki Maintenance Rules (MANDATORY)

> **All AI agents working on this project MUST follow these wiki update rules.**
> The wiki at `wiki/` is a living knowledge base that must stay current.

### When to Create Wiki Pages

| Trigger | Action | Location |
|---------|--------|----------|
| Encounter a **new concept** not yet in the wiki | Create a new concept page | `wiki/concepts/<concept-name>.md` |
| Solve an **issue or error** | Create a troubleshooting page | `wiki/troubleshooting/<issue-name>.md` |
| Add or modify a **tool** | Create/update the tool page | `wiki/tools/<tool-name>.md` |
| Discover a reusable **pattern** | Create a pattern page | `wiki/patterns/<pattern-name>.md` |
| Make significant **platform changes** | Update related concept/tool pages | Existing wiki pages |

### Wiki Page Format

Every wiki page MUST have:

1. **YAML frontmatter** with `tags` and `created` date:
   ```yaml
   ---
   tags: [concept, crossplane, relevant-topic]
   created: YYYY-MM-DD
   ---
   ```

2. **Title** as `# Heading` — this MUST match the `[[wikilink]]` name used elsewhere

3. **Teaching-oriented content** — explain WHY, not just WHAT:
   - What is it and why does it exist?
   - How does it work in the FinEdge context?
   - Include code examples from this project
   - Link to 3-5 other wiki pages minimum using `[[wikilinks]]`

4. **`## Related` section** at the bottom with wikilinks to related pages

### Wikilink Format

- Use Obsidian-style `[[Page Title]]` syntax (NOT markdown links)
- Page title must match the `# Heading` of the target page
- Examples: `[[Crossplane]]`, `[[Infrastructure as Code]]`, `[[ArgoCD]]`

### Troubleshooting Page Format

When creating troubleshooting pages, use this structure:
```markdown
---
tags: [troubleshooting, <relevant-topic>]
created: YYYY-MM-DD
---

# <Issue Title>

## Symptom
What you observe when the issue occurs.

## Root Cause
Why the issue happens.

## Solution
Step-by-step fix with commands and YAML.

## Prevention
How to avoid this issue in the future.

## Related
- [[Relevant Page 1]]
- [[Relevant Page 2]]
```

### Updating Existing Pages

When modifying platform components (Compositions, XRDs, policies, etc.):
1. Check if related wiki pages exist
2. Update them if the change affects the documented behavior
3. Add new `[[wikilinks]]` if introducing new concepts

### Wiki Index

After creating a new page, add it to `wiki/_index.md` in the appropriate section.

---

## Critical Rules for AI Agents

### 1. YAML Formatting

All Kubernetes manifests in this project use:
- **2-space indentation** (standard YAML)
- **Explicit `apiVersion` and `kind`** on every resource
- **`---` separator** between multiple resources in the same file
- **Comments** explaining non-obvious decisions (start with `#`)
- **No tabs** — spaces only

### 2. Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Kubernetes resources | `finedge-<team>-<env>-<purpose>` | `finedge-payments-dev-data` |
| S3 buckets | `finedge-<team>-<env>-data` | `finedge-analytics-dev-data` |
| SQS queues | `finedge-<team>-<env>-events` | `finedge-payments-dev-events` |
| SQS DLQ | `finedge-<team>-<env>-events-dlq` | `finedge-payments-dev-events-dlq` |
| SNS topics | `finedge-<team>-<env>-notify` | `finedge-notifications-dev-notify` |
| RDS instances | `finedge-<team>-<env>-db` | `finedge-analytics-dev-db` |
| Namespaces | Team name (lowercase) | `payments`, `analytics`, `notifications` |
| Kyverno policies | `finedge-<description>` | `finedge-appenv-require-owner` |

### 3. Required Labels & Annotations

Every **AppEnv claim** must have:
```yaml
metadata:
  annotations:
    platform.finedge.io/owner: "<email>@finedge.io"    # Required — Kyverno enforced
    platform.finedge.io/ticket: "INFRA-<number>"       # Required — Kyverno enforced
```

Every **composed managed resource** must have tags:
```yaml
tags:
  ManagedBy: crossplane
  Platform: finedge
  Team: <teamName>           # Patched from claim
  Environment: <environment> # Patched from claim
```

### 4. Crossplane v2 Specifics

- **Always use `mode: Pipeline`** in Compositions — `mode: Resources` is removed in v2
- **Always include `function-auto-ready`** as the last pipeline step
- **Always set `deletionPolicy`** explicitly — never rely on defaults
  - `Delete` for dev/LocalStack environments
  - `Orphan` for production RDS instances
- **`providerConfigRef.name: localstack`** for all LocalStack compositions
- **`providerConfigRef.name: aws-real`** for production AWS compositions
- **Usages are GA** — no `--enable-usages` flag needed
- The `function-patch-and-transform` input uses `apiVersion: pt.fn.crossplane.io/v1beta1`

### 5. Security Constraints

- **Never create resources in the `default` namespace** — use team namespaces
- **Never add `cluster-admin` bindings** for team developers — use `finedge-team-developer` role
- **Never set `publiclyAccessible: true`** on RDS instances
- **Never set `encryption: false`** on S3 buckets (PCI-DSS)
- **Pod Security Admission** is enforced:
  - `baseline` (enforce) on team namespaces
  - `restricted` (warn + audit) on team namespaces
- **Network policies** default-deny ingress/egress in team namespaces

### 6. ArgoCD Safety

- Root app-of-apps uses `prune: true` (manages child Application manifests)
- **Team apps use `prune: false`** — prevents accidental infra deletion when a claim is removed from Git
- Always use `selfHeal: true` for drift correction

### 7. Kyverno Policy Rules

- Policies currently use legacy `ClusterPolicy` (kyverno.io/v1) — this is **deprecated** but functional
- Migration to CEL-based `ValidatingPolicy` is planned before October 2026
- When writing new policies, prefer `ValidatingPolicy` (CEL) over `ClusterPolicy` (JMESPath)
- Always set `validationFailureAction: Enforce` (not `Audit`)

### 8. LocalStack Specifics

- LocalStack uses **calendar versioning** (`YYYY.MM.patch`) since March 2026
- **Auth token required** — set `LOCALSTACK_AUTH_TOKEN` env var
- All services start by default — no `SERVICES=` env var needed
- Endpoint: `http://host.docker.internal:4566` (from inside kind)
- Endpoint: `http://localhost:4566` (from host machine)
- Credentials: `aws_access_key_id=test`, `aws_secret_access_key=test`

---

## Common Tasks for AI Agents

### Adding a New Team

1. Create `infra/<team-name>/claim.yaml` following existing patterns
2. Add namespace creation to bootstrap flow (with PSA labels)
3. Add `RoleBinding` in `security/rbac/team-rolebindings.yaml`
4. Add `NetworkPolicy` for the namespace
5. Add `ResourceQuota` for the namespace
6. Add ArgoCD `Application` in `gitops/apps/<team-name>.yaml`
7. Add claim to validation tests in `tests/policies/`

### Adding a New AWS Resource to the Composition

1. Add the resource type to `crossplane/providers.yaml` (if not already installed)
2. Add the resource to the XRD schema in `crossplane/xrd-appenv.yaml`
3. Add the resource definition to `crossplane/composition-appenv.yaml` (before `auto-ready` step)
4. Set `deletionPolicy: Delete` explicitly
5. Add proper tags and patches following existing patterns
6. Add corresponding entry in the prod composition (`composition-appenv-aws.yaml`)
7. Test with `crossplane render` before applying

### Adding a New Kyverno Policy

1. Create the policy file in `policies/`
2. Name it `finedge-<description>.yaml`
3. Set `validationFailureAction: Enforce`
4. Add test fixtures in `tests/policies/` (both valid and invalid)
5. Test locally with `kyverno apply`

### Modifying the XRD Schema

1. Edit `crossplane/xrd-appenv.yaml`
2. Add new fields under `spec.parameters` with proper defaults
3. Update the Composition to use the new fields
4. Update team claims if needed
5. Update test fixtures
6. **Never remove existing fields** — only add new optional ones (backward compatibility)

---

## Environment Details

### Cluster Layout

| Namespace | Purpose | Components |
|-----------|---------|------------|
| `crossplane-system` | Crossplane core + providers + functions | Crossplane pods, provider pods, function pods |
| `argocd` | GitOps | ArgoCD server, repo server, application controller |
| `kyverno` | Policy engine | Kyverno admission controller |
| `monitoring` | Observability | Prometheus, Grafana, Alertmanager |
| `payments` | Payments team | AppEnv claims |
| `analytics` | Analytics team | AppEnv claims |
| `notifications` | Notifications team | AppEnv claims |

### API Groups

| Group | Resources | Scope |
|-------|-----------|-------|
| `platform.finedge.io` | `AppEnv` (claim), `XAppEnv` (composite) | Claims are namespaced; XRs are cluster-scoped |
| `s3.aws.upbound.io` | `Bucket`, `BucketVersioning`, `BucketServerSideEncryptionConfiguration` | Managed resources |
| `sqs.aws.upbound.io` | `Queue` | Managed resources |
| `rds.aws.upbound.io` | `Instance` | Managed resources |
| `sns.aws.upbound.io` | `Topic` | Managed resources |
| `aws.upbound.io` | `ProviderConfig` | Provider configuration |

### Useful Commands

```bash
# See all claims across namespaces
kubectl get appenv -A

# See all managed AWS resources
kubectl get managed

# Trace a claim's full resource tree
kubectl crossplane beta trace appenv <name> -n <namespace>

# Render composition locally (no cluster needed)
crossplane render infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml

# Test a Kyverno policy locally
kyverno apply policies/ --resource tests/policies/test-valid-claim.yaml

# Check provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/revision=provider-aws-s3 --tail=40

# Check function logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/revision=function-patch-and-transform --tail=20
```

---

## Implementation Reference

The full step-by-step implementation guide is in [Docs/initial_plan.md](Docs/initial_plan.md).
It covers all 8 phases from bootstrap to production graduation.

---

## Linting & Validation

- **Only run linting on services/directories that have changes** — do not run all linting tasks at once
- Use `crossplane render` to validate composition changes
- Use `kyverno apply` to validate policy changes
- Use `kubectl apply --dry-run=server` to validate manifests against the cluster API

---

## Version Pinning

All versions are explicitly pinned in manifests. When updating:

| Manifest | What to update |
|----------|---------------|
| `crossplane/providers.yaml` | `spec.package` version tags |
| `crossplane/functions.yaml` | `spec.package` version tags |
| Helm installs in `Docs/initial_plan.md` | `--version` flags |
| `README.md` | Badge versions |
