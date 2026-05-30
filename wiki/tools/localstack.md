---
tags: [tool, localstack, aws, development]
created: 2026-05-30
---

# LocalStack

**LocalStack** is a local AWS cloud emulator that provides a fully functional local AWS cloud stack. In FinEdge, it lets you develop and test [[Crossplane]] compositions against realistic AWS APIs without needing an AWS account, paying for resources, or risking production infrastructure.

## Why LocalStack?

| Without LocalStack | With LocalStack |
|-------------------|-----------------|
| Need an AWS account | No account needed |
| Pay for every resource | Free (Community) or cheap (Pro) |
| Risk creating real resources | Everything is local |
| Network latency to AWS | Instant local responses |
| Rate limits and throttling | No limits |
| Cleanup required after testing | `docker stop` and it's gone |

## How FinEdge Uses It

```
kind cluster (Kubernetes)          LocalStack (Docker)
┌─────────────────────────┐       ┌──────────────────┐
│ Crossplane              │       │ S3 API           │
│  → Provider calls ──────┼──────▶│ SQS API          │
│    CreateBucket, etc.   │       │ RDS API          │
│                         │       │ SNS API          │
│ Endpoint:               │       │                  │
│ host.docker.internal    │       │ Port: 4566       │
│ :4566                   │       │                  │
└─────────────────────────┘       └──────────────────┘
```

## Setup

### Running LocalStack

```bash
# Start with Docker (recommended)
docker run -d \
  --name localstack \
  -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=${LOCALSTACK_AUTH_TOKEN} \
  localstack/localstack:latest

# OR with Docker Compose (preferred for consistency)
# See scripts/docker-compose-localstack.yaml
```

### Important: Auth Token Required (2026+)

Since early 2026, LocalStack requires an auth token even for the Community edition:

```bash
# Set the token
export LOCALSTACK_AUTH_TOKEN="your-token-here"

# Without it, LocalStack won't start
# Get a free token at https://app.localstack.cloud
```

### Calendar Versioning (CalVer)

LocalStack switched from SemVer to **calendar versioning** in March 2026:
```
Old: v3.8.1 (SemVer)
New: 2026.05.1 (YYYY.MM.patch)
```

When pinning versions:
```yaml
# In Docker Compose
image: localstack/localstack:2026.05
```

## Connecting Crossplane to LocalStack

### The ProviderConfig

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: localstack
spec:
  endpoint:
    url:
      type: Static
      static: "http://host.docker.internal:4566"  # From inside kind
  credentials:
    source: Secret
    secretRef:
      name: localstack-creds
      namespace: crossplane-system
      key: credentials
  skip_credentials_validation: true
  skip_metadata_api_check: true
  skip_requesting_account_id: true
```

### The Credentials Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: localstack-creds
  namespace: crossplane-system
type: Opaque
stringData:
  credentials: |
    [default]
    aws_access_key_id = test
    aws_secret_access_key = test
```

Real AWS credentials aren't needed — LocalStack accepts anything.

## `host.docker.internal` on Linux

This is a common gotcha. On macOS/Windows, `host.docker.internal` just works. On Linux, you need to add it explicitly:

### For kind clusters
```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
# host.docker.internal is resolved via kind's networking
```

### Alternative: Use Docker network IP
```bash
# Find LocalStack's IP on the Docker network
docker inspect localstack | grep IPAddress

# Use this IP in ProviderConfig instead of host.docker.internal
```

## Supported AWS Services

LocalStack emulates many AWS services. FinEdge uses:

| Service | LocalStack Support | FinEdge Use |
|---------|-------------------|-------------|
| S3 | ✅ Full | Data storage buckets |
| SQS | ✅ Full | Event queues + DLQs |
| RDS | ⚠️ Partial (simulated) | Database instances |
| SNS | ✅ Full | Notification topics |
| IAM | ✅ Full | (Future: fine-grained policies) |

### Testing with AWS CLI

```bash
# List S3 buckets (after Crossplane creates them)
aws --endpoint-url=http://localhost:4566 s3 ls

# List SQS queues
aws --endpoint-url=http://localhost:4566 sqs list-queues

# Check a specific bucket
aws --endpoint-url=http://localhost:4566 s3api get-bucket-encryption \
  --bucket finedge-payments-dev-data
```

## Troubleshooting

### LocalStack Not Starting
```bash
# Check if auth token is set
echo $LOCALSTACK_AUTH_TOKEN

# Check container logs
docker logs localstack

# Common error: "Missing auth token"
# Fix: Set LOCALSTACK_AUTH_TOKEN env var
```

### Crossplane Can't Reach LocalStack
```bash
# From inside a kind node, test connectivity
docker exec kind-control-plane curl -s http://host.docker.internal:4566/_localstack/health

# If it fails, try the Docker bridge IP
docker exec kind-control-plane curl -s http://172.17.0.1:4566/_localstack/health
```

### Resources Not Appearing
```bash
# Check provider logs
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=provider-aws-s3 --tail=50

# Look for "connection refused" or "endpoint" errors
```

## Related

- [[Crossplane]]
- [[Providers]]
- [[kind]]
- [[Managed Resources]]
