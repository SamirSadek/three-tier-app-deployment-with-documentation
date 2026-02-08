# Monitoring Setup for BMI Health Tracker

This document describes how Prometheus, Loki, and Grafana are deployed to monitor the BMI Health application on Kubernetes. It includes metrics collection, log aggregation, dashboards, and alerting.

## Overview

The monitoring stack consists of:
- **Prometheus**: scrapes metrics from backend, frontend, Redis, and PostgreSQL exporters
- **Loki**: collects logs from all pods via Promtail DaemonSet
- **Grafana**: visualizes metrics and logs with dashboards; also integrates with AlertManager
- **Promtail**: DaemonSet that tails logs from pods and sends to Loki
- **AlertManager**: triggers alerts based on PrometheusRules

## Components & Manifests

### Prometheus
- `prometheus-config.yaml` — ConfigMap with scrape configs for all services
- `prometheus-deployment.yaml` — Deployment, Service, ServiceAccount, and RBAC for Prometheus

Metrics scraped:
- `backend:3000/metrics` — application request rates, latencies, errors
- `postgres-exporter:9187` — PostgreSQL connection stats, query performance
- `redis-exporter:9121` — Redis memory, commands, key counts
- Kubernetes API for pod/node metrics

### Loki
- `loki-deployment.yaml` — Deployment, Service, and log storage backend
- `promtail-deployment.yaml` — DaemonSet running on all nodes to collect container logs

Logs collected:
- All container STDOUT/STDERR from `bmi-health` namespace
- Pod and container labels included for filtering in Grafana

### Grafana
- `grafana-deployment.yaml` — Deployment, Service, and datasource ConfigMap
- Datasources: Prometheus and Loki pre-configured
- Admin user: `admin` / `admin` (change password on first login)

### Alerting
- `prometheus-rules.yaml` — PrometheusRule manifests with alert conditions (HighErrorRate, PodCrashLooping, ServiceDown, etc.)

Example alerts:
- High error rate (>5% of requests returning 5xx)
- Pod crash looping (container restarts)
- Backend/PostgreSQL/Redis unreachable
- High memory/CPU usage in pods

## How to Deploy

### Prerequisites
- Kubernetes cluster with metrics-server (for node/pod metrics) or kube-state-metrics
- Install Prometheus Operator (if using PrometheusRule CRDs), or use simple Deployment + ConfigMap

### Deploy Monitoring Stack

```bash
# Apply manifests
kubectl apply -f kubernetes/prometheus-config.yaml
kubectl apply -f kubernetes/prometheus-deployment.yaml
kubectl apply -f kubernetes/loki-deployment.yaml
kubectl apply -f kubernetes/promtail-deployment.yaml
kubectl apply -f kubernetes/grafana-deployment.yaml
kubectl apply -f kubernetes/prometheus-rules.yaml
```

Verify deployments:

```bash
kubectl get pods -n bmi-health | grep -E "prometheus|loki|grafana|promtail"
kubectl get svc -n bmi-health | grep -E "prometheus|loki|grafana"
```

## How to Access Dashboards

### Grafana UI

Forward port to access locally:

```bash
kubectl port-forward -n bmi-health svc/grafana 3000:3000
```

Open browser: `http://localhost:3000`
- Login: `admin` / `admin`
- Change password on first login
- Datasources are pre-configured (Prometheus, Loki)

### Prometheus UI

Forward port:

```bash
kubectl port-forward -n bmi-health svc/prometheus 9090:9090
```

Open: `http://localhost:9090`
- View scraped targets: Status → Targets
- Query metrics: Graph tab
- View alerts: Alerts tab

### Loki (via Grafana)

Access logs through Grafana's Explore feature:
- Open Grafana → Explore → select `Loki` datasource
- Write LogQL queries like: `{job="kubernetes-pods", namespace="bmi-health"}`

## Dashboards Provided

1. **Application Dashboard** (`grafana/dashboards/application-dashboard.json`)
   - Request rate over time
   - Error rate (5xx responses)
   - Response latency (p95)
   - Memory usage by pod

2. **Kubernetes Cluster Health** (`grafana/dashboards/kubernetes-dashboard.json`)
   - Ready nodes / Failed pods count
   - Pod CPU and memory usage over time
   - Node health status

Both are JSON exports and can be imported into Grafana via UI or ConfigMap volumes.

## Metrics Collection

### Backend Application Metrics

Ensure backend exposes metrics endpoint `/metrics` (Prometheus format). Add a metrics library to the backend if not present:

```javascript
// backend/src/metrics.js (example)
const register = require('prom-client').register;
const Counter = require('prom-client').Counter;
const Histogram = require('prom-client').Histogram;

const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'status', 'path']
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'status', 'path']
});

module.exports = { register, httpRequestsTotal, httpRequestDuration };
```

### Database Exporters

Add exporters to the cluster:

PostgreSQL exporter:
```bash
kubectl run postgres-exporter \
  --image=prometheuscommunity/postgres-exporter \
  --env="DATA_SOURCE_NAME=postgresql://bmi_user:password@postgresql:5432/bmidb" \
  -n bmi-health
```

Redis exporter:
```bash
kubectl run redis-exporter \
  --image=oliver006/redis_exporter \
  --env="REDIS_ADDR=redis:6379" \
  -n bmi-health
```

## Log Collection with Loki

Logs are automatically collected from all pods in the `bmi-health` namespace. To view logs:

1. Open Grafana Explore
2. Select Loki datasource
3. Write a query, e.g.:
   - `{app="backend"}` — all backend logs
   - `{pod="postgresql"}` — PostgreSQL logs
   - `{job="kubernetes-pods", namespace="bmi-health"}` — all namespace logs

## Alerting & AlertManager

### PrometheusRule Alerts

Alerts are defined in `prometheus-rules.yaml` with:
- Conditions (e.g., error rate > 5%)
- Duration (e.g., for 5 minutes before firing)
- Labels and annotations (for routing and descriptions)

### Integrate with AlertManager

To use AlertManager, deploy it:

```bash
kubectl apply -f monitoring/alertmanager-config.yaml
```

Or configure Prometheus to send alerts to an external alerting system (PagerDuty, Slack, etc.).

### Test an Alert

Trigger the backend to fail temporarily:

```bash
# Get backend pod
BACKEND_POD=$(kubectl get pods -n bmi-health -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Kill the pod to trigger PodCrashLooping alert
kubectl delete pod $BACKEND_POD -n bmi-health
```

Watch the Prometheus Alerts tab — alert should fire within 1 minute.

## Best Practices

1. **Retention**: adjust `storage.tsdb.retention` in Prometheus deployment for longer history (default 15 days).
2. **Resource limits**: Prometheus and Loki are set to modest requests/limits; adjust based on scale.
3. **High availability**: deploy multiple Prometheus/Grafana replicas behind a load balancer for production.
4. **Authentication**: Grafana default password is weak; change immediately in production. Add OIDC or LDAP auth.
5. **Persistence**: emptyDir volumes are used; for production, use PersistentVolumes for Prometheus/Loki.
6. **Encryption**: secure Grafana datasource passwords using Kubernetes Secrets (mount as files).

## Troubleshooting

- **No metrics appearing**: Check `kubectl logs -n bmi-health <prometheus-pod>` for scrape errors.
- **Loki not collecting logs**: Verify Promtail is running on all nodes: `kubectl get daemonset -n bmi-health`.
- **Grafana not connecting to Prometheus**: test connectivity: `kubectl exec -n bmi-health <grafana-pod> -- curl http://prometheus:9090 -v`.

## Next Steps (Optional)

- Add `AlertManager` deployment and configure integrations (Slack, PagerDuty, email).
- Use Prometheus Operator for advanced ServiceMonitor / PrometheusRule CRDs.
- Integrate with kube-state-metrics for richer Kubernetes metadata.
- Add custom business metrics to the backend application.
- Set up log alerting rules (e.g., alert on ERROR logs).

If you'd like, I can:
- Create AlertManager + Slack/email integration examples
- Add kube-state-metrics Deployment
- Add backend prometheus client library examples
