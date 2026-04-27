---
name: gcp-monitoring
description: GCP observability — Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting, Profiler
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Monitoring & Observability Skill

Instrument, observe, and alert on GCP workloads using Cloud Operations Suite.

---

## Cloud Monitoring

### Metrics

GCP has three types of metrics:
- **System metrics** — auto-collected from GCP resources (CPU, memory, latency, etc.)
- **Agent metrics** — from Ops Agent installed on VMs (process-level, custom system metrics)
- **Custom metrics** — emitted from your application code

```bash
# List available metric types
gcloud monitoring metrics-scopes list

# Describe a metric
gcloud monitoring metric-descriptors describe \
  compute.googleapis.com/instance/cpu/utilization

# List metrics for a resource
gcloud monitoring time-series list \
  --filter='metric.type="compute.googleapis.com/instance/cpu/utilization"' \
  --interval-start-time=$(date -d '1 hour ago' --iso-8601=seconds) \
  --interval-end-time=$(date --iso-8601=seconds)
```

### Uptime Checks

```bash
# Create an HTTP uptime check
gcloud monitoring uptime create \
  --display-name="My Service Health" \
  --resource-type=uptime-url \
  --hostname=example.com \
  --path=/health \
  --port=443 \
  --use-ssl \
  --period=60 \
  --timeout=10

# List uptime checks
gcloud monitoring uptime list

# Delete an uptime check
gcloud monitoring uptime delete CHECK_ID
```

### Alerting Policies

```bash
# Create an alert policy (CPU > 80% for 5 minutes)
gcloud alpha monitoring policies create \
  --policy='{
    "displayName": "High CPU Alert",
    "conditions": [{
      "displayName": "CPU utilization > 80%",
      "conditionThreshold": {
        "filter": "resource.type=\"gce_instance\" AND metric.type=\"compute.googleapis.com/instance/cpu/utilization\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.8,
        "duration": "300s",
        "aggregations": [{
          "alignmentPeriod": "60s",
          "perSeriesAligner": "ALIGN_MEAN"
        }]
      }
    }],
    "notificationChannels": ["projects/my-project/notificationChannels/CHANNEL_ID"],
    "alertStrategy": {
      "autoClose": "604800s"
    }
  }'

# List alert policies
gcloud alpha monitoring policies list

# List notification channels
gcloud alpha monitoring channels list

# Create an email notification channel
gcloud alpha monitoring channels create \
  --display-name="Ops Team Email" \
  --type=email \
  --channel-labels=email_address=ops@example.com

# Create a PagerDuty channel
gcloud alpha monitoring channels create \
  --display-name="PagerDuty" \
  --type=pagerduty \
  --channel-labels=service_key=PD_SERVICE_KEY
```

### MQL (Monitoring Query Language)

MQL gives more power than the visual editor for complex metric queries.

```
# CPU utilization for a specific instance label
fetch gce_instance
| metric 'compute.googleapis.com/instance/cpu/utilization'
| filter metadata.user_labels.env == 'production'
| group_by 1m, [value_utilization_mean: mean(value.utilization)]
| every 1m

# Request rate for Cloud Run
fetch cloud_run_revision
| metric 'run.googleapis.com/request_count'
| filter resource.service_name == 'my-service'
| group_by 1m, [sum_count: sum(value.request_count)]
| every 1m

# Error rate as a percentage
fetch cloud_run_revision
| metric 'run.googleapis.com/request_count'
| filter resource.service_name == 'my-service'
| group_by 1m,
    [value_request_count_aggregate: aggregate(value.request_count)]
| every 1m
| {
    filter response_code_class == '5xx'
    | align delta(1m)
  ; ident}
| ratio
```

### Installing Ops Agent (VM metrics + logs)

```bash
# Install on a Debian/Ubuntu VM
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Check status
sudo systemctl status google-cloud-ops-agent

# Configure custom log collection in /etc/google-cloud-ops-agent/config.yaml
cat > /etc/google-cloud-ops-agent/config.yaml << 'EOF'
logging:
  receivers:
    my_app_logs:
      type: files
      include_paths:
        - /var/log/my-app/*.log
  service:
    pipelines:
      my_app_pipeline:
        receivers: [my_app_logs]
metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics]
EOF
sudo systemctl restart google-cloud-ops-agent
```

---

## Cloud Logging

### Log Explorer Queries

```bash
# Query logs via CLI
gcloud logging read \
  'resource.type="cloud_run_revision" AND severity>=WARNING' \
  --project=my-project \
  --limit=50 \
  --format=json

# Stream logs in real-time
gcloud logging tail \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"' \
  --project=my-project

# Read logs from a specific timeframe
gcloud logging read \
  'resource.type="gce_instance"' \
  --project=my-project \
  --freshness=1h \
  --limit=100

# Filter by severity
gcloud logging read 'severity>=ERROR' \
  --project=my-project \
  --limit=20
```

### Common Log Filter Syntax (Logging Query Language)

```
# Cloud Run errors
resource.type="cloud_run_revision"
resource.labels.service_name="my-service"
severity>=ERROR

# GKE container logs
resource.type="k8s_container"
resource.labels.cluster_name="my-cluster"
resource.labels.namespace_name="production"
resource.labels.container_name="my-app"

# Specific text search
resource.type="cloud_run_revision"
textPayload:"NullPointerException"

# Structured log field
resource.type="cloud_run_revision"
jsonPayload.user_id="12345"

# HTTP 5xx errors on LB
resource.type="http_load_balancer"
httpRequest.status>=500

# Audit log: who deleted a bucket
protoPayload.methodName="storage.buckets.delete"
protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog"

# Exclude health check noise
resource.type="cloud_run_revision"
NOT httpRequest.requestUrl="/health"
```

### Log Sinks (Routing)

```bash
# Export logs to BigQuery
gcloud logging sinks create bq-errors-sink \
  bigquery.googleapis.com/projects/my-project/datasets/logs \
  --log-filter='severity>=ERROR' \
  --project=my-project

# Export logs to GCS (for archiving)
gcloud logging sinks create gcs-audit-sink \
  storage.googleapis.com/my-audit-log-bucket \
  --log-filter='logName="projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"' \
  --project=my-project

# Export to Pub/Sub (for real-time processing)
gcloud logging sinks create pubsub-sink \
  pubsub.googleapis.com/projects/my-project/topics/log-events \
  --log-filter='severity>=WARNING' \
  --project=my-project

# Grant the sink writer access to destination
# After creating, get the writer identity:
gcloud logging sinks describe bq-errors-sink --format="value(writerIdentity)"
# Then grant that SA access to the BigQuery dataset / GCS bucket / Pub/Sub topic

# List sinks
gcloud logging sinks list

# Update a sink's filter
gcloud logging sinks update bq-errors-sink \
  --log-filter='severity>=ERROR AND resource.type="cloud_run_revision"'
```

### Log-Based Metrics

```bash
# Create a metric that counts ERROR logs
gcloud logging metrics create error-count \
  --description="Count of ERROR-severity log entries" \
  --log-filter='severity=ERROR' \
  --project=my-project

# Create a distribution metric (e.g., request latency from structured logs)
gcloud logging metrics create request-latency \
  --description="Request latency distribution" \
  --log-filter='resource.type="cloud_run_revision" AND jsonPayload.latency_ms!=""' \
  --value-extractor='EXTRACT(jsonPayload.latency_ms)' \
  --metric-kind=DELTA \
  --value-type=DISTRIBUTION \
  --project=my-project
```

---

## Cloud Trace

```bash
# Trace is SDK-based — install the client in your app
pip install google-cloud-trace opentelemetry-exporter-gcp-trace

# List traces
gcloud trace list \
  --project=my-project \
  --start-time=2024-01-01T00:00:00Z \
  --end-time=2024-01-02T00:00:00Z

# View a specific trace
gcloud trace describe TRACE_ID --project=my-project
```

**Python example:**
```python
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(CloudTraceSpanExporter())
)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("my-operation") as span:
    span.set_attribute("user.id", user_id)
    result = do_work()
```

---

## Error Reporting

Error Reporting automatically groups and tracks application errors. Works with
Cloud Logging — just log exceptions with a stack trace.

```bash
# View error groups via API
gcloud beta error-reporting events list \
  --service=my-service \
  --project=my-project

# Errors are also viewable in the console at:
# console.cloud.google.com/errors
```

**Enable auto-detection in Cloud Run/GKE:** Just log to stderr with a stack trace.
GCP automatically parses and groups them.

---

## Cloud Profiler

Continuous CPU/heap profiling with low overhead (<1%).

```bash
# Install in your Python app
pip install google-cloud-profiler

# Initialize at startup
import googlecloudprofiler
googlecloudprofiler.start(
    service='my-service',
    service_version='1.0.0',
    project_id='my-project',
)
```

---

## Dashboards (CLI)

```bash
# List dashboards
gcloud monitoring dashboards list

# Create a dashboard from JSON
gcloud monitoring dashboards create --config-from-file=dashboard.json

# Export a dashboard
gcloud monitoring dashboards describe DASHBOARD_ID --format=json > dashboard.json

# Delete a dashboard
gcloud monitoring dashboards delete DASHBOARD_ID
```

---

## Service Mesh Observability (Istio / Cloud Service Mesh)

When using Anthos Service Mesh or Istio on GKE:

```bash
# View service-level SLOs
gcloud monitoring services list --project=my-project

# Create an SLO (latency-based)
gcloud alpha monitoring services slos create \
  --service=SERVICE_ID \
  --slo-id=latency-slo \
  --display-name="99% of requests < 200ms" \
  --request-based \
  --good-total-ratio-metric=latencies_below_threshold/total_requests \
  --goal=0.99 \
  --rolling-period=86400s
```

---

## Guardrails

- **Log structured JSON, not plain text.** Cloud Logging indexes JSON fields — makes filtering and metrics much more powerful.
- **Set log exclusions for high-volume noise** (e.g., health check hits) before it inflates costs.
- **Always configure a log retention policy** — default is 30 days; audit logs default to 400 days.
- **Uptime checks need firewall allow rules** — check GCP's published health check IP ranges.
- **Alert on error rate, not error count** — a spike in traffic can cause false count alerts.
- **Sink writer SAs need explicit IAM grants** — creating a sink doesn't automatically grant access to the destination.
- **Cloud Trace sampling is NOT 100% by default** — configure sampling rate explicitly for production.
- **Log-based metrics can backfill from existing logs** — useful for retroactive SLI measurement.
