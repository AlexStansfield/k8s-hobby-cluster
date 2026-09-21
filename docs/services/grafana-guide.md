# Grafana Usage Guide

This guide covers how to use Grafana for traces, dashboards, alerting, and more. Grafana is the unified frontend for the observability stack.

**URL**: `http://192.168.2.44`
**Datasources**: Prometheus (metrics), Loki (logs), Tempo (traces)

## Table of Contents

- [1. Navigating Grafana](#1-navigating-grafana)
- [2. Viewing Traces with Tempo](#2-viewing-traces-with-tempo)
- [3. Trace-to-Log Correlation](#3-trace-to-log-correlation)
- [4. Service Map](#4-service-map)
- [5. Dashboards](#5-dashboards)
- [6. Alerting](#6-alerting)
- [7. Other Observability Features](#7-other-observability-features)
- [8. Query Quick Reference](#8-query-quick-reference)

---

## 1. Navigating Grafana

### Key Sections (Left Sidebar)

- **Dashboards** - Saved dashboards and folders
- **Explore** - Ad-hoc querying against any datasource (this is where you'll spend most time investigating)
- **Alerting** - Alert rules, contact points, notification policies
- **Connections > Data sources** - Verify datasource health (should show green checkmarks)

### Explore View

The Explore view is the main tool for investigating traces, logs, and metrics:

1. Select a datasource from the top-left dropdown (Prometheus, Loki, or Tempo)
2. Each datasource has its own query editor (PromQL, LogQL, TraceQL)
3. Use the time range picker in the top-right
4. Click the **Split** button (top-right) for side-by-side comparison of two datasources

---

## 2. Viewing Traces with Tempo

### Prerequisites

Your applications must be instrumented with OpenTelemetry to send traces. Traces are sent to the OTel Collector:

- **gRPC**: `otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4317`
- **HTTP**: `otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4318`

See the "Application Instrumentation" section in [observability-setup.md](observability-setup.md) for code examples.

### Searching for Traces

1. Go to **Explore**
2. Select **Tempo** datasource
3. Use the **Search** tab (default)
4. Filter by:
   - **Service Name** - dropdown of instrumented services
   - **Span Name** - specific operations (e.g., `GET /api/health`)
   - **Duration** - min/max to find slow requests
   - **Tags** - custom span attributes (e.g., `http.status_code=500`)
5. Click **Run query**
6. Results show a table: Trace ID, root service, root span, duration, start time
7. Click any trace to open the detail view

### Reading the Trace Detail View

- **Trace timeline**: waterfall visualization showing the span hierarchy
- Each horizontal bar = one span (an operation within the trace)
- Nested/indented spans show parent-child relationships (e.g., an HTTP handler calling a database query)
- Click a span to see:
  - **Span attributes** - HTTP method, status code, URL, etc.
  - **Resource attributes** - namespace, pod, container, node
  - **Events** - logs or errors attached to the span
  - **Duration** - how long this specific operation took

### TraceQL Queries

TraceQL is Tempo's query language for advanced trace search:

```traceql
# All traces from a specific service
{ resource.service.name = "my-service" }

# Slow traces (over 2 seconds)
{ duration > 2s }

# Error traces only
{ status = error }

# Traces with HTTP 500 responses
{ span.http.status_code = 500 }

# Traces from a specific namespace
{ resource.k8s.namespace.name = "ai-assistant" }

# Combine filters
{ resource.k8s.namespace.name = "ai-assistant" && status = error }
```

---

## 3. Trace-to-Log Correlation

The stack is wired so you can jump between traces and logs seamlessly.

### Trace to Logs (Tempo -> Loki)

When viewing a trace in Tempo:

1. Click on a specific span
2. Click the **Logs for this span** button in the span detail panel
3. Grafana opens a split view with Loki pre-filtered by:
   - The trace ID
   - The namespace, pod, and container from span attributes
4. You see the exact application logs that correspond to that trace

This works because the Tempo datasource has `tracesToLogs` configured with Loki, filtering by `namespace`, `pod`, and `container` tags.

### Logs to Trace (Loki -> Tempo)

When viewing logs in Loki:

1. Query logs in **Explore > Loki**
2. If log lines contain `trace_id=<value>`, a blue **TraceID** link appears next to the log line
3. Click it to jump directly to the full trace in Tempo

**Requirement**: Your application must include the trace ID in log output. The configured regex matches `trace_id=(\w+)`. Example log line:

```
INFO Request completed trace_id=abc123def456 method=GET path=/api/health status=200
```

Or as structured JSON:

```json
{"level": "INFO", "message": "Request completed", "trace_id": "abc123def456", "method": "GET"}
```

### Common Trace ID Formats

Different frameworks output trace IDs differently. The current regex `trace_id=(\w+)` matches:
- `trace_id=abc123def456` (key=value format)

If your framework uses a different format (e.g., `traceId`, `trace-id`, `TraceID`), you can update the regex in `helm/grafana/grafana-values.yaml` under the Loki datasource's `derivedFields` section.

---

## 4. Service Map

The service map shows a visual graph of how services communicate with each other.

### Accessing the Service Map

1. Go to **Explore > Tempo**
2. Click the **Service Graph** tab (next to Search and TraceQL)
3. You'll see a node graph:
   - **Nodes** = services
   - **Edges** = requests between services
   - Node size can indicate request rate
   - Edge colors can indicate error rates
4. Click a node to drill into traces for that service

### Prerequisites

- Requires **at least 2 instrumented services** communicating with each other for edges to appear
- A single service will show as a single node with no connections
- Powered by Tempo's `metrics_generator` (already enabled in `tempo-values.yaml`)
- Metrics are written to Prometheus, which Grafana reads for the graph

### Node Graph in Trace View

When viewing an individual trace, you can also see a node graph showing the service dependency for that specific trace. This is enabled via the `nodeGraph.enabled: true` setting in the Tempo datasource.

---

## 5. Dashboards

### Pre-Provisioned Dashboards

The following community dashboards are provisioned automatically via Helm:

| Dashboard | Description | Grafana.net ID |
|-----------|-------------|----------------|
| Node Exporter Full | Per-node CPU, memory, disk, network | 1860 |
| K8s Cluster Overview | Cluster-wide CPU, memory, pod counts | 15282 |
| K8s Namespace Resources | Per-namespace resource usage breakdown | 15826 |
| K8s Pod Metrics | Per-pod CPU, memory, network | 17149 |
| Longhorn | Storage volume health and usage | 16888 |

Find these in **Dashboards** (left sidebar) after deploying.

**Note on Longhorn dashboard**: Requires Longhorn metrics to be scraped by Prometheus. Longhorn exposes metrics by default on its manager pods. If the dashboard shows "No data", you may need to add a Prometheus scrape annotation to the Longhorn manager service or add a static scrape target in `helm/prometheus/prometheus-values.yaml`.

### Importing Additional Dashboards via UI

1. Go to **Dashboards** (left sidebar)
2. Click **New** > **Import**
3. Enter a Grafana.net dashboard ID (e.g., `1860`)
4. Click **Load**
5. Select **Prometheus** as the datasource
6. Click **Import**

Browse community dashboards at: https://grafana.com/grafana/dashboards/

Some other dashboards to consider:
- **CoreDNS** (ID: 15463) - DNS query monitoring
- **Traefik** (ID: 17346) - Ingress controller metrics (requires Traefik metrics enabled)

### Provisioning Dashboards as Code (Helm)

Add entries to the `dashboards` section in `helm/grafana/grafana-values.yaml`:

```yaml
dashboards:
  default:
    my-dashboard:
      gnetId: 12345
      revision: 1
      datasource: Prometheus
```

Then upgrade Grafana:

```bash
helm upgrade grafana grafana/grafana \
  -f helm/grafana/grafana-values.yaml \
  --namespace monitoring
```

### Provisioning via ConfigMap Sidecar

The Grafana sidecar watches for ConfigMaps with the label `grafana_dashboard: "1"`:

1. Export a dashboard from Grafana UI: **Share > Export > Save to file**
2. Create a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-custom-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  my-dashboard.json: |
    { ... dashboard JSON ... }
```

3. Apply it:

```bash
kubectl apply -f my-custom-dashboard.yaml
```

Grafana picks it up automatically (no restart needed).

---

## 6. Alerting

Grafana unified alerting is enabled. You can create alert rules that evaluate PromQL or LogQL queries and send notifications.

### Creating an Alert Rule

1. Go to **Alerting** (left sidebar) > **Alert rules**
2. Click **New alert rule**
3. Configure:
   - **Rule name**: descriptive name (e.g., "High Node Memory")
   - **Query**: select datasource and write the query
   - **Condition**: set threshold (e.g., IS ABOVE 0.85)
   - **Evaluation**: choose folder, evaluation group, and interval (e.g., every 1m, for 5m)
   - **Labels**: add `severity: warning` or `severity: critical`
   - **Annotations**: add summary and description text

### Example Alert Rules

**Pod CrashLooping** (Prometheus):
```promql
increase(kube_pod_container_status_restarts_total[1h]) > 3
```
Fires when any pod restarts more than 3 times in an hour.

**High Node Memory** (Prometheus):
```promql
(1 - node_memory_AvailableBytes / node_memory_MemTotalBytes) > 0.85
```
Fires when any node exceeds 85% memory usage.

**Node Down** (Prometheus):
```promql
up{job="kubernetes-nodes"} == 0
```
Fires when a node stops responding.

**Disk Pressure on /mnt/data** (Prometheus):
```promql
(1 - node_filesystem_avail_bytes{mountpoint="/mnt/data"} / node_filesystem_size_bytes{mountpoint="/mnt/data"}) > 0.85
```
Fires when `/mnt/data` exceeds 85% usage on any node.

**Error Log Spike** (Loki):
```logql
sum(rate({namespace="ai-assistant"} |= "ERROR" [5m])) > 1
```
Fires when error logs exceed 1 per second over 5 minutes.

### Contact Points

Set up where alerts are delivered:

1. Go to **Alerting > Contact points**
2. Click **New contact point**
3. Choose a type: Email, Slack, Discord, Webhook, PagerDuty, etc.
4. For a home lab, **Webhook** or **Discord** are practical choices

### Notification Policies

Define routing rules for different alert severities:

1. Go to **Alerting > Notification policies**
2. Set the **default contact point** (catches all alerts)
3. Add matchers for specific routing (e.g., `severity=critical` goes to a different channel)

---

## 7. Other Observability Features

### Log-Based Metrics

Create metrics from log patterns without changing application code. Use LogQL metric queries in dashboard panels or alert rules:

```logql
# Count errors per app over time
sum by (app) (rate({namespace="ai-assistant"} |= "ERROR" [5m]))

# Parse JSON logs and count by level
sum by (level) (rate({namespace="ai-assistant"} | json | __error__="" [5m]))

# Count HTTP 5xx from logs using regex
sum(rate({namespace="ai-assistant"} |~ "status=(5\\d{2})" [5m]))
```

### Annotations

Mark deployments, incidents, or other events on dashboard graphs:

**Via UI**: In a dashboard panel, click the time axis area > **Add annotation** > enter a description.

**Via API**:
```bash
curl -X POST http://192.168.2.44/api/annotations \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"text":"Deployed v2.1.0","tags":["deployment"]}'
```

Create an API key in **Administration > Service accounts**.

### Recording Rules

Pre-compute expensive Prometheus queries that you use frequently in dashboards. Add to `helm/prometheus/prometheus-values.yaml` under `serverFiles`:

```yaml
recording_rules.yml:
  groups:
    - name: custom_rules
      rules:
        - record: namespace:pod_cpu_usage:sum
          expr: sum by (namespace) (rate(container_cpu_usage_seconds_total[5m]))
        - record: namespace:memory_usage:sum
          expr: sum by (namespace) (container_memory_working_set_bytes{container!=""})
```

Then reference `namespace:pod_cpu_usage:sum` in dashboards instead of the full expression.

### SLO Tracking

Track service level objectives using error rate and latency metrics from instrumented services:

**Error rate** (availability SLO):
```promql
# Percentage of non-5xx requests over the last hour
sum(rate(http_requests_total{status!~"5.."}[1h])) / sum(rate(http_requests_total[1h]))
```

**Latency** (performance SLO, requires histogram metrics):
```promql
# p99 request duration
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

---

## 8. Query Quick Reference

### PromQL (Prometheus)

```promql
# Cluster CPU usage percentage
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) / count(node_cpu_seconds_total{mode="idle"}) * 100

# Memory usage per node (percentage)
(1 - node_memory_AvailableBytes / node_memory_MemTotalBytes) * 100

# Pod CPU usage (cores)
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace, pod)

# Pod memory usage (bytes)
sum(container_memory_working_set_bytes{container!=""}) by (namespace, pod)

# Pod restart count
kube_pod_container_status_restarts_total

# Longhorn volume usage percentage
longhorn_volume_actual_size_bytes / longhorn_volume_capacity_bytes * 100

# Total pods by namespace
count by (namespace) (kube_pod_info)

# OTel Collector throughput
rate(otelcol_receiver_accepted_spans[5m])
```

### LogQL (Loki)

```logql
# All logs from a namespace
{namespace="ai-assistant"}

# Filter by pod name pattern and search for errors
{namespace="ai-assistant", pod=~"my-app.*"} |= "error"

# Case-insensitive error search
{namespace="ai-assistant"} |~ "(?i)error|exception|fatal"

# Parse JSON logs and filter by level
{namespace="ai-assistant"} | json | level="ERROR"

# Log volume over time (for dashboards)
sum by (namespace) (rate({namespace=~".+"} [5m]))

# Top 10 most frequent log lines
topk(10, sum by (message) (rate({namespace="ai-assistant"} | json [5m])))
```

### TraceQL (Tempo)

```traceql
# All traces from a service
{ resource.service.name = "my-service" }

# Slow traces
{ duration > 2s }

# Error traces
{ status = error }

# Specific HTTP status code
{ span.http.status_code = 500 }

# Traces from a namespace
{ resource.k8s.namespace.name = "ai-assistant" }

# Combined filters
{ resource.service.name = "my-service" && duration > 1s && status = error }
```

---

## References

- Grafana docs: https://grafana.com/docs/grafana/latest/
- PromQL: https://prometheus.io/docs/prometheus/latest/querying/basics/
- LogQL: https://grafana.com/docs/loki/latest/query/
- TraceQL: https://grafana.com/docs/tempo/latest/traceql/
- Grafana alerting: https://grafana.com/docs/grafana/latest/alerting/
- Community dashboards: https://grafana.com/grafana/dashboards/
