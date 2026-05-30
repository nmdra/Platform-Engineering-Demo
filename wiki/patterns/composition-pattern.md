---
tags: [pattern, crossplane, platform-engineering]
created: 2026-05-29
---

# Composition Pattern

The **Composition Pattern** is the central design pattern in [[Crossplane]] — it's how the platform team creates abstractions that hide infrastructure complexity from product teams.

## The Problem

Without this pattern, every team would need to:
- Understand AWS APIs, IAM, networking, encryption, tagging
- Write hundreds of lines of infrastructure configuration
- Maintain consistency across teams and environments
- Know the "right" way to wire resources together (e.g., DLQ → main queue)

## How It Works

```
Team writes 20 lines          Platform team wrote 200+ lines
(the Claim)                   (the Composition — once)
┌──────────────┐              ┌──────────────────────────────────┐
│ AppEnv        │──────────►  │ Composition                       │
│   teamName    │              │   ├─ S3 Bucket (encrypted, tagged)│
│   environment │              │   ├─ BucketVersioning             │
│   storage:    │              │   ├─ BucketEncryption             │
│     enabled   │              │   ├─ SQS DLQ (14-day retention)   │
│   queue:      │              │   ├─ SQS Main Queue (wired to DLQ)│
│     enabled   │              │   ├─ RDS Instance (encrypted)     │
│               │              │   └─ SNS Topic                    │
└──────────────┘              └──────────────────────────────────┘
```

## Anatomy of a Good Composition

### 1. Naming Conventions
Resources should follow a consistent naming pattern:
```yaml
# Pattern: finedge-<team>-<env>-<purpose>
fmt: "finedge-%s-%s-data"       # → finedge-payments-dev-data
fmt: "finedge-%s-%s-events"     # → finedge-payments-dev-events
fmt: "finedge-%s-%s-events-dlq" # → finedge-payments-dev-events-dlq
```

### 2. Explicit [[Deletion Policy]]
```yaml
spec:
  deletionPolicy: Delete    # dev/LocalStack
  # OR
  deletionPolicy: Orphan    # prod databases
```

### 3. Consistent Tags
```yaml
tags:
  ManagedBy: crossplane
  Platform: finedge
  Team: <patched-from-claim>
  Environment: <patched-from-claim>
```

### 4. Pipeline Steps
```yaml
pipeline:
  - step: patch-and-transform    # Map claim fields → resource fields
    functionRef:
      name: function-patch-and-transform
  - step: auto-ready             # Aggregate child readiness
    functionRef:
      name: function-auto-ready
```

## Design Principles

1. **Teams specify WHAT, not HOW** — Claims have simple boolean/enum fields, not cloud-specific configuration
2. **Platform team encodes decisions** — Encryption, tagging, naming, DLQ wiring are all in the Composition
3. **One Composition per environment class** — LocalStack composition vs AWS composition
4. **Composition selection via labels** — `compositionSelector.matchLabels.provider: localstack|aws`
5. **Defaults in the [[Composite Resource Definitions|XRD]]** — Teams only specify what's different from the default

## Anti-Patterns

| Anti-Pattern | Why It's Bad |
|-------------|-------------|
| Exposing cloud-specific fields in the XRD | Defeats the abstraction; teams shouldn't know about `instanceClass` details |
| One monolithic Composition for all environments | Makes it impossible to have different deletion policies per env |
| No `function-auto-ready` | XR shows Ready before child resources (especially RDS) are actually ready |
| Hardcoded names instead of patches | Can't have multiple claims with different team/env combinations |

## Related

- [[Crossplane]]
- [[Compositions]]
- [[Composite Resource Definitions]]
- [[Claims]]
- [[Composition Functions]]
- [[Golden Path]]
