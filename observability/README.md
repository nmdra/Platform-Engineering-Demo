# observability/ — Monitoring & Alerting

This directory contains **Prometheus rules** and **Grafana dashboards** for monitoring Crossplane resource health.

## Files

| File | Description |
|------|-------------|
| `prometheusrule-crossplane.yaml` | PrometheusRule with alerts for unsynced resources and unhealthy providers |
| `dashboard-crossplane.json` | Grafana dashboard JSON (import via Grafana UI) |

## Alerts

| Alert | Severity | Fires When |
|-------|----------|------------|
| `CrossplaneResourceNotSynced` | Warning | A managed resource has been `Ready: False` for > 5 minutes |
| `CrossplaneProviderUnhealthy` | Critical | A Crossplane provider is unhealthy for > 2 minutes |

## Grafana Panels

| Panel | Query | Type |
|-------|-------|------|
| Managed resources by kind | `count by (kind) (crossplane_managed_resource_ready)` | Bar chart |
| Unready resources | `sum(crossplane_managed_resource_ready{ready="False"})` | Time series |
| Provider health | `crossplane_pkg_revision_healthy{type="Provider"}` | Stat |
| Claims per team | `count by (namespace) (kube_customresource_info{kind="AppEnv"})` | Pie chart |

## Setup

```bash
# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=platform123 \
  --wait

# Apply Crossplane alerting rules
kubectl apply -f observability/prometheusrule-crossplane.yaml

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring &
# Open http://localhost:3000  (admin / platform123)
```
