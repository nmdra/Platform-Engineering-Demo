---
tags: [concept, platform-engineering, idp, fundamentals]
created: 2026-05-30
---

# Internal Developer Platform

An **Internal Developer Platform (IDP)** is the product that a [[Platform Engineering]] team builds. It's a self-service layer on top of your infrastructure that lets product teams provision and manage resources without deep cloud expertise.

## What Makes It a "Platform"?

An IDP isn't just a collection of scripts or a wiki page with AWS console screenshots. It's a **product with APIs, documentation, and versioning**:

| Component | What It Provides | FinEdge Implementation |
|-----------|-----------------|----------------------|
| **API** | A schema teams interact with | [[Composite Resource Definitions]] (XRD) |
| **Implementation** | Logic that fulfills requests | [[Compositions]] |
| **Request mechanism** | How teams ask for infrastructure | [[Claims]] (`AppEnv`) |
| **Policy layer** | Guardrails and compliance | [[Kyverno]] policies |
| **Delivery mechanism** | How changes reach the cluster | [[ArgoCD]] ([[GitOps]]) |
| **Observability** | Visibility into platform health | [[Prometheus and Grafana]] |

## The Developer Experience

### Without an IDP (The Bad Old Days)

```
Developer: "I need an S3 bucket for my service"
    → File a Jira ticket
    → Wait 3 days for platform team to respond
    → Back-and-forth on requirements
    → Platform team provisions manually
    → Developer gets credentials via Slack DM (😱)
    → Total time: 1-2 weeks
```

### With an IDP (FinEdge Model)

```
Developer: "I need an S3 bucket for my service"
    → Copy the claim template
    → Fill in teamName, environment, storage.enabled: true
    → git push
    → ArgoCD syncs → Crossplane provisions → bucket exists
    → Connection details appear in a Kubernetes Secret
    → Total time: 5 minutes
```

## The FinEdge IDP in Practice

### The Claim: Your Platform's API

```yaml
# This is all a developer writes
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: payments-dev
  namespace: payments
  annotations:
    platform.finedge.io/owner: "dev-lead@finedge.io"
    platform.finedge.io/ticket: "INFRA-042"
spec:
  parameters:
    teamName: payments
    environment: dev
    storage:
      enabled: true          # → gets an S3 bucket (encrypted, versioned)
    queue:
      enabled: true           # → gets SQS queue + DLQ (wired together)
    database:
      enabled: false          # → no RDS instance
    notifications:
      enabled: true           # → gets an SNS topic
```

### What They DON'T Need to Know

The developer above doesn't need to know:
- S3 bucket encryption uses AES-256 server-side encryption
- SQS dead-letter queue has 14-day retention and maxReceiveCount of 3
- Bucket naming follows `finedge-<team>-<env>-data` pattern
- Resources are tagged with `ManagedBy: crossplane`, `Platform: finedge`
- [[Network Policies]] prevent their namespace from talking to other team namespaces
- [[Pod Security Admission]] enforces baseline security profile

All of that is encoded in the [[Compositions|Composition]] — written once by the platform team.

## Layers of Abstraction

```
Layer 5: Developer writes a Claim (20 lines)
           │
Layer 4: XRD validates the Claim schema
           │
Layer 3: Composition maps Claim → Managed Resources
           │
Layer 2: Crossplane Provider calls cloud APIs
           │
Layer 1: Cloud provider (LocalStack / AWS) creates resources
```

Each layer hides complexity from the one above:
- **Developers** see Layer 5 (Claims)
- **Platform engineers** work at Layers 3-4 (Compositions, XRDs)
- **Crossplane** handles Layer 2 (API translation)
- **Cloud** handles Layer 1 (actual resource creation)

## Designing Good Abstractions

The hardest part of building an IDP is getting the abstraction level right:

### Too Low (Leaky Abstraction)
```yaml
# ❌ Bad: Exposes cloud-specific details
spec:
  parameters:
    s3BucketAcl: "private"
    rdsInstanceClass: "db.t3.micro"
    sqsVisibilityTimeout: 30
```
Teams shouldn't need to know about ACLs, instance classes, or visibility timeouts.

### Too High (Useless Abstraction)
```yaml
# ❌ Bad: Too abstract, no flexibility
spec:
  parameters:
    size: "small"  # What does "small" mean?!
```
Teams can't make meaningful choices.

### Just Right (FinEdge Model)
```yaml
# ✅ Good: Business-domain concepts
spec:
  parameters:
    teamName: payments           # Who owns it
    environment: dev             # Where it lives
    storage:
      enabled: true              # Do you need storage?
    database:
      enabled: true
      size: medium               # Meaningful sizes (small=t3.micro, medium=t3.small)
```
Teams make business decisions. The platform translates to cloud decisions.

## Self-Service Maturity Model

| Level | Description | FinEdge Status |
|-------|------------|----------------|
| 0 — Manual | Tickets and manual provisioning | ❌ |
| 1 — Scripted | Shell scripts, Terraform modules | ❌ |
| 2 — Declarative | Kubernetes-native, API-driven | ✅ (Crossplane) |
| 3 — Self-Healing | Drift detection, auto-reconciliation | ✅ ([[Reconciliation Loop]]) |
| 4 — Policy-Governed | Automated compliance, guardrails | ✅ ([[Policy as Code]]) |
| 5 — Observable | Full visibility into platform health | 🔄 (In progress) |

## Related

- [[Platform Engineering]]
- [[Claims]]
- [[Composite Resource Definitions]]
- [[Crossplane]]
- [[Golden Path]]
- [[Policy as Code]]
