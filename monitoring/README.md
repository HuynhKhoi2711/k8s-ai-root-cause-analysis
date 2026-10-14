# monitoring/

> **Status:** No monitoring configuration files have been created yet.  
> This README documents the planned observability stack and its purpose.

---

## Purpose

This directory will contain the Helm values files and configuration for the observability stack: **Prometheus** for metric collection and **Grafana** for visualisation.

Observability is a prerequisite for the ML training data pipeline. Prometheus scrapes service metrics during controlled incident runs; those metric snapshots are then labeled and used to train the root-cause classifier.

---

## Planned Stack

| Component | Role |
|-----------|------|
| **Prometheus** | Scrapes `/metrics` endpoints from all microservices and Kubernetes node/Pod metrics. Stores time-series data. |
| **Grafana** | Visualises Prometheus metrics through dashboards. Used during development to inspect incident patterns and validate data quality. |
| `kube-prometheus-stack` | Helm chart that bundles Prometheus Operator, Prometheus, Grafana, Alertmanager, and default Kubernetes dashboards. |

**Out of scope for the MVP:** Loki (log aggregation), Tempo or Jaeger (distributed tracing), and OpenTelemetry Collector. These may be added in a later phase.

---

## Required Metrics

The following metrics must be collected during incident runs to produce the ML training dataset:

| Metric | Source |
|--------|--------|
| `http_requests_total` (by status code) | Application `/metrics` endpoint |
| `http_request_duration_seconds` (histogram) | Application `/metrics` endpoint |
| `container_cpu_usage_seconds_total` | `kubelet` / `node-exporter` |
| `container_memory_working_set_bytes` | `kubelet` |
| `kube_pod_container_status_restarts_total` | `kube-state-metrics` |

These are the raw Prometheus metrics. Feature engineering (e.g., rate calculations, windowed aggregations) is performed in the `ml/` pipeline.

---

## Application Instrumentation Requirements

Each microservice must expose a Prometheus-compatible `/metrics` endpoint. Refer to `services/README.md` for the required metric list and the recommended Python client library.

Services must also have appropriate Kubernetes annotations so that the Prometheus Operator's `ServiceMonitor` can discover them automatically:

```yaml
metadata:
  labels:
    # This label selector must match the ServiceMonitor selector
    prometheus-scrape: "true"
```

The exact label and `ServiceMonitor` configuration will be defined when Kubernetes manifests are written.

---

## Planned Installation Approach

Prometheus and Grafana will be installed into the EKS cluster using Helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring/values-kube-prometheus-stack.yaml
```

A `values-kube-prometheus-stack.yaml` file will be created in this directory to configure:
- Grafana admin credentials (via Kubernetes Secret, not hardcoded in the values file).
- Persistent storage for Prometheus using the EBS CSI Driver.
- `ServiceMonitor` selectors for the microservices.

---

## Verification Steps

Once installed, verify the stack with:

```bash
# Check all Pods in the monitoring namespace are Running
kubectl get pods -n monitoring

# Port-forward Grafana to your local machine
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# Then open http://localhost:3000 in a browser

# Port-forward Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Then open http://localhost:9090 → Status → Targets
```

All application services should appear as active Prometheus scrape targets.

---

## Planned Subdirectory Structure

The following file will be created when implementation begins. It is **not created now** because no configuration exists yet.

```
monitoring/
└── values-kube-prometheus-stack.yaml
```

Grafana dashboard JSON exports may be stored here as well when they are ready.

---

## Current Implementation Status

No Helm values files or configuration exist yet. Implementation begins in **Phase 6** of the project roadmap (see root `README.md`).
