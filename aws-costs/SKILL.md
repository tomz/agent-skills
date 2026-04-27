---
name: aws-costs
description: AWS cost management — Cost Explorer, Budgets, Savings Plans, Reserved Instances, Spot, Trusted Advisor, Compute Optimizer, right-sizing, Cost Anomaly Detection
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

# AWS Cost Management — Comprehensive Reference

## Cost Explorer

### CLI Queries

```bash
# Monthly costs by service (current month)
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups | sort_by(@, &Keys[0])[].[Keys[0],Metrics.BlendedCost.Amount]' \
  --output table

# Daily costs for last 30 days
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics UnblendedCost \
  --query 'ResultsByTime[].[TimePeriod.Start,Total.UnblendedCost.Amount]' \
  --output table

# Costs by tag (requires cost allocation tags enabled)
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=TAG,Key=Project \
  --query 'ResultsByTime[0].Groups[].[Keys[0],Metrics.BlendedCost.Amount]' \
  --output table

# Filter to specific service
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}' \
  --query 'ResultsByTime[0].Total.BlendedCost.Amount' --output text

# Cost forecast (next 3 months)
aws ce get-cost-forecast \
  --time-period Start=$(date +%Y-%m-%d),End=$(date -d '90 days' +%Y-%m-%d) \
  --granularity MONTHLY \
  --metric BLENDED_COST \
  --query '[ForecastResultsByTime[].[TimePeriod.Start,MeanValue], Total.Amount]' \
  --output table
```

### Useful Filters

```json
// EC2 On-Demand only (exclude Spot/RI)
{"And": [
  {"Dimensions": {"Key": "SERVICE", "Values": ["Amazon Elastic Compute Cloud - Compute"]}},
  {"Dimensions": {"Key": "PURCHASE_TYPE", "Values": ["On Demand"]}}
]}

// Specific environment via tag
{"Tags": {"Key": "Environment", "Values": ["prod"]}}

// Exclude support costs
{"Not": {"Dimensions": {"Key": "SERVICE", "Values": ["AWS Support (Business)"]}}}
```

---

## AWS Budgets

```bash
# Create monthly cost budget ($500, alert at 80% and 100%)
aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{
    "BudgetName": "monthly-total",
    "BudgetLimit": {"Amount": "500", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "ops@example.com"}]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{"SubscriptionType": "SNS", "Address": "arn:aws:sns:us-east-1:123:budget-alerts"}]
    }
  ]'

# Usage budget (prevent runaway Lambda invocations)
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "lambda-invocations",
    "BudgetLimit": {"Amount": "1000000", "Unit": "Invocations"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "USAGE",
    "CostFilters": {"Service": ["AWS Lambda"]}
  }' \
  --notifications-with-subscribers '[...]'

# Budget actions (auto-remediation — e.g., attach a deny-all SCP when threshold hit)
aws budgets create-budget-action \
  --account-id 123456789012 \
  --budget-name monthly-total \
  --notification-type ACTUAL \
  --action-type APPLY_SCP_POLICY \
  --action-threshold ActionThresholdValue=110,ActionThresholdType=PERCENTAGE \
  --definition SscpActionDefinition='{
    "PolicyId": "p-xxxx",
    "TargetIds": ["123456789012"],
    "TargetType": "ACCOUNT"
  }' \
  --execution-role-arn arn:aws:iam::123:role/BudgetsActionRole \
  --approval-model AUTOMATIC
```

---

## Savings Plans

| Type | Flexibility | Discount |
|---|---|---|
| Compute Savings Plans | Any instance family, region, OS, tenancy, ECS Fargate, Lambda | Up to 66% |
| EC2 Instance Savings Plans | Specific instance family + region, any size/OS/tenancy | Up to 72% |
| SageMaker Savings Plans | SageMaker ML instances | Up to 64% |

```bash
# Check your current Savings Plans utilization
aws savingsplans describe-savings-plans-utilization \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --query 'SavingsPlansUtilizationsByTime[].Utilization.[UtilizationPercentage,UnusedCommitment.Amount]' \
  --output table

# Get Savings Plans recommendations
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.SavingsPlansPurchaseRecommendationDetails[:5].[EstimatedMonthlySavingsAmount,CurrentAverageHourlyOnDemandSpend]' \
  --output table

# Purchase Savings Plan
aws savingsplans create-savings-plan \
  --savings-plan-offering-id <offering-id> \   # from describe-offerings
  --commitment 0.50 \    # hourly commitment in USD
  --purchase-time 2024-03-01T00:00:00Z
```

**Rule of thumb:** Buy Compute Savings Plans first (most flexible). Only buy EC2 Instance SPs if you're confident in a specific family. Start with 1-year No Upfront to minimize commitment risk.

---

## Reserved Instances

RIs are being largely superseded by Savings Plans for compute, but still relevant for RDS, ElastiCache, Redshift, OpenSearch.

```bash
# Get RI purchase recommendations
aws ce get-reservation-purchase-recommendation \
  --service AmazonRDS \
  --term-in-years ONE_YEAR \
  --payment-option PARTIAL_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

# Check RI utilization
aws ce get-reservation-utilization \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'UtilizationsByTime[0].Groups[].[Keys[0],Utilization.UtilizationPercentage]' \
  --output table

# List your active RIs
aws ec2 describe-reserved-instances \
  --filters Name=state,Values=active \
  --query 'ReservedInstances[].[ReservedInstancesId,InstanceType,InstanceCount,End]' \
  --output table

# Sell unused RIs on Marketplace
aws ec2 create-reserved-instances-listing \
  --reserved-instances-id <ri-id> \
  --instance-count 2 \
  --price-schedules '[{"Price": 100, "Term": 1}]' \
  --client-token $(uuidgen)
```

---

## Spot Instances

```bash
# Get Spot price history
aws ec2 describe-spot-price-history \
  --instance-types m5.xlarge c5.xlarge \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'SpotPriceHistory | sort_by(@, &Timestamp)[].[AvailabilityZone,InstanceType,SpotPrice,Timestamp]' \
  --output table

# Request Spot instance
aws ec2 run-instances \
  --instance-type m5.xlarge \
  --image-id ami-0abcdef \
  --instance-market-options '{"MarketType":"spot","SpotOptions":{"SpotInstanceType":"one-time","InstanceInterruptionBehavior":"terminate"}}' \
  --count 1

# Spot Fleet (mixed instances for resilience)
aws ec2 request-spot-fleet --spot-fleet-request-config '{
  "IamFleetRole": "arn:aws:iam::123:role/SpotFleetRole",
  "TargetCapacity": 10,
  "AllocationStrategy": "capacityOptimized",
  "LaunchTemplateConfigs": [{
    "LaunchTemplateSpecification": {"LaunchTemplateId": "lt-xxx", "Version": "$Latest"},
    "Overrides": [
      {"InstanceType": "m5.xlarge", "WeightedCapacity": 1},
      {"InstanceType": "m5a.xlarge", "WeightedCapacity": 1},
      {"InstanceType": "m4.xlarge", "WeightedCapacity": 1}
    ]
  }]
}'
```

### Spot Interruption Handling

```bash
# Poll for interruption notice (from within instance)
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/spot/instance-action
# Returns {"action":"terminate","time":"2024-01-15T10:00:00Z"} 2 minutes before termination

# Graceful shutdown script
while true; do
  NOTICE=$(curl -sf -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/spot/instance-action 2>/dev/null)
  if [ $? -eq 0 ]; then
    echo "Spot interruption! Saving state..."
    # checkpoint work, deregister from ELB, etc.
    break
  fi
  sleep 5
done
```

---

## Trusted Advisor

```bash
# List checks
aws support describe-trusted-advisor-checks --language en \
  --query 'checks[].[id,category,name]' --output table

# Get check results (requires Business/Enterprise support)
aws support describe-trusted-advisor-check-result \
  --check-id Qch7DwouX1   # Low Utilization Amazon EC2 Instances
  --language en \
  --query 'result.flaggedResources[].[region,status,metadata]' --output table

# Refresh all checks
aws support refresh-trusted-advisor-check --check-id Qch7DwouX1
```

Key checks: Low Utilization EC2, Idle Load Balancers, Underutilized EBS, Unassociated Elastic IPs, RDS Idle, High Utilization EC2 (opposite — may need upsize).

---

## Compute Optimizer

```bash
# Get EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].[currentInstanceType,finding,recommendationOptions[0].instanceType,estimatedMonthlySavings.value]' \
  --output table

# Lambda recommendations
aws compute-optimizer get-lambda-function-recommendations \
  --query 'lambdaFunctionRecommendations[].[functionArn,finding,memorySizeRecommendationOptions[0].memorySize,estimatedMonthlySavings.value]' \
  --output table

# EBS recommendations
aws compute-optimizer get-ebs-volume-recommendations \
  --query 'volumeRecommendations[].[currentConfiguration.volumeType,finding,volumeRecommendationOptions[0].configuration.volumeType,estimatedMonthlySavings.value]' \
  --output table

# Opt in (required first)
aws compute-optimizer update-enrollment-status --status Active
```

---

## S3 Intelligent-Tiering

```bash
# Enable Intelligent-Tiering on a bucket
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-bucket \
  --id main-config \
  --intelligent-tiering-configuration '{
    "Id": "main-config",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'

# Check tier distribution
aws s3api list-bucket-metrics-configurations --bucket my-bucket
```

**When to use:** Objects >128 KB with unpredictable access. NOT worth it for objects <128 KB (monitoring fee exceeds savings).

---

## Cost Anomaly Detection

```bash
# Create anomaly monitor
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "service-anomalies",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Create subscription (alert when anomaly > $100)
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "big-anomalies",
    "MonitorArnList": ["arn:aws:ce::123:anomalymonitor/xxx"],
    "Subscribers": [{"Address": "arn:aws:sns:us-east-1:123:cost-alerts", "Type": "SNS"}],
    "Threshold": 100,
    "Frequency": "DAILY"
  }'

# Get recent anomalies
aws ce get-anomalies \
  --date-interval StartDate=$(date -d '30 days ago' +%Y-%m-%d),EndDate=$(date +%Y-%m-%d) \
  --query 'Anomalies[].[AnomalyId,AnomalyStartDate,MaxImpact.MaxImpactedService,Impact.TotalImpact]' \
  --output table
```

---

## Right-Sizing Checklist

```bash
# Identify oversized EC2 (CPU < 10% average over 14 days)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc \
  --start-time $(date -d '14 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date +%Y-%m-%dT%H:%M:%SZ) \
  --period 1209600 --statistics Average

# Lambda memory right-sizing (use AWS Lambda Power Tuning tool)
# https://github.com/alexcasalboni/aws-lambda-power-tuning
# Run via Step Functions — tests multiple memory configs, finds optimal

# Find idle RDS instances (no connections for 7 days)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=my-db \
  --start-time $(date -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date +%Y-%m-%dT%H:%M:%SZ) \
  --period 604800 --statistics Maximum
```

---

## Guardrails & Gotchas

- **Savings Plans utilization alert** — set a budget alarm if utilization drops below 90% — unused commitment is wasted money.
- **RI vs SP** — for new workloads, always prefer Compute Savings Plans. RIs only for databases (RDS, ElastiCache) where SPs don't apply.
- **Spot interruption = data loss** — never use Spot for stateful workloads without checkpointing. Use EFS/S3 for shared state.
- **NAT Gateway data** — often a surprise top-5 cost item. Enable S3/DynamoDB VPC Gateway endpoints to eliminate NAT data charges for those services.
- **CloudWatch metrics cost** — custom metrics $0.30/metric/month. 1000 custom metrics = $300/month. Use EMF (embedded metric format) to batch efficiently.
- **Cost allocation tags** — must be activated in Billing console separately from creating the tags. Takes 24h to appear in Cost Explorer.
- **Free tier** — monitored per service, not total. You can blow through t2.micro free hours and still get charged for Lambda separately.
- **Data transfer costs** — cross-AZ data transfer is $0.01/GB each direction. Put ALBs and their targets in the same AZ, or use AZ-aware routing.
- **Cost Explorer API** — costs $0.01 per API request. Automate sparingly; don't call in every Lambda invocation.
- **Budget actions** — test them! A misconfigured action that locks out your entire account is worse than overspending.
