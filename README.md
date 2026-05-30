# FinEdge Platform Engineering Demo

> **Self-service infrastructure for product teams — powered by Crossplane, LocalStack, ArgoCD, and Kyverno.**

[![Crossplane](https://img.shields.io/badge/Crossplane-v2.3-blue?logo=crossplane)](https://crossplane.io)
[![LocalStack](https://img.shields.io/badge/LocalStack-2026.x-orange)](https://localstack.cloud)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-v3.4-red?logo=argo)](https://argoproj.github.io/cd)
[![Kyverno](https://img.shields.io/badge/Kyverno-v1.18-green?logo=kyverno)](https://kyverno.io)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

---

## What Is This?

You are the **Platform Engineer** at *FinEdge*, a fintech startup. Three product teams — **Payments**, **Analytics**, and **Notifications** — need AWS infrastructure on demand. They shouldn't touch Terraform, the AWS console, or write IAM policies themselves.

This project builds a **self-service Internal Developer Platform (IDP)** that lets teams request infrastructure with a 20-line YAML claim while the platform team controls all the complexity behind the scenes.

### The Developer Experience

```yaml
# A team developer writes this...
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: payments-dev
  namespace: payments
  annotations:
    platform.finedge.io/owner: "payments-team@finedge.io"
    platform.finedge.io/ticket: "INFRA-201"
spec:
  parameters:
    teamName: payments
    environment: dev
    storage:
      enabled: true
      encryption: true
    queue:
      enabled: true
      visibilityTimeoutSeconds: 60
```

```
# ...and gets all of this automatically:
✓ S3 bucket (encrypted, tagged, named correctly)
✓ SQS queue (with dead-letter queue wired up)
✓ Proper IAM tags for cost tracking
✓ Drift protection (delete it, it comes back)
✓ Policy compliance (encryption enforced, owner required)
```

---

## Architecture

```
 Your Laptop
 ┌──────────────────────────────────────────────────────────────────┐
 │                                                                  │
 │  LocalStack (2026.x)          ◄──── Fake AWS on localhost:4566  │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  kind cluster (finedge-platform)                          │   │
 │  │                                                           │   │
 │  │  Crossplane v2.3 ──► Composition Functions ──► LocalStack │   │
 │  │  ArgoCD v3.4     ──► Git sync (claims)                    │   │
 │  │  Kyverno v1.18   ──► Policy enforcement                  │   │
 │  │  Prometheus       ──► Grafana dashboards                  │   │
 │  │                                                           │   │
 │  │  Teams: payments / analytics / notifications              │   │
 │  │    (RBAC · NetworkPolicy · ResourceQuota · PSA)           │   │
 │  └──────────────────────────────────────────────────────────┘   │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Tool | Minimum Version | Install |
|------|----------------|---------|
| kind | >= 0.31 | `brew install kind` |
| kubectl | latest | `brew install kubectl` |
| Helm | >= 3.14 | `brew install helm` |
| LocalStack | >= 2026.01 | `pip install localstack` |
| AWS CLI | >= 2.0 | `brew install awscli` |

> **⚠️ LocalStack Auth Token Required:** Since March 2026, LocalStack requires an auth token for all users (including the free Hobby plan). [Sign up here](https://app.localstack.cloud) and run `localstack auth set-token <your-token>`.

---

## Quick Start

```bash
# 1. Clone and enter the project
git clone https://github.com/your-org/finedge-platform.git
cd finedge-platform

# 2. Start LocalStack
localstack start -d

# 3. Create the kind cluster
kind create cluster --config kind/cluster.yaml

# 4. Run the bootstrap script
./scripts/bootstrap.sh

# 5. Submit a team claim
kubectl apply -f infra/payments/claim.yaml

# 6. Watch it provision
kubectl get appenv -A -w
```

For detailed step-by-step instructions, see [Docs/initial_plan.md](Docs/initial_plan.md).

---

## Project Structure

```
finedge-platform/
├── Docs/                        # 📖 Project documentation
│   └── initial_plan.md          #    Full implementation guide (8 phases)
│
├── kind/                        # 🏗️  Cluster configuration
│   └── cluster.yaml             #    kind multi-node cluster spec
│
├── crossplane/                  # ⚙️  Platform API definitions
│   ├── functions.yaml           #    Composition Functions (P&T + auto-ready)
│   ├── providers.yaml           #    AWS provider family packages
│   ├── providerconfig-*.yaml    #    LocalStack / real AWS wiring
│   ├── xrd-appenv.yaml          #    CompositeResourceDefinition (the API)
│   └── composition-appenv.yaml  #    Composition (the implementation)
│
├── infra/                       # 📋 Team infrastructure claims
│   ├── payments/                #    Payments team claim
│   ├── analytics/               #    Analytics team claim
│   └── notifications/           #    Notifications team claim
│
├── security/                    # 🔒 Security hardening
│   ├── rbac/                    #    Role & RoleBinding definitions
│   ├── network-policies/        #    Namespace isolation policies
│   └── resource-quotas/         #    Per-team resource limits
│
├── policies/                    # 🛡️  Kyverno guardrail policies
│   ├── require-owner-annotation.yaml
│   ├── restrict-prod-instance.yaml
│   └── block-public-buckets.yaml
│
├── gitops/                      # 🔄 ArgoCD application definitions
│   ├── argocd-apps.yaml         #    Root app-of-apps
│   └── apps/                    #    Per-team ArgoCD Applications
│
├── observability/               # 📊 Monitoring & alerting
│   ├── prometheusrule-crossplane.yaml
│   └── dashboard-crossplane.json
│
├── tests/                       # ✅ Validation & testing
│   ├── compositions/            #    Composition render tests
│   └── policies/                #    Kyverno policy test fixtures
│
├── scripts/                     # 🛠️  Automation scripts
│
├── wiki/                        # 📚 Obsidian-style knowledge base
│   ├── concepts/                #    Platform engineering concepts
│   ├── tools/                   #    Tool-specific guides
│   ├── patterns/                #    Architecture patterns
│   ├── troubleshooting/         #    Issue resolution guides
│   └── journal/                 #    Daily learning notes
│
├── .agents/                     # 🤖 AI agent instructions
│   └── agent.md                 #    Agent rules and context
└── README.md                    # 📄 This file
```

---

## Phases

| # | Phase | Description | Status |
|---|-------|-------------|--------|
| 1 | **Bootstrap** | kind + LocalStack + Crossplane v2.3 + Functions | 🔲 |
| 2 | **Platform API** | XRD + Composition with auto-ready & deletionPolicy | 🔲 |
| 3 | **Security + Claims** | RBAC, NetworkPolicy, Quotas, PSA + team claims | 🔲 |
| 4 | **GitOps** | ArgoCD app-of-apps with safe prune settings | 🔲 |
| 5 | **Guardrails** | Kyverno policies (owner, encryption, prod sizing) | 🔲 |
| 6 | **Observability** | Prometheus rules + Grafana + drift demo | 🔲 |
| 7 | **Testing** | Composition render tests + policy validation + CI | 🔲 |
| 8 | **Graduation** | One-label flip from LocalStack to real AWS | 🔲 |

---

## Key Concepts

### Who Owns What?

| Role | Owns | Touches |
|------|------|---------|
| **Platform Team** | Providers, ProviderConfigs, Functions, XRDs, Compositions, Policies, RBAC, Quotas | `crossplane/`, `policies/`, `security/` |
| **Product Teams** | Claims (~20 lines of YAML) | `infra/<team>/claim.yaml` |
| **Git** | Everything — every infra change is a PR | All directories |

### Why Crossplane v2?

- **Namespaced resources** — better multi-tenancy
- **Usages GA** — dependency ordering without flags
- **Broader composition** — compose any K8s resource, not just infra
- **Pipeline-only mode** — Composition Functions are the standard

### Why LocalStack?

- **No AWS account needed** — develop and demo locally
- **Fast iteration** — resources provision in seconds, not minutes
- **Same API** — Crossplane talks to the same AWS APIs
- **Graduation path** — one label change moves to real AWS

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-change`)
3. Run validation (`./scripts/validate.sh`)
4. Commit and push
5. Open a Pull Request

See [.agents/agent.md](.agents/agent.md) for AI agent contribution guidelines.
See [wiki/](./wiki/_index.md) for the learning knowledge base.

---

## License

MIT — see [LICENSE](LICENSE) for details.
