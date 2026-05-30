---
tags: [concept, kubernetes, security, psa]
created: 2026-05-30
---

# Pod Security Admission

**Pod Security Admission (PSA)** is Kubernetes' built-in mechanism for enforcing security standards on pods. It replaced PodSecurityPolicy (removed in K8s 1.25) with a simpler, namespace-level label-based system. In FinEdge, PSA ensures team pods can't run in privileged mode, mount host paths, or do other dangerous things.

## The Three Profiles

PSA defines three security profiles, each progressively more restrictive:

### 1. Privileged (No Restrictions)
```
Allow everything. Only for system-level workloads.
```
- Allows: hostNetwork, hostPID, privileged containers, any capabilities
- Used by: `crossplane-system`, `kyverno`, `monitoring` namespaces
- **Never** for team namespaces

### 2. Baseline (Reasonable Default)
```
Prevent known privilege escalations. Still allows most workloads.
```
Blocks:
- `hostNetwork: true`
- `hostPID: true`
- `privileged: true`
- `hostPath` volume mounts
- Dangerous capabilities (`NET_RAW`, `SYS_ADMIN`, etc.)

Allows:
- Running as root (not ideal, but many images require it)
- `emptyDir`, `configMap`, `secret` volumes
- Standard capabilities

### 3. Restricted (Maximum Security)
```
Best practices. Some legitimate workloads may need modification.
```
Everything in Baseline, PLUS:
- Must set `runAsNonRoot: true`
- Must drop ALL capabilities
- Must set `readOnlyRootFilesystem: true`
- Must set `allowPrivilegeEscalation: false`
- Seccomp profile must be `RuntimeDefault` or `Localhost`

## The Three Modes

Each profile can be applied in three modes:

| Mode | What Happens | When to Use |
|------|-------------|-------------|
| **enforce** | Pod creation is **rejected** | When you're confident the profile works |
| **warn** | Pod is created but kubectl shows a **warning** | Testing a stricter profile |
| **audit** | Pod is created, violation logged to **audit log** | Silent monitoring |

## FinEdge PSA Configuration

Team namespaces use a combination:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    # Enforce baseline — block dangerous pods
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    
    # Warn on restricted — show developers what to fix
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    
    # Audit restricted — log violations for review
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

This means:
1. Pods violating **baseline** are **blocked** (can't run privileged containers)
2. Pods violating **restricted** get a **warning** (so devs know what to fix)
3. All restricted violations are **logged** (for the platform team to review)

This gradual approach lets teams migrate to restricted over time without breaking their workloads.

## What Gets Blocked?

### Baseline Enforce (FinEdge)

```yaml
# ❌ BLOCKED — hostNetwork
spec:
  hostNetwork: true

# ❌ BLOCKED — privileged container
spec:
  containers:
    - securityContext:
        privileged: true

# ❌ BLOCKED — hostPath mount
spec:
  volumes:
    - hostPath:
        path: /etc/shadow

# ✅ ALLOWED — normal pod
spec:
  containers:
    - image: nginx:latest
      ports:
        - containerPort: 80
```

### Restricted Warn (FinEdge)

```yaml
# ⚠️ WARNING — running as root
spec:
  containers:
    - securityContext:
        runAsNonRoot: false  # or missing

# ⚠️ WARNING — not dropping capabilities  
spec:
  containers:
    - securityContext:
        capabilities:
          drop: []  # Should be ["ALL"]

# ✅ CLEAN — fully restricted-compliant
spec:
  containers:
    - securityContext:
        runAsNonRoot: true
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        seccompProfile:
          type: RuntimeDefault
```

## PSA vs. Kyverno

PSA and [[Kyverno]] serve different roles:

| Feature | PSA | Kyverno |
|---------|-----|---------|
| Scope | Pod security only | Any Kubernetes resource |
| Customization | 3 fixed profiles | Fully custom rules |
| Granularity | Namespace-level | Per-resource/per-field |
| Built-in | Yes (K8s core) | No (requires installation) |
| FinEdge use | Baseline security floor | Business-logic validation (owner, ticket) |

FinEdge uses **both**: PSA for pod security, [[Kyverno]] for application-level policies (require owner annotation, require ticket, validate claim parameters).

## Migration Path: Baseline → Restricted

1. Start with `enforce: baseline`, `warn: restricted`
2. Review warnings: `kubectl get events --field-selector reason=FailedCreate`
3. Fix workloads to be restricted-compliant (add security contexts)
4. Once no warnings appear, switch to `enforce: restricted`

## Related

- [[Namespace Isolation]]
- [[Kubernetes RBAC]]
- [[Policy as Code]]
- [[Kyverno]]
- [[Network Policies]]
