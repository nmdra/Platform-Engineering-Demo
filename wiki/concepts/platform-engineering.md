---
tags: [concept, platform-engineering, fundamentals]
created: 2026-05-30
---

# Platform Engineering

**Platform Engineering** is the discipline of designing and building toolchains and workflows that enable software engineering organizations to self-serve. It's about building an [[Internal Developer Platform]] that reduces cognitive load so product teams can focus on writing business logic — not wrestling with infrastructure.

## Why Does It Exist?

The "you build it, you run it" DevOps model promised developer empowerment. In practice, it often created **cognitive overload**:

- Teams needed to learn Kubernetes, AWS, Terraform, CI/CD, networking, security, monitoring...
- Every team solved the same problems differently (inconsistent naming, encryption, tagging)
- Platform knowledge was tribal — one person leaves, and the team loses its deployment capability

Platform Engineering emerged as a response: **a dedicated team builds a product (the platform) for other teams**.

## Core Principles

### 1. Self-Service with Guardrails

Teams should be able to provision infrastructure without filing tickets or waiting for a platform engineer. But they shouldn't be able to create unencrypted S3 buckets or skip tagging requirements.

In FinEdge, this looks like:
```yaml
# A team writes 20 lines (the Claim)
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: payments-dev
  namespace: payments
spec:
  parameters:
    teamName: payments
    environment: dev
    storage:
      enabled: true
    queue:
      enabled: true
```

Behind the scenes, [[Crossplane]] provisions 7+ AWS resources with encryption, tagging, DLQs, and versioning — all encoded by the platform team.

### 2. Golden Paths (Not Golden Cages)

A [[Golden Path]] is the opinionated, well-lit, well-documented way to do something. It's not a mandate — it's the path of least resistance. Teams *can* go off-path, but they rarely want to because the golden path is so much easier.

### 3. Cognitive Load Reduction

Team Topologies (Matthew Skelton & Manuel Pais) identifies three types of cognitive load:

| Type | Description | Platform Engineering's Answer |
|------|------------|------------------------------|
| **Intrinsic** | How hard is the problem domain? | Can't reduce — it's your business logic |
| **Extraneous** | How hard is the tooling? | **Eliminate this** — abstract away cloud APIs |
| **Germane** | How hard is learning the domain? | Reduce via docs, templates, golden paths |

### 4. Platform as a Product

The platform team operates like a product team:
- **Users** = product engineering teams (payments, analytics, notifications)
- **Product** = the Internal Developer Platform
- **Feedback loops** = team surveys, support tickets, usage metrics
- **Iteration** = continuously improve based on team needs

## Team Topologies

Platform Engineering draws heavily from the Team Topologies model:

```
┌─────────────────────────────────────────────────┐
│                Stream-Aligned Teams              │
│  (payments, analytics, notifications)            │
│  → Own business logic, submit Claims             │
├─────────────────────────────────────────────────┤
│                 Platform Team                    │
│  → Builds XRDs, Compositions, policies, tooling │
│  → Maintains Crossplane, ArgoCD, Kyverno        │
├─────────────────────────────────────────────────┤
│              Enabling Teams                      │
│  → Help stream-aligned teams adopt the platform  │
│  → Write docs, run workshops, pair on migration  │
└─────────────────────────────────────────────────┘
```

In FinEdge:
- **Platform team** writes [[Compositions]], [[Composite Resource Definitions]], policies, and the bootstrap scripts
- **Stream-aligned teams** (payments, analytics, notifications) submit [[Claims]] and get infrastructure
- The **wiki you're reading** is an enabling resource

## What Makes a Good Platform?

1. **It's optional (but irresistible)** — teams choose it because it's better, not because they're forced
2. **It has clear APIs** — the XRD schema IS the platform's API
3. **It's self-documenting** — `kubectl explain appenv.spec.parameters` tells you what's available
4. **It evolves** — new resource types, better defaults, tighter security
5. **It's tested** — [[Compositions]] are validated with `crossplane render`, policies with `kyverno apply`

## FinEdge Platform Architecture

```
                    Git (Source of Truth)
                           │
                    ┌──────┴──────┐
                    │   ArgoCD    │  ← [[GitOps]]
                    └──────┬──────┘
                           │ sync
                    ┌──────┴──────┐
                    │  Kubernetes  │  ← kind cluster
                    │   Cluster    │
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
     ┌──────┴──────┐ ┌────┴────┐  ┌──────┴──────┐
     │  Crossplane  │ │ Kyverno │  │ Prometheus  │
     │  (XRD+Comp)  │ │(Policies)│  │ + Grafana   │
     └──────┬──────┘ └─────────┘  └─────────────┘
            │
     ┌──────┴──────┐
     │  LocalStack  │  ← AWS emulation
     │  (S3,SQS,RDS)│
     └─────────────┘
```

## How This Differs from DevOps

| Aspect | DevOps | Platform Engineering |
|--------|--------|---------------------|
| **Scope** | Culture + practices | Dedicated product team |
| **Output** | CI/CD pipelines, runbooks | Self-service platform (APIs) |
| **Users** | Same team (you build it, you run it) | Other teams (you build it, *they* run it) |
| **Success metric** | Deployment frequency, MTTR | Developer satisfaction, time-to-first-deploy |
| **Tooling** | Team-specific | Organization-wide, standardized |

## Related

- [[Internal Developer Platform]]
- [[Crossplane]]
- [[GitOps]]
- [[Infrastructure as Code]]
- [[Golden Path]]
- [[Compositions]]
