# OpenTelemetry Instrumentation Guide

How to add distributed tracing to the ai-assistant Python services using zero-code auto-instrumentation.

**Stack**: Python 3.12, FastAPI, uvicorn, uv, httpx
**Target**: OTel Collector at `otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4317`

---

## Step 1: Add OTel Packages

Add these to the `dependencies` list in each service's `pyproject.toml`:

```toml
dependencies = [
    # ... your existing deps ...
    "opentelemetry-distro>=0.48b0",
    "opentelemetry-exporter-otlp>=1.27.0",
    "opentelemetry-instrumentation-fastapi>=0.48b0",
    "opentelemetry-instrumentation-httpx>=0.48b0",
    "opentelemetry-instrumentation-requests>=0.48b0",
    "opentelemetry-instrumentation-logging>=0.48b0",
]
```

Then update the lockfile:

```bash
uv lock
```

### What Each Package Does

| Package | Purpose |
|---------|---------|
| `opentelemetry-distro` | Provides the `opentelemetry-instrument` CLI for zero-code auto-instrumentation |
| `opentelemetry-exporter-otlp` | Exports traces/metrics to the OTel Collector via OTLP protocol |
| `opentelemetry-instrumentation-fastapi` | Auto-instruments FastAPI routes (creates spans per request) |
| `opentelemetry-instrumentation-httpx` | Auto-instruments outbound httpx calls (traces inter-service requests) |
| `opentelemetry-instrumentation-requests` | Auto-instruments outbound requests library calls |
| `opentelemetry-instrumentation-logging` | Injects trace_id/span_id into Python log records |

---

## Step 2: Modify Dockerfile Entrypoint

Change your `CMD` from:

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

To:

```dockerfile
CMD ["opentelemetry-instrument", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

That's it. The `opentelemetry-instrument` wrapper automatically discovers and activates all installed instrumentors (FastAPI, httpx, requests, logging) without any code changes.

---

## Step 3: Add Environment Variables to Kubernetes Deployments

Add these env vars to each deployment's container spec:

```yaml
env:
  # ... your existing env vars ...
  - name: OTEL_SERVICE_NAME
    value: "assistant-gateway"  # change per service (see table below)
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.namespace=ai-assistant,deployment.environment=production"
  - name: OTEL_LOGS_EXPORTER
    value: "none"
  - name: OTEL_PYTHON_LOG_CORRELATION
    value: "true"
```

### Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `OTEL_SERVICE_NAME` | Service-specific (see below) | Shows up in Tempo's service name dropdown |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector-...monitoring...:4317` | Where to send traces |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | Protocol for OTLP export |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.namespace=ai-assistant,...` | Extra metadata attached to all spans |
| `OTEL_LOGS_EXPORTER` | `none` | Don't send logs via OTel (Alloy already collects stdout) |
| `OTEL_PYTHON_LOG_CORRELATION` | `true` | Injects trace_id/span_id into Python log records |

### Service Names

Use these `OTEL_SERVICE_NAME` values per deployment:

| Deployment | OTEL_SERVICE_NAME |
|------------|-------------------|
| assistant-gateway | `assistant-gateway` |
| llm-service | `llm-service` |
| memory-store-service | `memory-store-service` |
| memory-agent-service | `memory-agent-service` |
| agent-memory-service | `agent-memory-service` |
| character-service | `character-service` |
| user-memory-service | `user-memory-service` |
| stt-service | `stt-service` |
| tts-service | `tts-service` |

---

## Step 4: Log Format for Trace-to-Log Linking

Your services already output JSON logs (`LOG_FORMAT=json`). With `OTEL_PYTHON_LOG_CORRELATION=true`, the Python `logging` module will have `otelTraceID` and `otelSpanID` attributes on each log record.

Your JSON formatter needs to include these fields. Example:

```python
import logging
import json

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "service": record.name,
        }
        # Injected by opentelemetry-instrumentation-logging
        if hasattr(record, "otelTraceID") and record.otelTraceID != "0":
            log_data["trace_id"] = record.otelTraceID
            log_data["span_id"] = record.otelSpanID
        return json.dumps(log_data)
```

The key requirement is that `trace_id=<value>` appears somewhere in the log line. The Grafana Loki datasource has a derived field regex `trace_id=(\w+)` that creates clickable links to Tempo. A JSON field like `"trace_id": "abc123"` matches because the raw line contains `trace_id`.

---

## Rollout Order

Start with the gateway and work inward:

1. **assistant-gateway** — the entry point; gives you traces for every inbound request
2. **llm-service** — traces LLM API calls and latency
3. **memory-store-service** — traces memory read/write operations

Once 2+ services are instrumented, the **service map** in Grafana (Explore > Tempo > Service Graph) will show the request flow between them.

### Services You Don't Need to Instrument

- **postgres, redis, qdrant** — third-party services need their own instrumentation
- **n8n, open-webui** — separate applications with their own stacks
- **casual-mcp-servers** — third-party MCP server images

However, `opentelemetry-instrumentation-httpx` will automatically create client spans for outbound calls from your services to these backends. You'll see the latency of those calls in traces even without instrumenting the backends.

---

## Verification

After deploying an instrumented service:

1. Send a request to the service (e.g., make a chat request through the gateway)
2. Open Grafana at `http://192.168.2.44`
3. Go to **Explore** > select **Tempo** datasource
4. Use the **Search** tab
5. Set **Service Name** to `assistant-gateway`
6. Click **Run query**
7. You should see traces appear within a few seconds
8. Click a trace to see the waterfall:
   - FastAPI handler span (the inbound request)
   - httpx client spans (outbound calls to llm-service, memory-store-service, etc.)
   - Each span shows duration, status code, and attributes

### Test Trace-to-Log Linking

1. In a trace, click on a span
2. Click **Logs for this span** — should open Loki filtered to that trace
3. In Loki, look for `trace_id` links on log lines — clicking should jump back to Tempo

### Test Service Map

1. Go to **Explore > Tempo > Service Graph** tab
2. After instrumenting 2+ services with traffic between them, you'll see nodes and edges

---

## Troubleshooting

### No Traces Appearing

1. Check the service logs for OTel errors:
   ```bash
   kubectl logs -n ai-assistant deploy/assistant-gateway | grep -i otel
   ```

2. Verify the OTel Collector is running:
   ```bash
   kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-collector
   ```

3. Check OTel Collector logs for received spans:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-collector | grep -i span
   ```

4. Test connectivity from the app pod to the collector:
   ```bash
   kubectl exec -n ai-assistant deploy/assistant-gateway -- \
     python -c "import socket; s=socket.socket(); s.settimeout(3); s.connect(('otel-collector-opentelemetry-collector.monitoring.svc.cluster.local', 4317)); print('OK'); s.close()"
   ```

### Traces Appear but No Log Linking

- Verify `OTEL_PYTHON_LOG_CORRELATION=true` is set
- Check that your JSON formatter includes `trace_id` in the output
- Check a raw log line in Loki — search for `trace_id` in the text

### Service Map is Empty

- Need 2+ instrumented services with actual traffic between them
- The gateway must make httpx/requests calls to another instrumented service
- Wait a few minutes for Tempo's metrics generator to produce service graph data
