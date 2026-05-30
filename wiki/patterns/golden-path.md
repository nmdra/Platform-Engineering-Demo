---
tags: [pattern, platform-engineering, developer-experience]
created: 2026-05-29
---

# Golden Path

A **Golden Path** (also called "paved road" or "happy path") is an opinionated, well-supported route for teams to accomplish common tasks — designed by the platform team.

## The Idea

> "Make the right thing the easy thing."

Instead of giving developers a blank canvas and saying "figure it out," you give them a pre-built, tested, supported path that handles 80% of use cases.

## FinEdge Golden Paths

### 🛣️ Path: "I need AWS infrastructure for my service"

```
Step 1: Copy a claim template from infra/<existing-team>/claim.yaml
Step 2: Fill in your team name, environment, and what you need
Step 3: Open a Pull Request
Step 4: CI validates your claim against Kyverno policies
Step 5: PR gets reviewed and merged
Step 6: ArgoCD syncs → Crossplane provisions → done
```

**What the developer doesn't need to know:**
- How S3 encryption works
- How to wire a DLQ to a main queue
- What instance class to use for dev vs prod
- How to tag resources for cost tracking
- How to configure IAM

### 🛣️ Path: "I need to promote from dev to staging"

```
Step 1: Edit your claim.yaml
Step 2: Change compositionSelector.matchLabels.provider: aws
Step 3: Change environment: staging
Step 4: Open a Pull Request
```

### 🛣️ Path: "I need to increase my database size"

```
Step 1: Edit your claim.yaml
Step 2: Change storageGb: 50 → storageGb: 100
Step 3: Open a Pull Request
```

## Characteristics of a Good Golden Path

| Characteristic | FinEdge Example |
|---------------|-----------------|
| **Opinionated** | S3 is always encrypted (you can't turn it off) |
| **Simple** | 20-line YAML claim, no AWS knowledge needed |
| **Self-service** | Teams don't need to file a ticket with the platform team |
| **Guardrailed** | [[Kyverno]] blocks invalid claims before they're applied |
| **Tested** | CI validates claims with `crossplane render` and `kyverno apply` |
| **Documented** | Claim template in `infra/README.md` |
| **Escapable** | Teams CAN request custom resources via platform team consultation |

## The Off-Ramp

A golden path isn't a golden cage. When teams need something the path doesn't support:

1. Team opens a discussion with the platform team
2. Platform team evaluates whether to extend the XRD/Composition
3. If it's a common need → add it to the golden path
4. If it's a one-off → help the team directly

## Related

- [[Platform Engineering]]
- [[Internal Developer Platform]]
- [[Composition Pattern]]
- [[Claims]]
- [[Policy as Code]]
