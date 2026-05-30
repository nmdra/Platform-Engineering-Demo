---
tags: [concept, iac, fundamentals]
created: 2026-05-30
---

# Infrastructure as Code

**Infrastructure as Code (IaC)** is the practice of managing and provisioning infrastructure through machine-readable definition files rather than manual processes. But not all IaC is created equal — and [[Crossplane]] represents a fundamental shift in how we think about it.

## The Evolution

### Generation 1: Scripts (Bash, PowerShell)
```bash
# Imperative: "do these steps in order"
aws s3 mb s3://my-bucket
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration ...
```
**Problems**: Not idempotent, no state tracking, breaks if run twice.

### Generation 2: Declarative Tools (Terraform, CloudFormation)
```hcl
# Declarative: "this is what I want"
resource "aws_s3_bucket" "data" {
  bucket = "my-bucket"
}
```
**Better**: State files track what exists. Plan before apply. But state is external (S3 backend, state locking).

### Generation 3: Infrastructure as Data (Crossplane)
```yaml
# Kubernetes-native: "this is a Kubernetes resource"
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-bucket
spec:
  forProvider:
    region: us-east-1
```
**Best**: No external state — Kubernetes IS the state store. Continuous reconciliation. Same API as your applications.

## Key Concepts

### Declarative vs. Imperative

| Approach | You specify | Tool does | Example |
|----------|------------|-----------|---------|
| **Imperative** | Steps to execute | Runs them in order | Bash scripts, Ansible playbooks |
| **Declarative** | Desired end state | Figures out the steps | Terraform, Crossplane, Kubernetes |

Declarative is almost always better for infrastructure because:
- It's **idempotent** — applying the same config twice has no additional effect
- It enables **drift detection** — compare desired vs. actual state (see [[Drift Detection]])
- It's **reviewable** — you can diff the desired state in Git

### State Management

Every IaC tool needs to know "what exists" to decide "what to do":

| Tool | State Storage | Locking | Refresh |
|------|--------------|---------|---------|
| **Terraform** | S3 + DynamoDB, Terraform Cloud, local file | Explicit (DynamoDB) | `terraform refresh` |
| **Pulumi** | Pulumi Cloud, S3, local | Built-in | `pulumi refresh` |
| **Crossplane** | Kubernetes etcd (built-in!) | Kubernetes API server | Continuous (~30s [[Reconciliation Loop]]) |

Crossplane's advantage: **no state file management**. The Kubernetes API server IS the state store. No S3 backends, no state locking bugs, no "state is out of sync" emergencies.

### Idempotency

An operation is idempotent if running it multiple times produces the same result as running it once.

```bash
# ❌ NOT idempotent — fails on second run
aws s3 mb s3://my-bucket
# Error: BucketAlreadyOwnedByYou

# ✅ Idempotent — Crossplane checks if it exists first
kubectl apply -f bucket.yaml
# bucket.s3.aws.upbound.io/my-bucket configured (or unchanged)
```

### Drift

**Drift** is when actual infrastructure state diverges from the declared desired state. Someone logs into the AWS console and changes a setting manually — that's drift.

| Tool | Drift Handling |
|------|---------------|
| **Terraform** | Detect with `terraform plan`, fix with `terraform apply` (manual) |
| **Crossplane** | Detect AND fix automatically every ~30 seconds |

See [[Drift Detection]] for a deep dive on self-healing.

## Why Crossplane for FinEdge?

FinEdge chose Crossplane over Terraform for several reasons:

### 1. Kubernetes-Native
Teams already know `kubectl`. No new CLI to learn, no new workflow. Infrastructure is just another Kubernetes resource:
```bash
kubectl get buckets          # See your S3 buckets
kubectl describe bucket X    # See its status
kubectl delete bucket X      # Delete it
```

### 2. Continuous Reconciliation
Terraform runs when you tell it to (`terraform apply`). Crossplane runs continuously — if something drifts, it fixes it. This is the [[Reconciliation Loop]] pattern.

### 3. RBAC Integration
Kubernetes [[Kubernetes RBAC|RBAC]] works on infrastructure resources too. A developer with access to the `payments` namespace can create [[Claims]] there but nowhere else. No IAM credentials to manage.

### 4. Composability
Crossplane [[Compositions]] let you build higher-level abstractions. Instead of raw S3 + SQS + RDS, teams get a single `AppEnv` that bundles everything they need.

### 5. GitOps Compatible
Since everything is a Kubernetes manifest, [[ArgoCD]] can sync infrastructure the same way it syncs applications. One workflow for everything.

## The FinEdge IaC Flow

```
Developer writes Claim (YAML)
        │
        ▼
Git repository (source of truth)
        │
        ▼
ArgoCD syncs to cluster ([[GitOps]])
        │
        ▼
Crossplane reads Claim, matches to Composition
        │
        ▼
Composition creates Managed Resources
        │
        ▼
Provider calls LocalStack/AWS APIs
        │
        ▼
Resources exist! Status propagates back up.
```

## Common Pitfalls

1. **Mixing imperative and declarative** — Don't `kubectl create` resources that should be in Git
2. **State drift** — If using Terraform, always run `plan` before `apply`
3. **Hardcoding values** — Use variables/patches so the same template works for all teams
4. **Ignoring deletion** — Always set [[Deletion Policy]] explicitly (especially for databases!)
5. **No testing** — Use `crossplane render` or `terraform plan` to validate before applying

## Related

- [[Crossplane]]
- [[Reconciliation Loop]]
- [[Drift Detection]]
- [[GitOps]]
- [[Managed Resources]]
