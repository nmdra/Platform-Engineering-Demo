---
tags: [concept, kubernetes, controller-pattern, reconciliation]
created: 2026-05-30
---

# Reconciliation Loop

The **Reconciliation Loop** is the fundamental control pattern in Kubernetes — and by extension, in [[Crossplane]] and [[ArgoCD]]. It's a continuous cycle: observe actual state, compare with desired state, and take action to close the gap. Understanding this pattern is key to understanding everything in this project.

## The Pattern

```
         ┌─────────────────────────┐
         │     Desired State       │
         │  (Kubernetes manifests) │
         └───────────┬─────────────┘
                     │ compare
         ┌───────────▼─────────────┐
         │       Controller        │
         │                         │
         │  if desired ≠ actual:   │
         │    take action          │
         │                         │
         └───────────┬─────────────┘
                     │ observe
         ┌───────────▼─────────────┐
         │      Actual State       │
         │  (running resources)    │
         └─────────────────────────┘
              ↻ repeat every ~30s
```

## Level-Triggered vs. Edge-Triggered

This is the most important distinction to understand:

### Edge-Triggered (Events)
"When something **changes**, do something."
```
Event: "Bucket was deleted"
Action: "Create a new bucket"
Problem: What if the event is missed? The bucket stays deleted.
```

### Level-Triggered (State)
"Whenever the actual state **differs** from desired, fix it."
```
Every 30 seconds:
  Check: "Does the bucket exist?"
  If no: "Create it"
  If yes: "Is it configured correctly?"
    If no: "Update it"
    If yes: "Do nothing"
```

Kubernetes and Crossplane are **level-triggered**. This is why:
- If a controller crashes and restarts, it doesn't miss anything — it just reads current state
- If the network blips and an event is lost, the next reconciliation catches it
- The system is **eventually consistent** — it always converges to the desired state

## Reconciliation in FinEdge

### Layer 1: Kubernetes Controllers
```
Desired: Deployment spec says 3 replicas
Actual:  Only 2 pods running (one crashed)
Action:  Start a new pod to reach 3
```

### Layer 2: Crossplane Providers
```
Desired: Bucket MR spec says region=us-east-1, encryption=AES256
Actual:  Bucket exists but encryption was changed manually
Action:  Update bucket encryption back to AES256
```
This is [[Drift Detection]] — Crossplane fixes drift automatically.

### Layer 3: ArgoCD
```
Desired: Git repo has Claim with storage.enabled: true
Actual:  Cluster has Claim with storage.enabled: false (someone changed it)
Action:  Revert Claim back to Git version (selfHeal: true)
```

## The Crossplane Reconciliation Cycle

Every ~30 seconds, for each [[Managed Resources|Managed Resource]]:

```
1. OBSERVE
   Provider calls cloud API (e.g., DescribeBucket)
   Returns: actual bucket config (tags, encryption, region)

2. COMPARE
   Crossplane compares:
   - spec.forProvider (desired) vs atProvider (actual)
   - Are they the same?

3. ACT (if different)
   - Resource missing     → CREATE (call CreateBucket)
   - Resource config wrong → UPDATE (call PutBucketEncryption)
   - Resource shouldn't exist → DELETE (if deletionPolicy: Delete)
   - Resource matches      → NOOP (do nothing)

4. STATUS
   Update MR status:
   - conditions.Ready: True/False
   - conditions.Synced: True/False
   - status.atProvider: { arn: ..., id: ... }
```

## Convergence and Eventual Consistency

The reconciliation loop doesn't guarantee INSTANT consistency — it guarantees **eventual** consistency:

```
t=0s:   Developer creates Claim
t=1s:   Crossplane creates XR and MRs
t=5s:   Provider calls CreateBucket
t=10s:  Bucket exists, provider updates status
t=15s:  CreateQueue called
t=20s:  Queue exists
t=25s:  function-auto-ready checks all resources
t=30s:  All Ready → XR Ready → Claim Ready ✅
```

If something fails mid-way (e.g., rate limit on SQS), the next reconciliation (30s later) retries. No manual intervention needed.

## Reconciliation Interval

| Component | Default Interval | Configurable? |
|-----------|-----------------|---------------|
| Crossplane providers | ~30 seconds | Yes (`--poll-interval` flag) |
| ArgoCD | ~3 minutes | Yes (`timeout.reconciliation` in argocd-cm) |
| Kubernetes controllers | Immediate (watch-based) | N/A |

## Why This Matters for FinEdge

1. **Self-healing**: If someone deletes an S3 bucket in LocalStack, Crossplane recreates it
2. **Drift correction**: If someone changes tags manually, Crossplane fixes them
3. **Crash recovery**: If Crossplane pods restart, they pick up where they left off
4. **No "apply and pray"**: Unlike Terraform, you don't need to re-run to catch drift

## Debugging Reconciliation

```bash
# Check if a resource is being reconciled
kubectl get managed -o wide
# Look at SYNCED and READY columns

# See reconciliation events
kubectl describe bucket finedge-payments-dev-data
# Events section shows reconciliation activity

# Check provider logs for reconciliation details
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/revision=provider-aws-s3 --tail=50
```

## Related

- [[Crossplane]]
- [[Drift Detection]]
- [[ArgoCD]]
- [[Managed Resources]]
- [[GitOps]]
