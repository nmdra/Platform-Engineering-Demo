---
tags: [concept, kubernetes, security, networking]
created: 2026-05-30
---

# Network Policies

**Network Policies** are Kubernetes resources that control pod-to-pod and pod-to-external traffic at the network level. By default, all pods in a Kubernetes cluster can communicate with all other pods — Network Policies let you restrict this to only what's needed. In FinEdge, they enforce [[Namespace Isolation]] by default-denying all traffic in team namespaces.

## The Default Problem

Without Network Policies:
```
payments pod ──── CAN REACH ────▶ analytics pod     ← BAD
payments pod ──── CAN REACH ────▶ crossplane pod    ← VERY BAD
payments pod ──── CAN REACH ────▶ kyverno pod       ← EXTREMELY BAD
payments pod ──── CAN REACH ────▶ internet          ← MAYBE BAD
```

With Network Policies:
```
payments pod ──── CAN REACH ────▶ payments services ← ALLOWED
payments pod ──── BLOCKED ──────▶ analytics pod      ← DENIED
payments pod ──── CAN REACH ────▶ DNS (kube-system)  ← ALLOWED (necessary)
payments pod ──── BLOCKED ──────▶ everything else     ← DENIED
```

## FinEdge Network Policy Model

### Step 1: Default Deny Everything

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}           # Applies to ALL pods in namespace
  policyTypes:
    - Ingress               # Block incoming traffic
    - Egress                # Block outgoing traffic
```

This is the foundation. With this in place, pods can't send or receive ANY traffic.

### Step 2: Allow DNS (Required)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Without DNS, pods can't resolve service names — nothing works.

### Step 3: Allow Intra-Namespace Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}    # Any pod in same namespace
  egress:
    - to:
        - podSelector: {}    # Any pod in same namespace
```

Pods within `payments` can talk to each other but not to pods in `analytics` or `notifications`.

## Selectors

Network Policies use selectors to match traffic:

### podSelector
Match pods by labels:
```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: frontend    # Only pods with app=frontend label
```

### namespaceSelector
Match pods in specific namespaces:
```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            platform.finedge.io/team: payments  # Only from payments namespace
```

### CIDR Blocks
Match external IP ranges:
```yaml
egress:
  - to:
      - ipBlock:
          cidr: 10.0.0.0/8        # Internal network
          except:
            - 10.0.1.0/24          # Except this subnet
```

### Combined Selectors (AND vs OR)

```yaml
# AND — pod must match BOTH selectors
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            team: payments
        podSelector:             # Same array element = AND
          matchLabels:
            role: api

# OR — pod matches EITHER selector
ingress:
  - from:
      - namespaceSelector:       # First array element
          matchLabels:
            team: payments
      - podSelector:             # Second array element = OR
          matchLabels:
            role: api
```

**This is the most common source of Network Policy bugs.** Read the YAML indentation carefully.

## Common Patterns

### Allow Monitoring (Prometheus Scrape)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090           # Prometheus metrics port
```

### Allow ArgoCD Health Checks
```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: argocd
```

## Testing Network Policies

```bash
# Deploy a test pod in payments namespace
kubectl run test-pod -n payments --image=busybox --rm -it -- /bin/sh

# From inside the pod:
# Should work (same namespace):
wget -qO- http://my-service.payments:8080

# Should be blocked (different namespace):
wget -qO- http://some-service.analytics:8080 --timeout=3
# wget: download timed out

# Should work (DNS):
nslookup kubernetes.default
```

## Important Notes

1. **Network Policies are additive** — if multiple policies match a pod, their rules are UNION'd (allow if any policy allows)
2. **No deny rules** — you can only ALLOW traffic. Denial is implicit (not listed = blocked, when a policy exists)
3. **CNI must support it** — kind uses kindnet which has limited NetworkPolicy support. For full support, install Calico or Cilium
4. **Stateful** — if you allow egress to port 443, the return traffic is automatically allowed

## Related

- [[Namespace Isolation]]
- [[Kubernetes RBAC]]
- [[Pod Security Admission]]
- [[Policy as Code]]
