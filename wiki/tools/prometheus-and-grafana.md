---
tags: [tool, prometheus, grafana, monitoring, observability]
created: 2026-05-30
---

# Prometheus and Grafana

**Prometheus** is a metrics collection and alerting system. **Grafana** is a visualization and dashboarding platform. Together, they form the observability layer for FinEdge — monitoring [[Crossplane]] resource health, [[Drift Detection|drift]] events, provider latency, and cluster health.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                    │
│                                                          │
│  ┌────────────┐    scrape     ┌──────────────────────┐  │
│  │ Crossplane  │──────────────▶│    Prometheus         │  │
│  │ Provider    │               │                      │  │
│  │ Metrics     │               │  - Stores metrics    │  │
│  └────────────┘               │  - Evaluates rules   │  │
│                                │  - Fires alerts      │  │
│  ┌────────────┐    scrape     │                      │  │
│  │ Kubernetes  │──────────────▶│                      │  │
│  │ API Metrics │               └───────────┬──────────┘  │
│  └────────────┘                            │             │
│                                   query    │             │
│  ┌────────────┐    scrape     ┌───────────▼──────────┐  │
│  │ Node        │──────────────▶│    Grafana           │  │
│  │ Metrics     │               │                      │  │
│  └────────────┘               │  - Dashboards        │  │
│                                │  - Visualizations    │  │
│  ┌────────────┐               │  - Alerts            │  │
│  │ Alertmanager│◀──── alerts ──│                      │  │
│  └────────────┘               └──────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Installation (kube-prometheus-stack)

FinEdge uses the `kube-prometheus-stack` [[Helm]] chart which bundles everything:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30300 \
  --set grafana.adminPassword=admin
```

This installs:
- Prometheus (metrics collection + alerting)
- Grafana (dashboards + visualization)
- Alertmanager (alert routing + notification)
- Node exporter (host-level metrics)
- kube-state-metrics (Kubernetes object metrics)

## Crossplane Monitoring

### Key Metrics

| Metric | Description | Use |
|--------|-------------|-----|
| `crossplane_managed_resource_exists` | Does the MR exist? | Alert if resource disappears |
| `crossplane_managed_resource_ready` | Is the MR Ready? | Alert on prolonged non-Ready |
| `controller_runtime_reconcile_total` | Total reconciliations | Monitor reconciliation rate |
| `controller_runtime_reconcile_errors_total` | Failed reconciliations | Alert on error spikes |
| `controller_runtime_reconcile_time_seconds` | Reconciliation duration | Detect slow providers |

### PrometheusRule for Crossplane

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: crossplane-alerts
  namespace: monitoring
spec:
  groups:
    - name: crossplane
      rules:
        - alert: ManagedResourceNotReady
          expr: |
            crossplane_managed_resource_ready == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Managed resource {{ $labels.name }} is not Ready"
            description: "The managed resource has been not Ready for more than 5 minutes"

        - alert: HighReconcileErrors
          expr: |
            rate(controller_runtime_reconcile_errors_total[5m]) > 0.1
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "High reconciliation error rate"
```

## Grafana Dashboards

### Accessing Grafana

```bash
# If using NodePort
open http://localhost:30300

# If using port-forward
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
open http://localhost:3000

# Default credentials
# Username: admin
# Password: admin (or what you set in Helm values)
```

### Useful Dashboard Panels

#### Crossplane Overview
- **Resource count by type** (how many Buckets, Queues, etc.)
- **Ready vs. Not Ready ratio**
- **Reconciliation rate** (reconciliations/minute)
- **Error rate** (errors/minute)

#### PromQL Examples

```promql
# Count of managed resources by type
count by (kind) (crossplane_managed_resource_exists == 1)

# Resources not ready
crossplane_managed_resource_ready == 0

# Reconciliation rate per controller
rate(controller_runtime_reconcile_total[5m])

# Average reconciliation time
histogram_quantile(0.95, rate(controller_runtime_reconcile_time_seconds_bucket[5m]))

# Provider pod memory usage
container_memory_working_set_bytes{namespace="crossplane-system"}
```

## Alerting

### Alert Flow

```
Prometheus evaluates rules every 30s
    │
    ▼ (condition met for configured duration)
Alertmanager receives alert
    │
    ▼ (route matching)
Notification sent (email, Slack, PagerDuty, etc.)
```

### Example: Alert on Drift

```yaml
- alert: DriftDetected
  expr: |
    crossplane_managed_resource_ready == 1
    AND crossplane_managed_resource_synced == 0
  for: 2m
  labels:
    severity: info
  annotations:
    summary: "Drift detected on {{ $labels.name }}"
    description: >
      Resource is Ready but not Synced, indicating
      actual state differs from desired. Crossplane
      should auto-correct within 30 seconds.
```

## Cluster Health Metrics

Beyond Crossplane, monitor overall cluster health:

```promql
# Node CPU usage
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Pod restarts (indicates instability)
increase(kube_pod_container_status_restarts_total[1h]) > 3

# Persistent volume usage
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100

# API server request latency
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))
```

## Related

- [[Crossplane]]
- [[Managed Resources]]
- [[Drift Detection]]
- [[Helm]]
- [[Reconciliation Loop]]
