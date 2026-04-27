---
name: aws-monitoring
description: AWS monitoring — CloudWatch metrics/logs/alarms, X-Ray, CloudTrail Lake, Config, EventBridge, SNS/SQS alerting
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

# AWS Monitoring & Observability — Comprehensive Reference

## CloudWatch Metrics

```bash
# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average Maximum \
  --query 'Datapoints | sort_by(@, &Timestamp)[].[Timestamp,Average,Maximum]' \
  --output table

# List available metrics for a namespace
aws cloudwatch list-metrics \
  --namespace AWS/ApplicationELB \
  --dimensions Name=LoadBalancer,Value=<alb-name> \
  --query 'Metrics[].MetricName' --output text | tr '\t' '\n' | sort -u

# Put custom metric
aws cloudwatch put-metric-data \
  --namespace MyApp/Business \
  --metric-data '[{
    "MetricName": "OrdersProcessed",
    "Value": 42,
    "Unit": "Count",
    "Dimensions": [{"Name": "Environment", "Value": "prod"}]
  }]'

# High-resolution custom metric (1-second granularity)
aws cloudwatch put-metric-data \
  --namespace MyApp/Perf \
  --metric-data 'MetricName=RequestLatency,Value=125,Unit=Milliseconds,StorageResolution=1'
```

### Useful Metric Namespaces

| Namespace | Service |
|---|---|
| `AWS/EC2` | CPU, NetworkIn/Out, DiskReadOps |
| `AWS/RDS` | CPUUtilization, FreeStorageSpace, ReadLatency |
| `AWS/ApplicationELB` | RequestCount, TargetResponseTime, HTTPCode_Target_5XX |
| `AWS/Lambda` | Invocations, Errors, Duration, Throttles, ConcurrentExecutions |
| `AWS/ECS/ContainerInsights` | CpuUtilized, MemoryUtilized, TaskCount |
| `AWS/DynamoDB` | ConsumedReadCapacityUnits, SystemErrors, SuccessfulRequestLatency |
| `AWS/SQS` | ApproximateNumberOfMessagesVisible, NumberOfMessagesDeleted |
| `AWS/ElastiCache` | CacheHits, CacheMisses, Evictions, CurrConnections |

---

## CloudWatch Alarms

```bash
# Simple threshold alarm
aws cloudwatch put-metric-alarm \
  --alarm-name prod-api-error-rate \
  --alarm-description "5XX rate above 1% for 5 minutes" \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=<alb-full-name> \
  --period 60 --evaluation-periods 5 \
  --statistic Sum \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123:prod-alerts \
  --ok-actions arn:aws:sns:us-east-1:123:prod-alerts

# Anomaly detection alarm (ML baseline)
aws cloudwatch put-metric-alarm \
  --alarm-name prod-latency-anomaly \
  --metrics '[{
    "Id": "m1",
    "MetricStat": {
      "Metric": {"Namespace": "AWS/ApplicationELB", "MetricName": "TargetResponseTime",
                 "Dimensions": [{"Name": "LoadBalancer", "Value": "<alb-name>"}]},
      "Period": 60, "Stat": "p99"
    }
  },{
    "Id": "ad1",
    "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
    "Label": "TargetResponseTime (expected)"
  }]' \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold-metric-id ad1 \
  --evaluation-periods 5 \
  --alarm-actions arn:aws:sns:us-east-1:123:prod-alerts

# Composite alarm (reduce alert fatigue)
aws cloudwatch put-composite-alarm \
  --alarm-name prod-service-degraded \
  --alarm-rule 'ALARM("prod-api-error-rate") AND ALARM("prod-latency-anomaly")' \
  --alarm-actions arn:aws:sns:us-east-1:123:pagerduty-critical

# Get alarm history
aws cloudwatch describe-alarm-history \
  --alarm-name prod-api-error-rate \
  --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[].[Timestamp,HistorySummary]' --output table
```

---

## CloudWatch Logs

```bash
# Create log group with retention
aws logs create-log-group --log-group-name /myapp/prod/api
aws logs put-retention-policy --log-group-name /myapp/prod/api --retention-in-days 30

# Tail logs (like `tail -f`)
aws logs tail /myapp/prod/api --follow --format short

# Filter (search) logs
aws logs filter-log-events \
  --log-group-name /myapp/prod/api \
  --filter-pattern '"ERROR" "database"' \
  --start-time $(($(date +%s) - 3600))000 \
  --query 'events[].[timestamp,message]' --output text

# Metric filter (count errors → custom metric)
aws logs put-metric-filter \
  --log-group-name /myapp/prod/api \
  --filter-name error-count \
  --filter-pattern '[timestamp, level="ERROR", ...]' \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=MyApp/Prod,metricValue=1,unit=Count

# Export to S3
aws logs create-export-task \
  --log-group-name /myapp/prod/api \
  --from 1704067200000 \
  --to 1706745600000 \
  --destination my-logs-bucket \
  --destination-prefix prod-api-export
```

### CloudWatch Logs Insights Queries

```
# Access the console (or use start-query CLI):
aws logs start-query \
  --log-group-name /myapp/prod/api \
  --start-time $(($(date +%s) - 3600)) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message
  | filter @message like /ERROR/
  | stats count(*) as errors by bin(5m)
  | sort @timestamp desc'
```

**Useful Insights patterns:**

```sql
-- Top 10 slowest requests
fields @timestamp, @message
| filter @message like /duration/
| parse @message "duration=* " as dur
| sort dur desc | limit 10

-- Error rate over time
fields @timestamp, @message
| stats count_if(@message like /ERROR/) as errors,
        count(*) as total by bin(1m)
| sort @timestamp asc

-- Lambda cold starts
filter @type = "REPORT"
| filter @initDuration > 0
| stats count(*) as coldStarts, avg(@initDuration) as avgInit by bin(30m)

-- P95/P99 latency
filter @type = "REPORT"
| stats pct(@duration, 95) as p95,
        pct(@duration, 99) as p99,
        avg(@duration) as avg by bin(5m)
```

---

## CloudWatch Dashboards

```bash
# Create dashboard (JSON body)
aws cloudwatch put-dashboard \
  --dashboard-name prod-overview \
  --dashboard-body file://dashboard.json

# Get dashboard widget JSON
aws cloudwatch get-dashboard --dashboard-name prod-overview \
  --query 'DashboardBody' --output text
```

---

## X-Ray (Distributed Tracing)

### SDK Setup (Python)

```python
from aws_xray_sdk.core import xray_recorder, patch_all
patch_all()   # auto-instruments boto3, requests, SQLAlchemy, etc.

xray_recorder.configure(service='my-api', daemon_address='127.0.0.1:2000')

@xray_recorder.capture('process_order')
def process_order(order_id):
    # Adds subsegment; auto-records exceptions
    ...

# Custom annotation (searchable)
xray_recorder.current_segment().put_annotation('order_id', order_id)
# Custom metadata (not searchable, for large data)
xray_recorder.current_segment().put_metadata('payload', order_data)
```

```bash
# Query traces
aws xray get-trace-summaries \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --filter-expression 'responsetime > 5 AND http.status = 500' \
  --query 'TraceSummaries[].[Id,Duration,Http.HttpStatus]' --output table

# Get service map
aws xray get-service-graph \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date -u +%s)

# Get specific trace
aws xray batch-get-traces --trace-ids 1-5759e988-bd862e3fe1be46a994272793
```

---

## EventBridge

```bash
# Create custom event bus
aws events create-event-bus --name myapp-events

# Create rule matching custom events
aws events put-rule \
  --name order-completed \
  --event-bus-name myapp-events \
  --event-pattern '{
    "source": ["myapp.orders"],
    "detail-type": ["OrderCompleted"],
    "detail": {"status": ["completed"], "amount": [{"numeric": [">", 100]}]}
  }' \
  --state ENABLED

# Add Lambda target
aws events put-targets \
  --rule order-completed \
  --event-bus-name myapp-events \
  --targets Id=lambda-processor,Arn=arn:aws:lambda:us-east-1:123:function:order-processor

# Put events (publish)
aws events put-events \
  --entries '[{
    "EventBusName": "myapp-events",
    "Source": "myapp.orders",
    "DetailType": "OrderCompleted",
    "Detail": "{\"orderId\": \"o123\", \"status\": \"completed\", \"amount\": 299.99}"
  }]'

# Scheduled rule (cron)
aws events put-rule \
  --name daily-report \
  --schedule-expression 'cron(0 6 * * ? *)' \
  --state ENABLED
```

---

## SNS / SQS for Alerting

```bash
# Create SNS topic
TOPIC=$(aws sns create-topic --name prod-alerts --query 'TopicArn' --output text)

# Email subscription
aws sns subscribe --topic-arn $TOPIC --protocol email --notification-endpoint ops@example.com

# SMS subscription
aws sns subscribe --topic-arn $TOPIC --protocol sms --notification-endpoint +15555555555

# Create SQS queue with DLQ
DLQ=$(aws sqs create-queue --queue-name prod-alerts-dlq --query 'QueueUrl' --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

QUEUE=$(aws sqs create-queue \
  --queue-name prod-alerts \
  --attributes '{
    "VisibilityTimeout": "300",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"'"$DLQ_ARN"'\",\"maxReceiveCount\":\"3\"}"
  }' \
  --query 'QueueUrl' --output text)

# SNS → SQS fan-out
aws sns subscribe \
  --topic-arn $TOPIC --protocol sqs \
  --notification-endpoint $DLQ_ARN

# Publish message
aws sns publish \
  --topic-arn $TOPIC \
  --subject "ALARM: prod-api-error-rate" \
  --message "Error rate exceeded threshold. Check CloudWatch."
```

---

## AWS Config (Continuous Compliance)

```bash
# Query resource inventory
aws configservice select-resource-config \
  --expression "SELECT resourceId, resourceType, configuration
                WHERE resourceType = 'AWS::EC2::Instance'
                AND configuration.state.name = 'running'"

# Get resource history
aws configservice get-resource-config-history \
  --resource-type AWS::S3::Bucket \
  --resource-id my-bucket \
  --limit 10 \
  --query 'configurationItems[].[configurationItemCaptureTime,configurationItemStatus]' \
  --output table
```

---

## Guardrails & Gotchas

- **CloudWatch metric retention** — high-resolution (1s) kept 3h, 60s kept 15d, 5min kept 63d, 1h kept 455d (15 months). Plan dashboards accordingly.
- **Alarm in INSUFFICIENT_DATA** — new alarms start here. Set `--treat-missing-data notBreaching` for metrics that don't always emit.
- **Logs Insights cost** — billed per GB scanned. Add `--log-group-name` and time filters to limit scope.
- **X-Ray sampling** — default 5% after first request/second. Customize sampling rules to avoid missing rare errors.
- **EventBridge delivery retry** — retries for 24h with exponential backoff. Add a DLQ target to catch permanently failed events.
- **SNS email confirmation** — subscriptions require email confirmation. Can't automate — tell ops to check their email.
- **CloudWatch alarms on missing data** — `breaching` is safe for heartbeat-style checks; `notBreaching` for sparse metrics.
- **Composite alarms don't cross-account** — each component alarm must be in the same account/region.
- **Log Insights** — max 30-day lookback, max 50 log groups per query. Use CloudTrail Lake for longer retention queries.
