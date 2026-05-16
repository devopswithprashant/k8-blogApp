# Central monitoring (Prometheus + Grafana)

Deploy the shared observability stack into the `monitoring` namespace.

```bash
kubectl apply -k monitoring/
```

## Scrape targets

| Prometheus job | Target | Metrics |
|----------------|--------|---------|
| `blog-service` | Pods `app=blog-service` with `prometheus.io/scrape=true` | Spring `/actuator/prometheus` |
| `dwp-platform-ui` | Pods `app=dwp-platform-ui` with scrape annotations | Next.js `/api/metrics` (port 3000) |
| `dwp-platform-ui-nginx` | Pods `app=dwp-platform-ui` on port 9113 | nginx-prometheus-exporter |
| `blackbox-dwp-platform-ui` | `dwp-platform-ui` Service `/health` and `/` | `probe_success`, `probe_duration_seconds` |

UI workload manifests: `k8s/ui/` (apply with `kubectl apply -k k8s/`).

Examples: `blog-service-scrape-example.yaml`, `dwp-platform-ui-scrape-example.yaml`.

## Access

```bash
kubectl -n monitoring port-forward svc/grafana 3000:3000
kubectl -n monitoring port-forward svc/prometheus 9090:9090
```

Default Grafana credentials are in `grafana-secret.yaml` (change before production).

## Verify UI scraping

```bash
# Targets should show dwp-platform-ui, dwp-platform-ui-nginx, blackbox-dwp-platform-ui as UP
kubectl -n monitoring port-forward svc/prometheus 9090:9090
# Open http://localhost:9090/targets
```
