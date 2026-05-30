---
tags: [troubleshooting, index]
created: 2026-05-29
---

# 🔥 Troubleshooting Index

> Add pages here as you encounter and solve problems.
> Each troubleshooting page should document: **symptoms → diagnosis → solution → prevention**.

---

## Template for New Troubleshooting Pages

When you encounter an issue, create a new page in `troubleshooting/` using this template:

```markdown
---
tags: [troubleshooting, <component>]
created: YYYY-MM-DD
resolved: true/false
---

# <Short Description of the Problem>

## Symptoms
- What you observed
- Error messages (exact text)
- Which commands failed

## Context
- What you were trying to do
- Which phase of the project
- Related concepts: [[wikilink]]

## Diagnosis
- How you figured out what was wrong
- Commands that helped diagnose

## Solution
- What fixed it
- Code/command changes made

## Prevention
- How to avoid this in the future
- Any guardrails or tests to add

## Related
- [[relevant concept]]
- [[relevant tool]]
```

---

## Common Issues (Pre-populated)

### Crossplane
- _Resources stuck in `Synced: False`_ — create a page when encountered
- _Provider not becoming `Healthy`_ — create a page when encountered
- _Composition not creating all resources_ — create a page when encountered

### LocalStack
- _`host.docker.internal` not resolving on Linux_ — see [[LocalStack]], [[kind]]
- _Auth token errors after LocalStack 2026.x update_ — see [[LocalStack]]
- _Services not responding on port 4566_ — create a page when encountered

### ArgoCD
- _Application stuck in `Progressing`_ — create a page when encountered
- _Sync failed with "resource already exists"_ — create a page when encountered

### Kyverno
- _Policy not blocking invalid claims_ — create a page when encountered
- _`ClusterPolicy` deprecation warnings_ — see [[Kyverno]]

### kind
- _Nodes not reaching `Ready` state_ — create a page when encountered
- _Port conflicts on 8080/8443_ — create a page when encountered

---

## Quick Diagnostic Commands

```bash
# Crossplane — resource status
kubectl describe <kind> <name>
kubectl crossplane beta trace appenv <name> -n <namespace>
kubectl get events -n crossplane-system --sort-by='.lastTimestamp' | tail -20

# Crossplane — provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/revision=provider-aws-s3 --tail=40

# LocalStack — health check
curl http://localhost:4566/_localstack/health | jq .services

# LocalStack — connectivity from inside cluster
kubectl run reach-test --rm -it --restart=Never --image=curlimages/curl \
  -- curl -s http://host.docker.internal:4566/_localstack/health

# ArgoCD — sync status
kubectl get applications -n argocd
argocd app get <app-name>

# Kyverno — policy reports
kubectl get policyreport -A
kubectl get clusterpolicyreport
```
