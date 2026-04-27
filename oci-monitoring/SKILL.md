---
name: oci-monitoring
description: OCI observability — Monitoring metrics/MQL/alarms, Logging, Logging Analytics, Events, Notifications, Service Connector, APM
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools:
  - shell
  - read_file
  - write_file
  - glob
  - grep
---

# OCI Monitoring & Observability

## Monitoring Service (Metrics & Alarms)

OCI Monitoring uses **Metrics Query Language (MQL)** to query time-series metrics emitted by OCI services and custom sources.

### Querying Metrics (MQL Basics)

```bash
# Query a metric via CLI
oci monitoring metric-data summarize-metrics-data \
  --compartment-id $C \
  --query-text 'CpuUtilization[5m].mean()' \
  --namespace "oci_computeagent" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')"

# Filter by dimension (specific instance)
oci monitoring metric-data summarize-metrics-data \
  --compartment-id $C \
  --query-text 'CpuUtilization[5m]{resourceId = "'$INSTANCE_OCID'"}.mean()' \
  --namespace "oci_computeagent"
```

### MQL Reference

```
# Basic structure: MetricName[interval]{filter}.statistic()
CpuUtilization[5m].mean()
MemoryUtilization[1m].max()
NetworksBytesIn[5m].sum()

# Grouping (aggregate across all instances in compartment)
CpuUtilization[5m].grouping().mean()

# Percentile
RequestLatency[1m].percentile(95)

# Rate of change
DiskWriteBytes[5m].rate()

# Math operations
(BytesIn[5m].sum() + BytesOut[5m].sum()) / 1048576   # Total MB/s

# Filter by tag
CpuUtilization[5m]{freeformTags.Environment = "prod"}.mean()

# Absence detection (no data in last 5 min)
CpuUtilization[5m].absent()
```

### Namespaces (built-in OCI services)
```
oci_computeagent        # Compute CPU, Memory, Disk, Network
oci_blockstore          # Block Volume IOPS, throughput, latency
oci_lbaas               # Load Balancer connections, requests, errors
oci_oke                 # OKE cluster and node metrics
oci_autonomous_database # ADB CPU, storage, IOPS
oci_objectstorage       # Bucket requests, bytes, latency
oci_functions           # Function invocations, duration, errors
oci_streaming           # Stream messages, consumer lag
oci_vcn                 # VCN packet flow metrics
```

### Alarms
```bash
# Create CPU alarm
oci monitoring alarm create \
  --compartment-id $C \
  --display-name "high-cpu-alarm" \
  --namespace "oci_computeagent" \
  --query 'CpuUtilization[5m].mean() > 85' \
  --severity "CRITICAL" \
  --destinations "[\"$TOPIC_ARN\"]" \
  --is-enabled true \
  --pending-duration "PT5M" \    # 5 min before firing
  --message-format "ONS_OPTIMIZED" \
  --body "CPU utilization exceeded 85% for 5 minutes on {resourceDisplayName}"

# Alarm with no-data detection
oci monitoring alarm create \
  --compartment-id $C \
  --display-name "metric-absent-alarm" \
  --namespace "oci_computeagent" \
  --query 'CpuUtilization[5m].absent()' \
  --severity "WARNING" \
  --destinations "[\"$TOPIC_ARN\"]" \
  --is-enabled true

# List alarms and their state
oci monitoring alarm list --compartment-id $C --all --output table
oci monitoring alarm-status list --compartment-id $C --all \
  --query 'data[?"status"==`FIRING`]'

# Suppress alarm (maintenance window)
oci monitoring alarm suppress \
  --alarm-id $ALARM_ID \
  --time-suppress-from "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --time-suppress-until "$(date -u -d '+2 hours' '+%Y-%m-%dT%H:%M:%SZ')"
```

## Logging Service

### Log Groups & Logs
```bash
# Create log group
LOG_GROUP_ID=$(oci logging log-group create \
  --compartment-id $C \
  --display-name "app-logs" \
  --description "Application log group" \
  --query 'data.id' --raw-output)

# Enable service log (e.g., Load Balancer access logs)
oci logging log create \
  --log-group-id $LOG_GROUP_ID \
  --display-name "lb-access-logs" \
  --log-type "SERVICE" \
  --configuration '{
    "source": {
      "sourceType": "OCISERVICE",
      "service": "loadbalancer",
      "resource": "'$LB_ID'",
      "category": "access"
    }
  }'

# Create custom log (for application logs via Unified Monitoring Agent)
oci logging log create \
  --log-group-id $LOG_GROUP_ID \
  --display-name "app-custom-log" \
  --log-type "CUSTOM"

# Search logs (last 1 hour)
oci logging-search search-logs \
  --search-query 'search "'$LOG_GROUP_ID'" | where message like "ERROR"' \
  --time-start "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --time-end "$(date -u '+%Y-%m-%dT%H:%M:%SZ')"
```

### Unified Monitoring Agent (on compute instances)
```bash
# Install OCI Logging agent on Oracle Linux
sudo yum install -y oracle-cloud-agent
sudo systemctl enable oracle-cloud-agent
sudo systemctl start oracle-cloud-agent

# Configure agent to tail application logs
cat > /etc/unified-monitoring-agent/conf.d/app.conf << 'EOF'
<source>
  @type tail
  path /var/log/myapp/*.log
  pos_file /var/log/unified-monitoring-agent/app.log.pos
  tag myapp.logs
  <parse>
    @type none
  </parse>
</source>

<match myapp.logs>
  @type oci_logging
  log_object_id <custom-log-ocid>
</match>
EOF

sudo systemctl restart unified-monitoring-agent
```

## Logging Analytics

OCI Logging Analytics provides log parsing, ML-based anomaly detection, and dashboards.

```bash
# Create log analytics namespace (one-time setup)
oci log-analytics namespace onboard --namespace-name $TENANCY_NAMESPACE

# Upload log file for analysis
oci log-analytics upload upload-log-file \
  --namespace-name $TENANCY_NAMESPACE \
  --log-source-name "Linux Syslog Logs" \
  --upload-name "syslog-$(date +%Y%m%d)" \
  --filename syslog.log \
  --file syslog.log

# Query with Logging Analytics Query Language (LAQL)
oci log-analytics query --namespace-name $TENANCY_NAMESPACE \
  --query-string "* | where 'Log Source' = 'Linux Secure Logs' and 'Message' like 'Failed password' | stats count as failures by 'Source Host' | sort -failures"

# List log sources (parsers)
oci log-analytics source list --namespace-name $TENANCY_NAMESPACE --all \
  --output table --query 'data.items[*].{"Name":"name","Type":"source-type"}'
```

## Events Service

Events triggers actions when OCI resources change state.

```bash
# Create event rule: trigger when any instance is terminated
oci events rule create \
  --compartment-id $C \
  --display-name "instance-terminated-alert" \
  --is-enabled true \
  --condition '{
    "eventType": ["com.oraclecloud.computeapi.terminateinstance.end"],
    "data": {}
  }' \
  --actions '{
    "actions": [{
      "actionType": "ONS",
      "topicId": "'$TOPIC_ID'",
      "isEnabled": true,
      "description": "Notify on instance termination"
    }]
  }'

# Common event types:
# com.oraclecloud.computeapi.launchinstance.end
# com.oraclecloud.computeapi.terminateinstance.end
# com.oraclecloud.objectstorage.createobject
# com.oraclecloud.objectstorage.deleteobject
# com.oraclecloud.identitycontrolplane.createuser.end
# com.oraclecloud.database.autonomous.stop
# com.oraclecloud.cloudguard.problemdetected

# Trigger a Function on Object Storage upload
oci events rule create \
  --compartment-id $C \
  --display-name "process-upload" \
  --is-enabled true \
  --condition '{"eventType":["com.oraclecloud.objectstorage.createobject"],"data":{"additionalDetails":{"bucketName":"uploads"}}}' \
  --actions '{"actions":[{"actionType":"FAAS","functionId":"'$FUNCTION_ID'","isEnabled":true}]}'
```

## Notifications Service

```bash
# Create topic
TOPIC_ID=$(oci ons topic create \
  --compartment-id $C \
  --name "ops-alerts" \
  --query 'data."topic-id"' --raw-output)

# Add email subscription
oci ons subscription create \
  --compartment-id $C \
  --topic-id $TOPIC_ID \
  --protocol "EMAIL" \
  --endpoint "ops-team@example.com"

# Add Slack via HTTPS endpoint (webhook)
oci ons subscription create \
  --compartment-id $C \
  --topic-id $TOPIC_ID \
  --protocol "HTTPS" \
  --endpoint "https://hooks.slack.com/services/T.../B.../..."

# Add PagerDuty
oci ons subscription create \
  --compartment-id $C \
  --topic-id $TOPIC_ID \
  --protocol "PAGERDUTY" \
  --endpoint "https://events.pagerduty.com/integration/<key>/enqueue"

# Publish test message
oci ons message publish \
  --topic-id $TOPIC_ID \
  --title "Test Alert" \
  --body "This is a test notification from OCI Monitoring"
```

## Service Connector Hub

Routes data between OCI services — log processing pipelines, metric export, etc.

```bash
# Route logs to Object Storage
oci sch service-connector create \
  --compartment-id $C \
  --display-name "logs-to-object-storage" \
  --source '{
    "kind": "logging",
    "logSources": [{"logGroupId": "'$LOG_GROUP_ID'"}]
  }' \
  --target '{
    "kind": "objectStorage",
    "bucketName": "log-archive",
    "objectNamePrefix": "logs/"
  }'

# Route metrics to Streaming (for external SIEM)
oci sch service-connector create \
  --compartment-id $C \
  --display-name "metrics-to-stream" \
  --source '{
    "kind": "monitoring",
    "monitoringSources": [{
      "compartmentId": "'$C'",
      "namespaceDetails": {
        "kind": "selected",
        "namespaces": [{"namespace": "oci_computeagent", "metrics": {"kind":"all"}}]
      }
    }]
  }' \
  --target '{"kind":"streaming","streamId":"'$STREAM_ID'"}'
```

## Application Performance Monitoring (APM)

```bash
# Create APM domain
APM_DOMAIN_ID=$(oci apm-control-plane apm-domain create \
  --compartment-id $C \
  --display-name "prod-apm" \
  --is-free-tier false \
  --query 'data.id' --raw-output)

# Get data upload endpoint and private key
oci apm-control-plane apm-domain get --apm-domain-id $APM_DOMAIN_ID
oci apm-control-plane data-keys list --apm-domain-id $APM_DOMAIN_ID

# APM with OpenTelemetry (recommended)
# Set OTEL_EXPORTER_OTLP_ENDPOINT to APM's OTLP endpoint
export OTEL_EXPORTER_OTLP_ENDPOINT="https://apm-ingest.us-ashburn-1.apm.oci.oraclecloud.com:443"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=dataKey <private-data-key>"
export OTEL_SERVICE_NAME="my-app"
```

```python
# Python APM with OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
exporter = OTLPSpanExporter(
    endpoint="https://apm-ingest.us-ashburn-1.apm.oci.oraclecloud.com/20200101/opentelemetry/",
    headers={"Authorization": f"dataKey {PRIVATE_DATA_KEY}"}
)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("my-app")
with tracer.start_as_current_span("my-operation") as span:
    span.set_attribute("user.id", "12345")
    # ... do work
```

## Gotchas & Tips

- **Metric lag** — OCI Monitoring metrics have a 2–5 minute ingestion delay. Alarm `pending-duration` should be ≥ 5 minutes to avoid false positives.
- **Custom metrics** — post custom metrics via `oci monitoring metric-data post`. Maximum 50 datapoints per POST; batch them.
- **Log search limits** — `oci logging-search search-logs` returns max 100 results per call. Use `--page` for pagination.
- **Events condition JSON** — the `data` field in event conditions supports nested path matching. Test conditions using `oci events rule test` before enabling.
- **Notification confirmations** — email and HTTPS subscriptions require confirmation. Check the subscription for a pending confirmation link after creation.
- **Service Connector tasks** — SCH can apply Functions as a transformation step between source and target. Use this to filter/enrich log data before storage.
- **APM always-on tracing** — tracing 100% of requests is expensive. Use sampling (1–10%) in production. Configure via `OTEL_TRACES_SAMPLER=parentbased_traceidratio` and `OTEL_TRACES_SAMPLER_ARG=0.05`.
- **Logging Analytics retention** — default retention is 30 days. Configurable up to 12 months. Archive to Object Storage for long-term via SCH.
- **Alarm split by dimension** — alarms fire per-dimension (per-instance), not aggregated. One alarm rule can trigger many individual alerts. Use `grouping()` MQL to aggregate before applying threshold.
