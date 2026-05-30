# FinEdge Platform Engineering — Crossplane + LocalStack

> **Scenario:** You are the Platform Engineer at *FinEdge*, a fintech startup.
> Three product teams — **Payments**, **Analytics**, and **Notifications** — need
> AWS infrastructure on demand. They shouldn't touch Terraform, the AWS console,
> or write IAM policies themselves.
>
> You will build a **self-service Internal Developer Platform** across seven phases,
> starting from a raw laptop and ending with ArgoCD-driven GitOps, Kyverno guardrails,
> observability, security hardening, and a one-label graduation path to real AWS.

---

## Project Structure

```
finedge-platform/
├── kind/
│   └── cluster.yaml                  # kind cluster config
├── crossplane/
│   ├── functions.yaml                # Composition Functions
│   ├── providers.yaml                # AWS provider family
│   ├── providerconfig-localstack.yaml
│   ├── xrd-appenv.yaml               # CompositeResourceDefinition
│   └── composition-appenv.yaml       # Composition (platform logic)
├── infra/
│   ├── payments/
│   │   └── claim.yaml
│   ├── analytics/
│   │   └── claim.yaml
│   └── notifications/
│       └── claim.yaml
├── security/
│   ├── rbac/
│   │   ├── platform-admin-role.yaml
│   │   ├── team-developer-role.yaml
│   │   └── team-rolebindings.yaml
│   ├── network-policies/
│   │   ├── default-deny.yaml
│   │   └── allow-crossplane.yaml
│   └── resource-quotas/
│       └── team-quotas.yaml
├── policies/
│   ├── require-owner-annotation.yaml
│   ├── restrict-prod-instance.yaml
│   └── block-public-buckets.yaml
├── gitops/
│   └── argocd-apps.yaml
├── observability/
│   ├── prometheusrule-crossplane.yaml
│   └── dashboard-crossplane.json
└── tests/
    ├── compositions/
    │   └── test-appenv-render.yaml
    └── policies/
        ├── test-valid-claim.yaml
        └── test-invalid-claim.yaml
```

---

## Architecture

```
 Your Laptop
 ┌──────────────────────────────────────────────────────────────────┐
 │                                                                  │
 │  LocalStack (localstack CLI — 2026.05.x, auth token required)   │
 │  Listening on 0.0.0.0:4566                                       │
 │  Services: s3, sqs, sns, iam, rds, ssm, secretsmanager          │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  kind cluster (finedge-platform)                          │   │
 │  │                                                           │   │
 │  │  crossplane-system/                                       │   │
 │  │    Crossplane v2.3 + AWS Provider Family                  │   │
 │  │    Composition Functions (P&T + auto-ready)               │   │
 │  │    ClusterProviderConfig → host.docker.internal:4566     │──┼──▶ LocalStack
 │  │                                                           │   │
 │  │  argocd/         ArgoCD v3.4 watches Git, applies claims  │   │
 │  │  payments/       AppEnv claim (storage + queue)           │   │
 │  │  analytics/      AppEnv claim (storage + db)              │   │
 │  │  notifications/  AppEnv claim (queue + sns)               │   │
 │  │  monitoring/     Prometheus + Grafana                     │   │
 │  │  kyverno/        ValidatingPolicy enforcement (CEL)       │   │
 │  │                                                           │   │
 │  │  Security: RBAC per namespace · NetworkPolicies ·         │   │
 │  │    ResourceQuotas · Pod Security Admission                │   │
 │  └──────────────────────────────────────────────────────────┘   │
 └──────────────────────────────────────────────────────────────────┘
```

> **Why `host.docker.internal`?**
> kind nodes are Docker containers. To reach LocalStack running on the host machine,
> containers use `host.docker.internal` — Docker's built-in DNS alias for the host.

---

## Prerequisites

```bash
# Install once
brew install kind kubectl helm awscli                             # macOS
pip install localstack                                            # cross-platform

# Verify
kind version          # >= 0.31
kubectl version --client
helm version          # >= 3.14
localstack --version  # >= 2026.01  (calendar versioning: YYYY.MM.patch)
aws --version         # >= 2.0
```

> **⚠️ LocalStack Auth Token (Required since March 2026)**
>
> LocalStack now requires an authentication token for all users, including the free Hobby plan.
> 1. Sign up at [localstack.cloud](https://app.localstack.cloud)
> 2. Generate a token under Account → Auth Tokens
> 3. Configure it:
>    ```bash
>    localstack auth set-token <your-token>
>    # OR export LOCALSTACK_AUTH_TOKEN=<your-token>
>    ```

---

## Phase 1 — Local Environment Bootstrap

**Goal:** kind cluster running, LocalStack live on the host, Crossplane v2.3 installed
with Composition Functions, talking to LocalStack. No real AWS account needed.

### 1.1 Start LocalStack on the host

```bash
# Start LocalStack (all services start by default in 2026.x — no SERVICES= needed)
localstack start -d

# Verify it's up
curl -s http://localhost:4566/_localstack/health | jq '.services | to_entries[] | select(.value=="running")'
```

Configure a local AWS CLI profile pointing to LocalStack:

```bash
aws configure set aws_access_key_id     test     --profile localstack
aws configure set aws_secret_access_key test     --profile localstack
aws configure set region                us-east-1 --profile localstack
aws configure set output                json     --profile localstack

# Quick smoke test
aws s3 ls --profile localstack --endpoint-url http://localhost:4566
# (empty — no buckets yet. No error = success)
```

### 1.2 Create the kind cluster

```yaml
# kind/cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: finedge-platform
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
  - role: worker
    labels:
      workload: platform
  - role: worker
    labels:
      workload: apps
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

```bash
kind create cluster --config kind/cluster.yaml

kubectl get nodes
# NAME                             STATUS   ROLES           AGE
# finedge-platform-control-plane   Ready    control-plane   60s
# finedge-platform-worker          Ready    <none>          45s
# finedge-platform-worker2         Ready    <none>          45s
```

### 1.3 Install Crossplane v2.3

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 2.3.1 \
  --wait

# Note: --enable-usages is no longer needed in v2.x (Usages are GA by default)

kubectl get pods -n crossplane-system
# crossplane-xxxxx              1/1   Running
# crossplane-rbac-manager-xxxx  1/1   Running
```

> **Migration Note:** Crossplane jumped from v1.x to v2.0 in August 2025.
> Key changes in v2:
> - **Namespaced resources** — XRs and MRs are namespaced by default
> - **Usages GA** — no flag needed
> - **Native P&T removed** — must use `function-patch-and-transform` (we already do)
> - **Broader Composition** — can compose ANY K8s resource (Deployments, ConfigMaps, etc.)
> - **ControllerConfig removed** — use DeploymentRuntimeConfig instead

### 1.4 Install Composition Functions

Crossplane v2 requires Composition Functions for all composition logic. Install the
two essential functions:

```yaml
# crossplane/functions.yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.9.0
---
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.4.0
```

```bash
kubectl apply -f crossplane/functions.yaml

# Wait for functions to become healthy
kubectl get functions -w
# NAME                             INSTALLED   HEALTHY
# function-patch-and-transform     True        True
# function-auto-ready              True        True
```

> **What is `function-auto-ready`?**
> It automatically aggregates readiness from all composed managed resources and
> propagates it to the Composite Resource (XR). Without it, your XR may show
> `Ready: True` even when child resources (like RDS) are still provisioning.

### 1.5 Install AWS Provider Family

```yaml
# crossplane/providers.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-family-aws
spec:
  package: xpkg.upbound.io/upbound/provider-family-aws:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-sqs
spec:
  package: xpkg.upbound.io/upbound/provider-aws-sqs:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-rds
spec:
  package: xpkg.upbound.io/upbound/provider-aws-rds:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-iam
spec:
  package: xpkg.upbound.io/upbound/provider-aws-iam:v1.17.0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-sns
spec:
  package: xpkg.upbound.io/upbound/provider-aws-sns:v1.17.0
```

```bash
kubectl apply -f crossplane/providers.yaml

# Wait — providers pull images; takes ~2 minutes
kubectl get providers -w
# NAME                           INSTALLED   HEALTHY
# upbound-provider-family-aws    True        True
# provider-aws-s3                True        True
# provider-aws-sqs               True        True
# provider-aws-rds               True        True
# provider-aws-iam               True        True
# provider-aws-sns               True        True
```

> **⚠️ Upbound Access Policy Note:**
> Since March 2025, free-tier users can only pull the **latest** version of official
> Upbound packages. Version pinning (e.g., `v1.17.0`) requires a paid subscription
> and pull secret. If you're on the free tier, omit the version tag or use the
> community builds from `crossplane-contrib`.

### 1.6 Wire Crossplane to LocalStack

```bash
# LocalStack accepts any non-empty credentials
kubectl create secret generic aws-localstack-creds \
  --namespace crossplane-system \
  --from-literal=credentials='[default]
aws_access_key_id = test
aws_secret_access_key = test
'
```

```yaml
# crossplane/providerconfig-localstack.yaml
#
# host.docker.internal resolves to your laptop from inside kind nodes.
# On Linux: add --add-host host.docker.internal:host-gateway to kind node extraArgs,
# or use your machine's LAN IP directly.
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: localstack
  annotations:
    platform.finedge.io/description: "Routes all AWS API calls to local LocalStack instance"
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-localstack-creds
      key: credentials
  endpoint:
    hostnameImmutable: true
    partitionOverride: aws
    signingRegion: us-east-1
    url:
      type: Static
      static: http://host.docker.internal:4566
    services:
      - id: S3
        url:
          type: Static
          static: http://host.docker.internal:4566
      - id: SQS
        url:
          type: Static
          static: http://host.docker.internal:4566
      - id: RDS
        url:
          type: Static
          static: http://host.docker.internal:4566
      - id: IAM
        url:
          type: Static
          static: http://host.docker.internal:4566
      - id: SNS
        url:
          type: Static
          static: http://host.docker.internal:4566
      - id: SSM
        url:
          type: Static
          static: http://host.docker.internal:4566
      - id: SecretsManager
        url:
          type: Static
          static: http://host.docker.internal:4566
```

> **Crossplane v2 Note — ProviderConfig Scoping:**
> In Crossplane v2, `ProviderConfig` can be either namespaced or cluster-scoped.
> For backward compatibility, the legacy `ProviderConfig` (shown above) still works
> as cluster-scoped. For explicit v2-style scoping, use `ClusterProviderConfig`.
> We keep `ProviderConfig` here since it's compatible with both v1 and v2 providers.

```bash
kubectl apply -f crossplane/providerconfig-localstack.yaml
```

**Verify the connection from inside the cluster:**

```bash
kubectl run connectivity-check --rm -it --restart=Never \
  --image=amazon/aws-cli \
  --env="AWS_ACCESS_KEY_ID=test" \
  --env="AWS_SECRET_ACCESS_KEY=test" \
  --env="AWS_DEFAULT_REGION=us-east-1" \
  -- s3 ls --endpoint-url http://host.docker.internal:4566
# (empty output, no error) = Phase 1 complete ✓
```

**Linux note:** `host.docker.internal` is not automatic on Linux kind. Run this once:

```bash
# Find your host IP used by Docker bridge
HOST_IP=$(docker inspect finedge-platform-control-plane \
  --format '{{.NetworkSettings.Gateway}}')

# Patch CoreDNS to resolve the alias
kubectl -n kube-system patch configmap coredns --type merge -p "{
  \"data\": {
    \"Corefile\": \".:53 {\n    hosts {\n        $HOST_IP host.docker.internal\n        fallthrough\n    }\n    forward . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n\"
  }
}"
kubectl rollout restart deployment coredns -n kube-system
```

---

## Phase 2 — Platform API: XRD + Composition

**Goal:** Define the `AppEnv` API that product teams use. Write the Composition
that maps simple claims to real AWS resources. No team touches AWS directly ever again.

### 2.1 CompositeResourceDefinition — the "form" teams fill out

```yaml
# crossplane/xrd-appenv.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xappenvs.platform.finedge.io
spec:
  group: platform.finedge.io
  names:
    kind: XAppEnv
    plural: xappenvs
  claimNames:
    kind: AppEnv          # namespaced — teams create this
    plural: appenvs
  defaultCompositionRef:
    name: finedge-appenv-localstack
  connectionSecretKeys:
    - bucketName
    - queueUrl
    - dbEndpoint
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [parameters]
              properties:
                parameters:
                  type: object
                  required: [teamName, environment]
                  properties:
                    teamName:
                      type: string
                      description: "Team slug: payments | analytics | notifications"
                      pattern: "^[a-z][a-z0-9-]{1,20}$"
                    environment:
                      type: string
                      enum: [dev, staging, prod]
                    region:
                      type: string
                      default: us-east-1
                    storage:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: false
                        versioning:
                          type: boolean
                          default: false
                        encryption:
                          type: boolean
                          default: true
                        lifecycleDays:
                          type: integer
                          default: 90
                          description: "Move objects to Glacier after N days"
                    queue:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: false
                        visibilityTimeoutSeconds:
                          type: integer
                          default: 30
                          minimum: 0
                          maximum: 43200
                        retentionDays:
                          type: integer
                          default: 4
                        deadLetterEnabled:
                          type: boolean
                          default: true
                        maxReceiveCount:
                          type: integer
                          default: 3
                    database:
                      type: object
                      properties:
                        enabled:
                          type: boolean
                          default: false
                        engine:
                          type: string
                          enum: [postgres, mysql]
                          default: postgres
                        engineVersion:
                          type: string
                          default: "14"
                        instanceClass:
                          type: string
                          default: db.t3.micro
                        storageGb:
                          type: integer
                          default: 20
                    notifications:
                      type: object
                      properties:
                        snsEnabled:
                          type: boolean
                          default: false
                        topicType:
                          type: string
                          enum: [standard, fifo]
                          default: standard
            status:
              type: object
              properties:
                bucketName:
                  type: string
                queueUrl:
                  type: string
                deadLetterQueueUrl:
                  type: string
                dbEndpoint:
                  type: string
                snsTopicArn:
                  type: string
```

```bash
kubectl apply -f crossplane/xrd-appenv.yaml

# Crossplane registers two new CRDs
kubectl get crd | grep finedge
# appenvs.platform.finedge.io          (namespaced — for teams)
# xappenvs.platform.finedge.io         (cluster-scoped — internal)
```

### 2.2 Composition — the platform team's infrastructure logic

This is the only file product teams never touch. All tagging, naming conventions,
DLQ wiring, encryption decisions, and deletion policies live here.

**Key improvements over a naive composition:**
- **`deletionPolicy`** explicitly set on every resource (Delete for dev, Orphan for prod databases)
- **`function-auto-ready`** pipeline step aggregates child resource readiness
- **`publishConnectionDetailsTo`** for modern secret management
- **Consistent tagging** with `ManagedBy`, `Platform`, `Team`, `Environment` labels

```yaml
# crossplane/composition-appenv.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: finedge-appenv-localstack
  labels:
    provider: localstack
    channel: dev
spec:
  compositeTypeRef:
    apiVersion: platform.finedge.io/v1alpha1
    kind: XAppEnv
  mode: Pipeline
  pipeline:
    # ── Step 1: Patch and Transform — create resources ─────────────────
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:

          # ── S3 Data Bucket ─────────────────────────────────────────────────
          - name: data-bucket
            base:
              apiVersion: s3.aws.upbound.io/v1beta1
              kind: Bucket
              metadata:
                labels:
                  platform.finedge.io/resource: storage
              spec:
                deletionPolicy: Delete    # Safe for dev/LocalStack
                forProvider:
                  region: us-east-1
                  tags:
                    ManagedBy: crossplane
                    Platform: finedge
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-data"
                toFieldPath: metadata.name
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.teamName
                toFieldPath: spec.forProvider.tags.Team
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.environment
                toFieldPath: spec.forProvider.tags.Environment
              - type: ToCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: status.bucketName

          # ── S3 Versioning ─────────────────────────────────────────────────
          - name: bucket-versioning
            base:
              apiVersion: s3.aws.upbound.io/v1beta1
              kind: BucketVersioning
              spec:
                deletionPolicy: Delete
                forProvider:
                  region: us-east-1
                  versioningConfiguration:
                    - status: Enabled
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-data"
                toFieldPath: spec.forProvider.bucket

          # ── S3 Server-Side Encryption ──────────────────────────────────────
          - name: bucket-encryption
            base:
              apiVersion: s3.aws.upbound.io/v1beta1
              kind: BucketServerSideEncryptionConfiguration
              spec:
                deletionPolicy: Delete
                forProvider:
                  region: us-east-1
                  rule:
                    - applyServerSideEncryptionByDefault:
                        - sseAlgorithm: AES256
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-data"
                toFieldPath: spec.forProvider.bucket

          # ── SQS Dead Letter Queue ─────────────────────────────────────────
          - name: dead-letter-queue
            base:
              apiVersion: sqs.aws.upbound.io/v1beta1
              kind: Queue
              metadata:
                labels:
                  platform.finedge.io/resource: dlq
              spec:
                deletionPolicy: Delete
                forProvider:
                  region: us-east-1
                  messageRetentionSeconds: 1209600   # 14 days for DLQ
                  tags:
                    ManagedBy: crossplane
                    Role: dead-letter
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-events-dlq"
                toFieldPath: metadata.name
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.url
                toFieldPath: status.deadLetterQueueUrl
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.arn
                toFieldPath: metadata.annotations[platform.finedge.io/dlq-arn]

          # ── SQS Main Queue ────────────────────────────────────────────────
          - name: main-queue
            base:
              apiVersion: sqs.aws.upbound.io/v1beta1
              kind: Queue
              metadata:
                labels:
                  platform.finedge.io/resource: queue
              spec:
                deletionPolicy: Delete
                forProvider:
                  region: us-east-1
                  tags:
                    ManagedBy: crossplane
                    Role: primary
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-events"
                toFieldPath: metadata.name
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.queue.visibilityTimeoutSeconds
                toFieldPath: spec.forProvider.visibilityTimeoutSeconds
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.queue.retentionDays
                toFieldPath: spec.forProvider.messageRetentionSeconds
                transforms:
                  - type: math
                    math:
                      multiply: 86400
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.url
                toFieldPath: status.queueUrl

          # ── RDS Instance ──────────────────────────────────────────────────
          - name: rds-instance
            base:
              apiVersion: rds.aws.upbound.io/v1beta1
              kind: Instance
              spec:
                deletionPolicy: Delete    # Use Orphan for prod composition
                forProvider:
                  region: us-east-1
                  skipFinalSnapshot: true
                  publiclyAccessible: false
                  storageEncrypted: true
                  multiAz: false
                  autoMinorVersionUpgrade: true
                  deletionProtection: false  # Set true in prod composition
                  username: platformadmin
                  passwordSecretRef:
                    namespace: crossplane-system
                    name: rds-master-password
                    key: password
                  tags:
                    ManagedBy: crossplane
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-db"
                toFieldPath: metadata.name
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.database.engine
                toFieldPath: spec.forProvider.engine
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.database.engineVersion
                toFieldPath: spec.forProvider.engineVersion
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.database.instanceClass
                toFieldPath: spec.forProvider.instanceClass
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.database.storageGb
                toFieldPath: spec.forProvider.allocatedStorage
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.endpoint
                toFieldPath: status.dbEndpoint

          # ── SNS Topic ─────────────────────────────────────────────────────
          - name: sns-topic
            base:
              apiVersion: sns.aws.upbound.io/v1beta1
              kind: Topic
              spec:
                deletionPolicy: Delete
                forProvider:
                  region: us-east-1
                  tags:
                    ManagedBy: crossplane
                providerConfigRef:
                  name: localstack
            patches:
              - type: CombineFromComposite
                combine:
                  variables:
                    - fromFieldPath: spec.parameters.teamName
                    - fromFieldPath: spec.parameters.environment
                  strategy: string
                  string:
                    fmt: "finedge-%s-%s-notify"
                toFieldPath: metadata.name
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.arn
                toFieldPath: status.snsTopicArn

    # ── Step 2: Auto-ready — aggregate readiness from all composed resources ──
    - step: auto-ready
      functionRef:
        name: function-auto-ready
```

```bash
kubectl apply -f crossplane/composition-appenv.yaml
```

> **Why `function-auto-ready`?**
> Without this step, the XR's `Ready` condition could be `True` while child resources
> (especially RDS, which takes 5-10+ minutes) are still provisioning. `function-auto-ready`
> checks that all composed resources have `status.conditions[?type=="Ready"].status == "True"`
> before marking the XR itself as ready.

---

## Phase 3 — Security Hardening & Team Claims

**Goal:** Set up RBAC, network policies, and resource quotas *before* teams start
submitting claims. Then each team submits a minimal YAML that declares *what* they
need, not *how*. The Composition figures out the rest.

### 3.1 Team Namespaces with Security

```bash
# Create namespaces with Pod Security Admission labels
for ns in payments analytics notifications; do
  kubectl create namespace $ns
  kubectl label namespace $ns \
    pod-security.kubernetes.io/enforce=baseline \
    pod-security.kubernetes.io/warn=restricted \
    pod-security.kubernetes.io/audit=restricted
done
```

### 3.2 RBAC — Least-Privilege Access

```yaml
# security/rbac/platform-admin-role.yaml
#
# Platform admins can manage XRDs, Compositions, ProviderConfigs, and Policies.
# They should NOT need cluster-admin.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: finedge-platform-admin
rules:
  - apiGroups: ["apiextensions.crossplane.io"]
    resources: ["compositeresourcedefinitions", "compositions"]
    verbs: ["*"]
  - apiGroups: ["aws.upbound.io"]
    resources: ["providerconfigs"]
    verbs: ["*"]
  - apiGroups: ["pkg.crossplane.io"]
    resources: ["providers", "functions", "configurations"]
    verbs: ["*"]
  - apiGroups: ["kyverno.io"]
    resources: ["validatingpolicies", "clusterpolicies"]
    verbs: ["*"]
  - apiGroups: ["platform.finedge.io"]
    resources: ["xappenvs"]
    verbs: ["*"]
---
# security/rbac/team-developer-role.yaml
#
# Team developers can ONLY create/read/update/delete AppEnv claims in their namespace.
# They cannot see other teams' claims, XRDs, Compositions, or ProviderConfigs.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: finedge-team-developer
rules:
  - apiGroups: ["platform.finedge.io"]
    resources: ["appenvs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["platform.finedge.io"]
    resources: ["appenvs/status"]
    verbs: ["get", "list", "watch"]
```

```yaml
# security/rbac/team-rolebindings.yaml
#
# Bind the team-developer role to each team's namespace.
# Replace group names with your identity provider's groups.
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-developers
  namespace: payments
subjects:
  - kind: Group
    name: finedge-payments-team    # From your IdP (OIDC, LDAP, etc.)
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: finedge-team-developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: analytics-developers
  namespace: analytics
subjects:
  - kind: Group
    name: finedge-analytics-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: finedge-team-developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: notifications-developers
  namespace: notifications
subjects:
  - kind: Group
    name: finedge-notifications-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: finedge-team-developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f security/rbac/
```

### 3.3 Network Policies — Namespace Isolation

```yaml
# security/network-policies/default-deny.yaml
#
# Default deny all ingress and egress in team namespaces.
# Only explicitly allowed traffic can flow.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  # Apply to each team namespace
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# security/network-policies/allow-crossplane.yaml
#
# Allow crossplane-system pods to reach LocalStack and the K8s API server.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-crossplane-egress
  namespace: crossplane-system
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - {}    # Crossplane needs outbound to cloud APIs and the K8s API server
```

```bash
# Apply default-deny to all team namespaces
for ns in payments analytics notifications; do
  kubectl apply -f security/network-policies/default-deny.yaml -n $ns
done
kubectl apply -f security/network-policies/allow-crossplane.yaml
```

### 3.4 Resource Quotas — Prevent Runaway Provisioning

```yaml
# security/resource-quotas/team-quotas.yaml
#
# Limit the number of AppEnv claims per namespace.
# Prevents a single team from exhausting platform resources.
apiVersion: v1
kind: ResourceQuota
metadata:
  name: appenv-quota
  # Apply per namespace
spec:
  hard:
    count/appenvs.platform.finedge.io: "5"    # Max 5 AppEnv claims per team
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
```

```bash
for ns in payments analytics notifications; do
  kubectl apply -f security/resource-quotas/team-quotas.yaml -n $ns
done
```

### 3.5 Team Claims

#### Payments Team

Needs encrypted storage and a queue with longer visibility timeout for payment processing.

```yaml
# infra/payments/claim.yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: payments-dev
  namespace: payments
  annotations:
    platform.finedge.io/owner:  "payments-team@finedge.io"
    platform.finedge.io/ticket: "INFRA-201"
    platform.finedge.io/docs:   "https://wiki.finedge.io/payments/infra"
spec:
  parameters:
    teamName:    payments
    environment: dev
    storage:
      enabled:    true
      encryption: true      # PCI-DSS requirement
      versioning: false
    queue:
      enabled:                  true
      visibilityTimeoutSeconds: 60   # payment jobs can take up to 60s
      retentionDays:            7
      deadLetterEnabled:        true
      maxReceiveCount:          5
    database:
      enabled: false             # payments uses an external managed DB
    notifications:
      snsEnabled: false
```

#### Analytics Team

Needs versioned S3 for audit trails and a PostgreSQL database for query results.

```yaml
# infra/analytics/claim.yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: analytics-dev
  namespace: analytics
  annotations:
    platform.finedge.io/owner:  "analytics-team@finedge.io"
    platform.finedge.io/ticket: "INFRA-202"
spec:
  parameters:
    teamName:    analytics
    environment: dev
    storage:
      enabled:      true
      versioning:   true    # full history for data audits
      encryption:   true
      lifecycleDays: 365    # keep data for one year before archiving
    queue:
      enabled: false
    database:
      enabled:        true
      engine:         postgres
      engineVersion:  "14"
      instanceClass:  db.t3.small
      storageGb:      50
    notifications:
      snsEnabled: false
```

#### Notifications Team

Needs a queue for outbound message jobs and an SNS topic for fan-out to email/SMS.

```yaml
# infra/notifications/claim.yaml
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: notifications-dev
  namespace: notifications
  annotations:
    platform.finedge.io/owner:  "notifications-team@finedge.io"
    platform.finedge.io/ticket: "INFRA-203"
spec:
  parameters:
    teamName:    notifications
    environment: dev
    storage:
      enabled:    true      # template storage
      versioning: false
    queue:
      enabled:                  true
      visibilityTimeoutSeconds: 30
      retentionDays:            4
      deadLetterEnabled:        true
    database:
      enabled: false
    notifications:
      snsEnabled: true
      topicType:  standard
```

### 3.6 Apply all claims

```bash
kubectl apply -f infra/payments/claim.yaml
kubectl apply -f infra/analytics/claim.yaml
kubectl apply -f infra/notifications/claim.yaml
```

### 3.7 Watch everything provision

```bash
# All claims at once
kubectl get appenv -A
# NAMESPACE      NAME                SYNCED   READY
# payments       payments-dev        True     True
# analytics      analytics-dev       True     False  ← RDS still creating
# notifications  notifications-dev   True     True

# See every managed AWS resource in the cluster
kubectl get managed
# NAME                                              READY   SYNCED
# bucket/finedge-payments-dev-data                  True    True
# bucket/finedge-analytics-dev-data                 True    True
# bucket/finedge-notifications-dev-data             True    True
# bucketversioning/finedge-analytics-dev-data       True    True
# bucketencryption/finedge-payments-dev-data        True    True
# queue/finedge-payments-dev-events-dlq             True    True
# queue/finedge-payments-dev-events                 True    True
# queue/finedge-notifications-dev-events-dlq        True    True
# queue/finedge-notifications-dev-events            True    True
# instance/finedge-analytics-dev-db                 False   True
# topic/finedge-notifications-dev-notify            True    True

# Full resource tree for one claim
kubectl crossplane beta trace appenv payments-dev -n payments
```

### 3.8 Verify in LocalStack

```bash
export AWS_PROFILE=localstack
export AWS_ENDPOINT_URL=http://localhost:4566

# S3 buckets
aws s3 ls
# finedge-payments-dev-data
# finedge-analytics-dev-data
# finedge-notifications-dev-data

# Versioning on analytics bucket
aws s3api get-bucket-versioning --bucket finedge-analytics-dev-data
# { "Status": "Enabled" }

# SQS queues
aws sqs list-queues --query 'QueueUrls[]'
# finedge-payments-dev-events-dlq
# finedge-payments-dev-events
# finedge-notifications-dev-events-dlq
# finedge-notifications-dev-events

# SNS topics
aws sns list-topics --query 'Topics[*].TopicArn'
# arn:aws:sns:us-east-1:000000000000:finedge-notifications-dev-notify

# RDS
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus]'
# [["finedge-analytics-dev-db", "available"]]
```

---

## Phase 4 — GitOps with ArgoCD

**Goal:** Stop applying YAMLs manually. Teams commit claims to Git and ArgoCD
continuously reconciles the cluster to match. Infrastructure changes go through
pull requests with approval gates.

### 4.1 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

# Port-forward the UI
kubectl port-forward svc/argocd-server 8080:443 -n argocd &
ARGOCD_PASSWORD=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD password: $ARGOCD_PASSWORD"
# Open https://localhost:8080  user: admin
```

### 4.2 ArgoCD App-of-Apps

```yaml
# gitops/argocd-apps.yaml
#
# One root Application that manages all team infra claims as child Applications.
# Changing a claim.yaml in Git automatically updates the cluster within 3 minutes.

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finedge-infra-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/finedge/platform-infra
    targetRevision: main
    path: gitops/apps           # folder containing child Application YAMLs
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
# gitops/apps/payments.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-infra
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/finedge/platform-infra
    targetRevision: main
    path: infra/payments
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    automated:
      prune: false         # ⚠️ Don't auto-delete infra claims — require manual cleanup
      selfHeal: true
---
# gitops/apps/analytics.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: analytics-infra
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/finedge/platform-infra
    targetRevision: main
    path: infra/analytics
  destination:
    server: https://kubernetes.default.svc
    namespace: analytics
  syncPolicy:
    automated:
      prune: false         # ⚠️ Protect database claims from accidental deletion
      selfHeal: true
---
# gitops/apps/notifications.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: notifications-infra
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/finedge/platform-infra
    targetRevision: main
    path: infra/notifications
  destination:
    server: https://kubernetes.default.svc
    namespace: notifications
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

> **⚠️ Why `prune: false` for team apps?**
> The original plan set `prune: true` on all team Applications. This is dangerous:
> if someone removes a `claim.yaml` from Git (even accidentally), ArgoCD would
> delete the claim, which cascades to deleting the underlying AWS resources (S3,
> RDS, SQS). Setting `prune: false` requires explicit manual deletion via ArgoCD
> UI or CLI, providing a safety net.

```bash
kubectl apply -f gitops/argocd-apps.yaml

# Watch sync status
kubectl get applications -n argocd
# NAME                   SYNC STATUS   HEALTH STATUS
# finedge-infra-root     Synced        Healthy
# payments-infra         Synced        Healthy
# analytics-infra        Synced        Progressing
# notifications-infra    Synced        Healthy
```

**Developer workflow from this point forward:**

```bash
# 1. Edit a claim in Git
#    e.g. analytics team wants to increase storage
git checkout -b infra/analytics-storage-increase

# 2. Edit infra/analytics/claim.yaml
#    storageGb: 50 → 100

git commit -am "infra: increase analytics db storage to 100GB (INFRA-204)"
git push origin infra/analytics-storage-increase

# 3. Open a Pull Request → review → merge
# 4. ArgoCD detects the change within 3 minutes and applies it
# 5. Crossplane reconciles the RDS instance
```

---

## Phase 5 — Guardrails with Kyverno

**Goal:** Enforce platform policies that prevent unsafe claims from being admitted.

> **Kyverno API Migration Note (2026):**
> Kyverno has shifted to a CEL-first architecture. The legacy `ClusterPolicy`
> (kyverno.io/v1) is deprecated and scheduled for removal in v1.20 (~October 2026).
> New policy types: `ValidatingPolicy`, `MutatingPolicy`, `GeneratingPolicy`.
>
> This plan uses the **legacy `ClusterPolicy` API** for simplicity and wider
> compatibility. A migration section is provided at the end for teams ready to
> adopt the new CEL-based policies.

### 5.1 Install Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno \
  --namespace kyverno --create-namespace \
  --set replicaCount=1 \
  --wait
```

### 5.2 Guardrail Policies

```yaml
# policies/require-owner-annotation.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: finedge-appenv-require-owner
  annotations:
    policies.kyverno.io/description: >
      Every AppEnv claim must declare an owner email and a JIRA ticket.
      This creates an audit trail and ensures someone is accountable for cost.
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: require-owner-email
      match:
        resources:
          kinds: [AppEnv]
      validate:
        message: "Missing annotation: platform.finedge.io/owner (must be a @finedge.io email)"
        pattern:
          metadata:
            annotations:
              platform.finedge.io/owner: "*@finedge.io"
    - name: require-infra-ticket
      match:
        resources:
          kinds: [AppEnv]
      validate:
        message: "Missing annotation: platform.finedge.io/ticket (format: INFRA-NNN)"
        pattern:
          metadata:
            annotations:
              platform.finedge.io/ticket: "INFRA-?"
---
# policies/block-public-buckets.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: finedge-block-public-storage
spec:
  validationFailureAction: Enforce
  rules:
    - name: encryption-required
      match:
        resources:
          kinds: [AppEnv]
      preconditions:
        any:
          - key: "{{ request.object.spec.parameters.storage.enabled }}"
            operator: Equals
            value: true
      validate:
        message: "Storage must have encryption enabled (PCI-DSS requirement)"
        pattern:
          spec:
            parameters:
              storage:
                encryption: true
---
# policies/restrict-prod-instance.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: finedge-restrict-prod-db
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-micro-in-prod
      match:
        resources:
          kinds: [AppEnv]
      preconditions:
        all:
          - key: "{{ request.object.spec.parameters.environment }}"
            operator: Equals
            value: prod
          - key: "{{ request.object.spec.parameters.database.enabled }}"
            operator: Equals
            value: true
      validate:
        message: "Production databases cannot use db.t3.micro — minimum is db.t3.small"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.parameters.database.instanceClass }}"
                operator: Equals
                value: db.t3.micro
    - name: prod-requires-minimum-storage
      match:
        resources:
          kinds: [AppEnv]
      preconditions:
        all:
          - key: "{{ request.object.spec.parameters.environment }}"
            operator: Equals
            value: prod
          - key: "{{ request.object.spec.parameters.database.enabled }}"
            operator: Equals
            value: true
      validate:
        message: "Production databases must declare storageGb >= 50"
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.parameters.database.storageGb }}"
                operator: LessThan
                value: 50
```

```bash
kubectl apply -f policies/

# Test the guard — should be denied
kubectl apply -f - <<'EOF'
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: bad-claim
  namespace: payments          # no annotations!
spec:
  parameters:
    teamName:    rogue
    environment: dev
    storage:
      enabled:    true
      encryption: false        # also violates block-public-buckets
EOF
# Error: Missing annotation: platform.finedge.io/owner (must be a @finedge.io email)
```

### 5.3 Future Migration to CEL-based ValidatingPolicy

When ready to migrate to the new Kyverno CEL APIs (recommended before October 2026),
here's the equivalent of the owner-annotation policy:

```yaml
# Example: CEL-based ValidatingPolicy (Kyverno v1.17+)
apiVersion: kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: finedge-appenv-require-owner-cel
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: ["platform.finedge.io"]
        apiVersions: ["v1alpha1"]
        resources: ["appenvs"]
        operations: ["CREATE", "UPDATE"]
  validations:
    - expression: >
        has(object.metadata.annotations) &&
        'platform.finedge.io/owner' in object.metadata.annotations &&
        object.metadata.annotations['platform.finedge.io/owner'].endsWith('@finedge.io')
      message: "Missing annotation: platform.finedge.io/owner (must be a @finedge.io email)"
    - expression: >
        has(object.metadata.annotations) &&
        'platform.finedge.io/ticket' in object.metadata.annotations &&
        object.metadata.annotations['platform.finedge.io/ticket'].startsWith('INFRA-')
      message: "Missing annotation: platform.finedge.io/ticket (format: INFRA-NNN)"
```

---

## Phase 6 — Observability & Drift Recovery

**Goal:** Monitor Crossplane health in Grafana and demonstrate self-healing when
resources are manually deleted.

### 6.1 Install kube-prometheus-stack

```bash
# Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=platform123 \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --wait

kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring &
# Open http://localhost:3000  user: admin / platform123
```

### 6.2 Prometheus Alert Rules

```yaml
# observability/prometheusrule-crossplane.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: crossplane-platform-rules
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: crossplane.resources
      interval: 30s
      rules:
        # Alert when any managed resource has been unsynced for > 5 minutes
        - alert: CrossplaneResourceNotSynced
          expr: |
            crossplane_managed_resource_ready{ready="False"} == 1
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Crossplane resource {{ $labels.name }} is not synced"
            description: >
              Resource {{ $labels.kind }}/{{ $labels.name }} has been unready for 5+ minutes.
              This may indicate a LocalStack or provider issue.

        # Alert when a provider is unhealthy
        - alert: CrossplaneProviderUnhealthy
          expr: |
            crossplane_pkg_revision_healthy{type="Provider"} == 0
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Crossplane provider {{ $labels.name }} is unhealthy"

        # Track total managed resources by kind (for dashboards)
        - record: crossplane:managed_resources:total
          expr: count by (kind) (crossplane_managed_resource_ready)
```

```bash
kubectl apply -f observability/prometheusrule-crossplane.yaml
```

Key Grafana panels to build:

```
Panel 1: Total managed resources by kind (bar chart)
  Query: count by (kind) (crossplane_managed_resource_ready)

Panel 2: Unready resources over time (time series)
  Query: sum(crossplane_managed_resource_ready{ready="False"})

Panel 3: Provider health status (stat panels)
  Query: crossplane_pkg_revision_healthy{type="Provider"}

Panel 4: Claims per team namespace (pie chart)
  Query: count by (namespace) (kube_customresource_info{kind="AppEnv"})
```

### 6.3 Drift Detection Demo

Crossplane's reconcile loop runs every ~30 seconds. Delete a LocalStack resource
and watch it come back.

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_PROFILE=localstack

# 1. Confirm the bucket exists
aws s3 ls | grep payments
# finedge-payments-dev-data

# 2. Someone manually deletes it (simulating a "rogue action" or accidental CLI delete)
aws s3 rb s3://finedge-payments-dev-data --force
aws s3 ls | grep payments
# (nothing — bucket is gone)

# 3. Watch Crossplane notice and fix it
kubectl get events -n crossplane-system --field-selector reason=CannotObserveExternalResource
# ...Bucket finedge-payments-dev-data not found, will recreate

# After ~30 seconds:
aws s3 ls | grep payments
# finedge-payments-dev-data  ← recreated automatically
```

---

## Phase 7 — Testing & Validation

**Goal:** Validate XRDs, Compositions, and Policies before applying to the cluster.
Catch errors in CI, not in production.

### 7.1 Composition Rendering Tests

Use `crossplane render` to preview what resources a composition will produce
*without* applying to a cluster:

```bash
# Render a composition locally to see what it would create
crossplane render \
  infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml \
  --observed-resources tests/compositions/observed-resources.yaml
```

### 7.2 Kyverno Policy Tests

Use the Kyverno CLI to test policies against sample claims:

```yaml
# tests/policies/test-valid-claim.yaml
# This claim should pass all policies
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: test-valid
  namespace: payments
  annotations:
    platform.finedge.io/owner: "test@finedge.io"
    platform.finedge.io/ticket: "INFRA-999"
spec:
  parameters:
    teamName: payments
    environment: dev
    storage:
      enabled: true
      encryption: true
```

```yaml
# tests/policies/test-invalid-claim.yaml
# This claim should be rejected (missing annotations, encryption off)
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: test-invalid
  namespace: payments
spec:
  parameters:
    teamName: rogue
    environment: dev
    storage:
      enabled: true
      encryption: false
```

```bash
# Test policies locally (no cluster needed)
kyverno apply policies/ --resource tests/policies/test-valid-claim.yaml
# pass: finedge-appenv-require-owner / require-owner-email: pass
# pass: finedge-block-public-storage / encryption-required: pass

kyverno apply policies/ --resource tests/policies/test-invalid-claim.yaml
# fail: finedge-appenv-require-owner / require-owner-email: fail
# fail: finedge-block-public-storage / encryption-required: fail
```

### 7.3 CI Pipeline Integration

Add these validation steps to your CI pipeline (GitHub Actions example):

```yaml
# .github/workflows/validate-infra.yaml
name: Validate Infrastructure Changes
on:
  pull_request:
    paths: ['infra/**', 'crossplane/**', 'policies/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Crossplane CLI
        run: |
          curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
          sudo mv crossplane /usr/local/bin/

      - name: Render compositions
        run: |
          for claim in infra/*/claim.yaml; do
            echo "--- Rendering $claim ---"
            crossplane render $claim \
              crossplane/composition-appenv.yaml \
              crossplane/functions.yaml
          done

      - name: Install Kyverno CLI
        run: |
          curl -sL https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli_linux_amd64.tar.gz | tar xz
          sudo mv kyverno /usr/local/bin/

      - name: Validate policies
        run: |
          for claim in infra/*/claim.yaml; do
            echo "--- Validating $claim ---"
            kyverno apply policies/ --resource $claim
          done
```

---

## Phase 8 — Graduation to Real AWS

**Goal:** When a team is ready for staging or production, flip one label.
The same claim, same schema, same Git workflow — but now real AWS resources are created.

### 8.1 Add a real AWS ProviderConfig

```bash
# Store real credentials as a secret
kubectl create secret generic aws-real-creds \
  --namespace crossplane-system \
  --from-file=credentials=$HOME/.aws/credentials
```

```yaml
# crossplane/providerconfig-aws.yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-real
  annotations:
    platform.finedge.io/description: "Real AWS — staging/prod only"
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-real-creds
      key: credentials
  # No endpoint override — traffic goes to real AWS
```

### 8.2 Add a Composition for real AWS

The production composition differs from LocalStack in three critical ways:
1. **`providerConfigRef.name: aws-real`** — routes to real AWS
2. **`deletionPolicy: Orphan`** on RDS — prevents accidental database deletion
3. **`deletionProtection: true`** on RDS — AWS-level deletion guard

```yaml
# crossplane/composition-appenv-aws.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: finedge-appenv-aws
  labels:
    provider: aws          # ← different label from localstack
    channel: prod
spec:
  compositeTypeRef:
    apiVersion: platform.finedge.io/v1alpha1
    kind: XAppEnv
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: data-bucket
            base:
              apiVersion: s3.aws.upbound.io/v1beta1
              kind: Bucket
              spec:
                deletionPolicy: Delete
                forProvider:
                  region: us-east-1
                providerConfigRef:
                  name: aws-real          # ← only this changes
            patches:
              # same patches as localstack composition

          - name: rds-instance
            base:
              apiVersion: rds.aws.upbound.io/v1beta1
              kind: Instance
              spec:
                deletionPolicy: Orphan    # ← CRITICAL: don't delete prod databases
                forProvider:
                  region: us-east-1
                  skipFinalSnapshot: false # ← Take final snapshot before deletion
                  deletionProtection: true # ← AWS-level guard
                  publiclyAccessible: false
                  storageEncrypted: true
                  multiAz: true           # ← Required for production HA
                providerConfigRef:
                  name: aws-real
            patches:
              # same patches as localstack composition

          # ... remaining resources with providerConfigRef: aws-real

    - step: auto-ready
      functionRef:
        name: function-auto-ready
```

### 8.3 Team graduation: one field change

```yaml
# infra/analytics/claim.yaml  — promoting to staging
apiVersion: platform.finedge.io/v1alpha1
kind: AppEnv
metadata:
  name: analytics-staging
  namespace: analytics
  annotations:
    platform.finedge.io/owner:  "analytics-team@finedge.io"
    platform.finedge.io/ticket: "INFRA-250"
spec:
  compositionSelector:
    matchLabels:
      provider: aws           # ← was: localstack
      channel: prod
  parameters:
    teamName:    analytics
    environment: staging
    storage:
      enabled:    true
      versioning: true
      encryption: true
    database:
      enabled:       true
      instanceClass: db.t3.small   # Kyverno enforces this minimum for prod
      storageGb:     100
```

The PR diff is six lines. Merging it creates real AWS infrastructure.

---

## Troubleshooting Quick Reference

```bash
# Resource stuck in Synced: False
kubectl describe <kind> <name>              # look at Events section

# Full claim tree with health status
kubectl crossplane beta trace appenv <name> -n <namespace>

# Provider logs (change provider-aws-s3 to whichever is failing)
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=provider-aws-s3 --tail=40

# LocalStack is alive on the host?
curl http://localhost:4566/_localstack/health | jq .services

# kind nodes can reach LocalStack on the host?
kubectl run reach-test --rm -it --restart=Never --image=curlimages/curl \
  -- curl -s http://host.docker.internal:4566/_localstack/health

# See all events across crossplane-system
kubectl get events -n crossplane-system --sort-by='.lastTimestamp' | tail -20

# Render a composition locally for debugging
crossplane render infra/payments/claim.yaml \
  crossplane/composition-appenv.yaml \
  crossplane/functions.yaml

# Check function health
kubectl get functions
kubectl logs -n crossplane-system -l pkg.crossplane.io/revision=function-patch-and-transform --tail=20
```

---

## What You've Built

| Phase | What you did |
|---|---|
| **1 — Bootstrap** | kind cluster + LocalStack 2026.x (auth token) + Crossplane **v2.3** with Composition Functions pointing to `host.docker.internal:4566` |
| **2 — Platform API** | XRD schema (the claim form) + Composition with `function-auto-ready`, explicit `deletionPolicy`, and `defaultCompositionRef` |
| **3 — Security + Claims** | RBAC per namespace · NetworkPolicies · ResourceQuotas · Pod Security Admission · Three realistic team claims |
| **4 — GitOps** | ArgoCD app-of-apps with `prune: false` safety — infra changes flow through Git PRs, not kubectl |
| **5 — Guardrails** | Kyverno policies block unsafe claims (owner required, encryption required, prod DB sizing); CEL migration path documented |
| **6 — Observability** | Prometheus rules + Grafana dashboards track resource health; drift self-heals in ~30s |
| **7 — Testing** | `crossplane render` for composition preview + `kyverno apply` for policy validation + CI pipeline integration |
| **8 — Graduation** | One label change in a claim promotes LocalStack infra to real AWS with production-grade settings (Orphan deletion, deletionProtection, Multi-AZ) |

**Platform team owns:** Providers, ProviderConfigs, Functions, XRDs, Compositions, Policies, RBAC, Quotas.
**Product teams own:** Claims (20 lines, no AWS knowledge required).
**Git owns:** Everything — every infra change is a PR, reviewable, auditable, rollback-able.

---

## Changelog from Initial Plan

| Area | Before | After | Why |
|------|--------|-------|-----|
| Crossplane | v1.15.0 | **v2.3.1** | v1.x is EOL; v2 adds namespaced resources, GA Usages, broader composition |
| `--enable-usages` | Set in Helm args | **Removed** | Usages are GA/default in v2.x |
| Composition Functions | Not installed | **functions.yaml** added | `function-patch-and-transform` + `function-auto-ready` required in v2 |
| AWS Provider | v1.2.0 | **v1.17.0** | 15 versions behind; many bug fixes and new resources |
| kind prerequisite | >= 0.22 | **>= 0.31** | Matches latest stable with cgroup v2 support |
| LocalStack | >= 3.4, `SERVICES=` env var | **>= 2026.01** (CalVer), auth token required | Calendar versioning since March 2026; all services start by default |
| `deletionPolicy` | Not set | **Explicit on every resource** | Never rely on defaults; Delete for dev, Orphan for prod RDS |
| `function-auto-ready` | Not used | **Added as pipeline step** | Prevents false-positive Ready status during provisioning |
| RBAC | Not present | **Full RBAC** | Platform admin + team developer roles with namespace-scoped bindings |
| Network Policies | Not present | **Default deny + allow-lists** | Namespace isolation between teams |
| Resource Quotas | Not present | **5 claims/namespace limit** | Prevent runaway provisioning |
| Pod Security | Not present | **PSA labels** on team namespaces | Baseline enforce / Restricted warn+audit |
| ArgoCD prune | `prune: true` on all apps | **`prune: false`** for team apps | Prevent accidental infra deletion when claims removed from Git |
| RDS protection | Not set | **`deletionProtection`** added | AWS-level guard for production databases |
| Kyverno | Using `ClusterPolicy` only | Added **CEL migration example** | ClusterPolicy deprecated, removal ~October 2026 |
| Testing | Not present | **Phase 7** with render + policy tests + CI | Catch errors before they reach the cluster |
| `defaultCompositionRef` | Not set in XRD | **Set to localstack** | Teams don't need to specify composition for dev |
| Project structure | 5 directories | **7 directories** (`security/`, `tests/` added) | Security and testing are first-class concerns |
| Phases | 6 phases | **8 phases** | Security hardening (Phase 3) and Testing (Phase 7) added as dedicated phases |
