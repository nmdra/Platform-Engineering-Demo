# infra/ — Team Infrastructure Claims

This directory contains **AppEnv claims** — the YAML files product teams submit to request infrastructure. Each team has its own subdirectory.

## Structure

```
infra/
├── payments/
│   └── claim.yaml       # S3 (encrypted) + SQS (60s visibility) + DLQ
├── analytics/
│   └── claim.yaml       # S3 (versioned) + RDS PostgreSQL (50GB)
└── notifications/
    └── claim.yaml       # S3 + SQS + SNS topic
```

## How It Works

1. A team developer creates or modifies a `claim.yaml`
2. They open a Pull Request
3. CI validates the claim against Kyverno policies
4. After merge, ArgoCD syncs the claim to the cluster
5. Crossplane reconciles and provisions the AWS resources in LocalStack

## Claim Template

```yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: <team>-<env>
  namespace: <team>
  annotations:
    platform.finedge.io/owner: "<email>@finedge.io"    # REQUIRED
    platform.finedge.io/ticket: "INFRA-<number>"       # REQUIRED
spec:
  parameters:
    teamName: <team>          # payments | analytics | notifications
    environment: <env>        # dev | staging | prod
    storage:
      enabled: true/false
      encryption: true        # Must be true (PCI-DSS)
      versioning: true/false
    queue:
      enabled: true/false
      visibilityTimeoutSeconds: 30
      retentionDays: 4
      deadLetterEnabled: true
    database:
      enabled: true/false
      engine: postgres        # postgres | mysql
      instanceClass: db.t3.micro
      storageGb: 20
    notifications:
      snsEnabled: true/false
      topicType: standard     # standard | fifo
```

## Rules

- Every claim **must** have `platform.finedge.io/owner` and `platform.finedge.io/ticket` annotations
- Storage **must** have `encryption: true` (enforced by Kyverno)
- Production databases **cannot** use `db.t3.micro` (minimum: `db.t3.small`)
- Production databases **must** have `storageGb >= 50`
