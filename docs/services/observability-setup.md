# Observability Stack Setup

This document describes how to deploy the **General Observability Stack** to the K3s cluster.

The stack provides centralized logging, metrics, and distributed tracing for all applications.

## Stack Components

- **Loki** - Log aggregation and storage
- **Prometheus** - Metrics collection and storage
- **Tempo** - Distributed tracing storage
- **Grafana** - Unified visualization dashboard
- **OpenTelemetry Collector** - Telemetry data receiver and processor
- **Alloy** - Container log collection agent

## Prerequisites

- Running K3s cluster (master + workers)
- **Longhorn** installed and healthy (storage backend)
- **MetalLB** installed and configured (for LoadBalancer support)
- **kubectl** configured on master node
- **Helm 3** installed

## Deployment Overview

The observability stack is deployed using official Helm charts with custom values files.

**Namespace**: `monitoring`
**Grafana URL**: `http://192.168.2.44` (MetalLB LoadBalancer)
**Initial Admin Password**: `changeme123` (change after first login)
**Retention**: Logs 30 days, Metrics 15 days, Traces 7 days

---

## Phase 0: Add Helm Repositories

Add the required Helm chart repositories:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

---

## Phase 1: Create Namespace and Secrets

Create the monitoring namespace:

```bash
kubectl apply -f manifests/monitoring/namespace.yaml
```

Create the Grafana admin password secret:

```bash
kubectl apply -f manifests/monitoring/grafana-admin-secret.yaml
```

Verify:

```bash
kubectl get namespace monitoring
kubectl get secret -n monitoring grafana-admin-secret
```

---

## Phase 2: Deploy Loki (Log Storage)

Deploy Loki for log aggregation:

```bash
helm install loki grafana/loki \
  -f helm/loki/loki-values.yaml \
  --namespace monitoring
```

Verify deployment:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
kubectl get pvc -n monitoring
```

Expected output:
- Loki pod in `Running` state
- PVC `storage-loki-0` (20Gi) in `Bound` state with Longhorn StorageClass

Test Loki health:

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
curl http://localhost:3100/ready
# Should return "ready"
```

---

## Phase 3: Deploy Prometheus (Metrics Storage)

Deploy Prometheus for metrics collection:

```bash
helm install prometheus prometheus-community/prometheus \
  -f helm/prometheus/prometheus-values.yaml \
  --namespace monitoring
```

Verify deployment:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
kubectl get pvc -n monitoring | grep prometheus
```

Expected output:
- Prometheus server pod in `Running` state
- PVC `prometheus-server` (50Gi) in `Bound` state

Test Prometheus UI:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
# Open http://localhost:9090
# Check Status → Targets to see discovered endpoints
```

### Storage Optimization

The Prometheus configuration includes optimizations for home lab use:

**Retention**: 15 days (sufficient for troubleshooting, reduces storage)

**Dropped high-cardinality metrics**: Histogram bucket metrics from Kubernetes internals are dropped via `metric_relabel_configs` on both the `kubernetes-apiservers` and `kubernetes-nodes` scrape jobs. These are only needed for percentile calculations (p50, p90, p99) which are overkill for a home lab. The drop rules use broad patterns:

- `(apiserver|etcd)_.+_bucket` - all API server and etcd histogram buckets
- `workqueue_.+_bucket` - all workqueue histogram buckets
- `(scheduler|storage)_.+_bucket` - all scheduler and storage histogram buckets

The `_sum` and `_count` variants are still collected, so you can calculate average latencies.

**Note**: Drop rules must be applied to both `kubernetes-apiservers` and `kubernetes-nodes` jobs, as the kubelet proxies API server metrics on each node.

---

## Phase 4: Deploy Tempo (Trace Storage)

Deploy Tempo for distributed tracing:

```bash
helm install tempo grafana/tempo \
  -f helm/tempo/tempo-values.yaml \
  --namespace monitoring
```

Verify deployment:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=tempo
kubectl get pvc -n monitoring | grep tempo
```

Expected output:
- Tempo pod in `Running` state
- PVC `storage-tempo-0` (10Gi) in `Bound` state

Test Tempo health:

```bash
kubectl port-forward -n monitoring svc/tempo 3200:3200
curl http://localhost:3200/ready
# Should return "ready"
```

---

## Phase 5: Deploy Alloy (Log Collector)

First, create the Alloy configuration ConfigMap:

```bash
kubectl apply -f helm/alloy/alloy-config.yaml
```

Then deploy Alloy as a DaemonSet:

```bash
helm install alloy grafana/alloy \
  -f helm/alloy/alloy-values.yaml \
  --namespace monitoring
```

Verify deployment:

```bash
kubectl get daemonset -n monitoring alloy
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
```

Expected output:
- 1 Alloy pod per node (DaemonSet)
- All pods in `Running` state

Check Alloy logs:

```bash
kubectl logs -n monitoring daemonset/alloy --tail=50
# Should see "component started" messages
```

---

## Phase 6: Deploy OpenTelemetry Collector

Deploy the OTel Collector for telemetry processing:

```bash
helm install otel-collector open-telemetry/opentelemetry-collector \
  -f helm/otel-collector/otel-collector-values.yaml \
  --namespace monitoring
```

Verify deployment:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-collector
kubectl get svc -n monitoring otel-collector-opentelemetry-collector
```

Expected output:
- OTel Collector pod in `Running` state
- Service exposing ports 4317 (OTLP gRPC), 4318 (OTLP HTTP), 8888 (metrics)

Test OTel Collector:

```bash
kubectl port-forward -n monitoring svc/otel-collector-opentelemetry-collector 8888:8888
curl http://localhost:8888/metrics
# Should return Prometheus metrics
```

---

## Phase 7: Deploy Grafana (Visualization)

Deploy Grafana with pre-configured datasources:

```bash
helm install grafana grafana/grafana \
  -f helm/grafana/grafana-values.yaml \
  --namespace monitoring
```

Verify deployment:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
kubectl get svc -n monitoring grafana
kubectl get pvc -n monitoring | grep grafana
```

Expected output:
- Grafana pod in `Running` state
- Service type `LoadBalancer` with EXTERNAL-IP `192.168.2.44`
- PVC `grafana` (5Gi) in `Bound` state

Access Grafana:

```bash
# Open in browser: http://192.168.2.44
# Username: admin
# Password: changeme123
```

**IMPORTANT**: Change the admin password after first login!

---

## Phase 8: Verification

### 1. Infrastructure Health Check

```bash
kubectl get pods -n monitoring
kubectl get pvc -n monitoring
kubectl get svc -n monitoring
```

All pods should be `Running`, all PVCs should be `Bound` with Longhorn.

### 2. Grafana Datasource Verification

1. Login to Grafana at `http://192.168.2.44`
2. Go to **Configuration → Data Sources**
3. Verify all three datasources show **green checkmarks**:
   - Loki
   - Prometheus (default)
   - Tempo

### 3. Log Ingestion Test

Create a test pod that generates logs:

```bash
kubectl run test-logger --image=busybox --restart=Never -- \
  sh -c "for i in \$(seq 1 10); do echo 'Test log message #'\$i; sleep 1; done"
```

Wait 30 seconds for Alloy to collect logs, then query in Grafana:

1. Go to **Explore**
2. Select **Loki** datasource
3. Query: `{namespace="default", pod="test-logger"}`
4. You should see 10 log lines

Clean up:

```bash
kubectl delete pod test-logger
```

### 4. Metrics Scraping Test

Check Prometheus targets:

1. Port-forward Prometheus: `kubectl port-forward -n monitoring svc/prometheus-server 9090:80`
2. Open `http://localhost:9090/targets`
3. Verify targets are UP:
   - `prometheus` (self-monitoring)
   - `otel-collector` (meta-monitoring)
   - Any pods with `prometheus.io/scrape: "true"` annotation

### 5. Trace Ingestion Test (OpenTelemetry Demo App)

Deploy the OTel demo application:

```bash
kubectl create namespace otel-demo
kubectl apply -n otel-demo -f https://raw.githubusercontent.com/open-telemetry/opentelemetry-demo/main/kubernetes/opentelemetry-demo.yaml
```

**Note**: Edit the demo app's OTel Collector endpoint to point to `otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4317`

Generate some traffic, then check traces in Grafana:

1. Go to **Explore**
2. Select **Tempo** datasource
3. Query Type: **Search**
4. You should see traces from the demo app

Clean up (optional):

```bash
kubectl delete namespace otel-demo
```

### 6. Meta-Monitoring Test

Verify the observability stack monitors itself:

**Logs**: Query monitoring namespace logs
```logql
{namespace="monitoring"}
```

**Metrics**: Query OTel Collector metrics in Prometheus
```promql
up{job="otel-collector"}
```

---

## Production Readiness Checklist

Before marking the stack as production-ready, complete this checklist:

- [ ] All pods in `monitoring` namespace are Running
- [ ] All PVCs are Bound with Longhorn StorageClass
- [ ] Grafana accessible at `http://192.168.2.44`
- [ ] Grafana admin password changed from default
- [ ] All 3 datasources (Loki, Prometheus, Tempo) show green checkmarks
- [ ] Test log query returns results
- [ ] Prometheus targets page shows discovered endpoints
- [ ] OTel Collector accepting OTLP traffic
- [ ] Alloy has 1 pod per node (all Running)
- [ ] Meta-monitoring works (can query monitoring namespace logs/metrics)

---

## Service Endpoints (for Applications)

Applications should send telemetry to these endpoints:

- **Loki** (internal): `http://loki.monitoring.svc.cluster.local:3100`
- **Prometheus** (internal): `http://prometheus-server.monitoring.svc.cluster.local:80`
- **Tempo** (internal): `http://tempo.monitoring.svc.cluster.local:3200`
- **OTel Collector** (gRPC): `http://otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4317`
- **OTel Collector** (HTTP): `http://otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4318`
- **Grafana** (external): `http://192.168.2.44`

---

## Application Instrumentation

### Logs (Automatic)

Alloy automatically collects all container stdout/stderr logs. No application changes required.

**Best Practice**: Output structured JSON logs:

```json
{"timestamp": "2025-01-26T10:00:00Z", "level": "INFO", "service": "my-service", "trace_id": "abc123", "message": "Request completed"}
```

### Metrics (Manual)

Expose a `/metrics` endpoint in Prometheus format and add annotations to your Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "80"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: my-service
  ports:
    - port: 80
```

### Traces (Manual)

Send traces to OpenTelemetry Collector:

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Configure tracer
trace.set_tracer_provider(TracerProvider())
otlp_exporter = OTLPSpanExporter(
    endpoint="http://otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4317",
    insecure=True
)
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(otlp_exporter))
```

---

## Troubleshooting

### Loki Not Receiving Logs

**Symptoms**: Grafana shows "No logs found"

**Checks**:
1. `kubectl get daemonset -n monitoring alloy` - Verify Alloy is running on all nodes
2. `kubectl logs -n monitoring daemonset/alloy` - Check Alloy logs for errors
3. `kubectl exec -n monitoring daemonset/alloy -- wget -qO- http://loki:3100/ready` - Test Loki connectivity

### Prometheus Not Scraping Metrics

**Symptoms**: No metrics appearing

**Checks**:
1. Visit Prometheus targets page: `http://localhost:9090/targets` (via port-forward)
2. Verify service has correct annotations (`prometheus.io/scrape: "true"`)
3. Check if `/metrics` endpoint is accessible: `kubectl exec -n <namespace> <pod> -- wget -O- http://localhost:<port>/metrics`

### Tempo Not Showing Traces

**Symptoms**: No traces in Grafana

**Checks**:
1. `kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-collector` - Check OTel Collector logs
2. Verify application is sending traces to `otel-collector-opentelemetry-collector:4317`
3. Test OTLP endpoint: `kubectl exec -n <namespace> <pod> -- nc -zv otel-collector-opentelemetry-collector.monitoring.svc.cluster.local 4317`

### Grafana LoadBalancer Pending

**Symptoms**: Grafana service stuck in `Pending` state

**Checks**:
1. `kubectl get svc -n monitoring grafana` - Check service status
2. `kubectl -n metallb-system get pods` - Verify MetalLB is running
3. Verify IP `192.168.2.44` is available in MetalLB pool and not in use

### Alloy "too many open files" Errors

**Symptoms**: Alloy logs show `failed to create fsnotify watcher: too many open files`

**Cause**: Alloy uses inotify file watchers to tail logs from all pods across the cluster. The default Linux inotify limits are too low for large clusters.

**Solution**: Increase inotify limits on **all cluster nodes** (master + workers):

```bash
# On each node, increase inotify limits
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512

# Make persistent across reboots
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
```

After applying on all nodes, restart Alloy:

```bash
kubectl rollout restart daemonset -n monitoring alloy
```

**Note**: Log collection will still work despite these errors, but Alloy may struggle to watch all log files simultaneously until the limits are increased.

---

## Upgrading Components

To upgrade a component to the latest version:

```bash
# Update Helm repos
helm repo update

# Upgrade component
helm upgrade <release-name> <chart> \
  -f helm/<component>/<component>-values.yaml \
  --namespace monitoring
```

Example:

```bash
helm upgrade grafana grafana/grafana \
  -f helm/grafana/grafana-values.yaml \
  --namespace monitoring
```

---

## Uninstalling

To remove the entire observability stack:

```bash
# Uninstall Helm releases
helm uninstall grafana -n monitoring
helm uninstall otel-collector -n monitoring
helm uninstall alloy -n monitoring
helm uninstall tempo -n monitoring
helm uninstall prometheus -n monitoring
helm uninstall loki -n monitoring

# Delete PVCs (if you want to remove data)
kubectl delete pvc -n monitoring --all

# Delete namespace
kubectl delete namespace monitoring
```

**Warning**: This will delete all logs, metrics, and traces permanently!

---

## Next Steps

1. **Change Grafana Password**: Login and change from `changeme123`
2. **Create Dashboards**: Build custom dashboards for your applications
3. **Configure Alerts**: Set up Prometheus alerting rules
4. **Instrument Applications**: Add metrics and trace instrumentation to your services
5. **Backup Dashboards**: Export dashboards as JSON and commit to git

---

## References

- Grafana Loki: https://grafana.com/docs/loki/latest/
- Prometheus: https://prometheus.io/docs/
- Tempo: https://grafana.com/docs/tempo/latest/
- OpenTelemetry: https://opentelemetry.io/docs/
- Grafana Alloy: https://grafana.com/docs/alloy/latest/
