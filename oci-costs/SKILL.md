---
name: oci-costs
description: OCI cost management — Cost Analysis, Budgets, Usage Reports, quotas, Always Free, Reserved capacity, tagging, Cost Advisor
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

# OCI Cost Management

## Cost Analysis

OCI Cost Analysis provides usage and cost breakdowns with filters and groupings.

```bash
# Query costs for last 30 days, grouped by service
oci usage-api usage-summary request-summarized-usages \
  --tenant-id <tenancy-ocid> \
  --time-usage-started "$(date -u -d '30 days ago' '+%Y-%m-%dT00:00:00Z')" \
  --time-usage-ended "$(date -u '+%Y-%m-%dT00:00:00Z')" \
  --granularity "MONTHLY" \
  --query-type "COST" \
  --group-by '["service"]' \
  --output table

# Group by compartment
oci usage-api usage-summary request-summarized-usages \
  --tenant-id <tenancy-ocid> \
  --time-usage-started "2024-01-01T00:00:00Z" \
  --time-usage-ended "2024-02-01T00:00:00Z" \
  --granularity "DAILY" \
  --query-type "COST" \
  --group-by '["compartmentId","compartmentPath"]'

# Filter to specific compartment
oci usage-api usage-summary request-summarized-usages \
  --tenant-id <tenancy-ocid> \
  --time-usage-started "2024-01-01T00:00:00Z" \
  --time-usage-ended "2024-02-01T00:00:00Z" \
  --granularity "MONTHLY" \
  --query-type "COST" \
  --filter '{
    "operator": "AND",
    "dimensions": [{"key":"compartmentId","value":"'$C'"}]
  }'

# Group by tag
oci usage-api usage-summary request-summarized-usages \
  --tenant-id <tenancy-ocid> \
  --time-usage-started "2024-01-01T00:00:00Z" \
  --time-usage-ended "2024-02-01T00:00:00Z" \
  --granularity "MONTHLY" \
  --query-type "COST" \
  --group-by '["tags"]' \
  --group-by-tag '[{"namespace":"","key":"Environment"}]'

# Cost forecast (next 30 days)
oci usage-api usage-summary request-summarized-usages \
  --tenant-id <tenancy-ocid> \
  --time-usage-started "$(date -u '+%Y-%m-%dT00:00:00Z')" \
  --time-usage-ended "$(date -u -d '+30 days' '+%Y-%m-%dT00:00:00Z')" \
  --granularity "MONTHLY" \
  --query-type "FORECAST" \
  --forecast-creation "StatisticalAnalysis"
```

### Save Cost Analysis Queries
```bash
# Create a saved query
oci usage-api query create \
  --compartment-id <tenancy-ocid> \
  --query-definition '{
    "displayName": "Monthly Costs by Service",
    "reportQuery": {
      "tenantId": "<tenancy-ocid>",
      "timeUsageStarted": "2024-01-01T00:00:00Z",
      "timeUsageEnded": "2025-01-01T00:00:00Z",
      "granularity": "MONTHLY",
      "queryType": "COST",
      "groupBy": ["service"]
    },
    "costAnalysisUI": {"graph": "BARS"}
  }'
```

## Budgets

```bash
# Create a monthly budget for the tenancy
oci budgets budget create \
  --compartment-id <tenancy-ocid> \
  --amount 1000 \
  --reset-period "MONTHLY" \
  --target-type "COMPARTMENT" \
  --targets "[\"$C\"]" \
  --display-name "prod-monthly-budget" \
  --description "Alert when production costs exceed $1000/month"

# Create budget on the entire tenancy
oci budgets budget create \
  --compartment-id <tenancy-ocid> \
  --amount 5000 \
  --reset-period "MONTHLY" \
  --target-type "COMPARTMENT" \
  --targets "[\"<tenancy-ocid>\"]" \
  --display-name "total-monthly-budget"

# Create alert rule on budget (80% and 100% thresholds)
BUDGET_ID=$(oci budgets budget list --compartment-id <tenancy-ocid> \
  --query "data[?\"display-name\"=='prod-monthly-budget'].id | [0]" --raw-output)

oci budgets alert-rule create \
  --budget-id $BUDGET_ID \
  --display-name "80-percent-alert" \
  --threshold 80 \
  --threshold-type "PERCENTAGE" \
  --type "ACTUAL" \
  --recipients "finops@example.com" \
  --message "Production budget at 80%. Review spending."

oci budgets alert-rule create \
  --budget-id $BUDGET_ID \
  --display-name "100-percent-alert" \
  --threshold 100 \
  --threshold-type "PERCENTAGE" \
  --type "ACTUAL" \
  --recipients "finops@example.com,management@example.com" \
  --message "CRITICAL: Production budget fully consumed this month!"

# Forecast alert (fire before you hit 100%)
oci budgets alert-rule create \
  --budget-id $BUDGET_ID \
  --display-name "forecast-110-alert" \
  --threshold 110 \
  --threshold-type "PERCENTAGE" \
  --type "FORECAST" \
  --recipients "finops@example.com" \
  --message "Forecasted to exceed budget by 10% this month."

# Check budget status
oci budgets budget list --compartment-id <tenancy-ocid> --all --output table \
  --query 'data[*].{"Name":"display-name","Budget":"amount","Spent":"actual-spend","Forecast":"forecasted-spend","State":"lifecycle-state"}'
```

## Usage Reports (CSV Export)

OCI automatically writes CSV usage reports to a special tenancy-owned Object Storage bucket.

```bash
# Find the usage reports bucket name
oci usage-api configuration get \
  --query 'data."items"'

# Or: tenancy namespace is your tenancy name, bucket is fixed
# oci-reports-<tenancy-namespace>

# List available report files
oci os object list \
  --namespace-name <reports-namespace> \
  --bucket-name "oci-reports-<tenancy>" \
  --all --output table

# Download a specific report
oci os object get \
  --namespace-name <reports-namespace> \
  --bucket-name "oci-reports-<tenancy>" \
  --name "reports/cost-csv/0001000000012345.csv.gz" \
  --file ./cost-report.csv.gz

# Decompress and inspect
gunzip cost-report.csv.gz
head -2 cost-report.csv

# Key columns in the CSV:
# lineItem/referenceNo, lineItem/tenantId, lineItem/intervalUsageStart
# lineItem/intervalUsageEnd, product/service, product/compartmentName
# usage/billedQuantity, usage/billedQuantityOverage, cost/currencyCode
# cost/myCost, cost/overageFlag, tags/[key]
```

### Automate report processing
```bash
# Download all new reports since last run (sync-like pattern)
LAST_DATE="2024-01-01"
oci os object list \
  --namespace-name <namespace> \
  --bucket-name "oci-reports-<tenancy>" \
  --all \
  --query "data[?\"time-modified\" > \`${LAST_DATE}T00:00:00Z\`].name" \
  --raw-output | while read REPORT; do
    oci os object get --namespace-name <namespace> \
      --bucket-name "oci-reports-<tenancy>" \
      --name "$REPORT" \
      --file "./reports/$(basename $REPORT)"
  done
```

## Always Free Resources

OCI Always Free is permanent (not a trial) — includes:

| Resource | Limit |
|----------|-------|
| Compute VM | 2× AMD VM.Standard.E2.1.Micro (1 OCPU, 1 GB RAM each) |
| Arm Compute | 4 OCPUs + 24 GB RAM total across A1.Flex instances |
| Object Storage | 20 GB standard storage |
| Block Volumes | 2 block volumes, 200 GB total |
| Autonomous Database | 2 ADB instances, 20 GB each |
| Load Balancer | 1 flexible LB (10 Mbps) |
| Outbound Data | 10 TB/month |
| Monitoring | 500M datapoints ingestion, 1B retrieval |
| Notifications | 1M HTTPS, 1000 email deliveries |
| VCN | 2 VCNs |

```bash
# Check if you're hitting Always Free limits
oci limits resource-availability get \
  --compartment-id <tenancy-ocid> \
  --service-name "compute" \
  --limit-name "vm-standard-a1-micro-count"

# Check quota usage across services
oci limits limit-value list \
  --compartment-id <tenancy-ocid> \
  --service-name "compute" --all --output table
```

## Compartment Quotas

Quotas restrict resource usage within compartments — use for chargeback or to prevent runaway spending.

```bash
# Create quota policy (quota statements are NOT IAM policies)
oci limits quota create \
  --compartment-id <tenancy-ocid> \
  --name "dev-compute-quota" \
  --description "Limit dev environment compute" \
  --statements '[
    "Zero compute quotas in compartment dev",
    "Set compute quota vm-standard-e4-flex-ocpus to 32 in compartment dev",
    "Set compute quota vm-standard-a1-flex-memory-gbs to 256 in compartment dev",
    "Set objectstorage quota storage-size-gbs to 1000 in compartment dev"
  ]'

# List quotas
oci limits quota list --compartment-id <tenancy-ocid> --all --output table

# Check effective quota vs usage
oci limits resource-availability get \
  --compartment-id $C \
  --service-name "objectstorage" \
  --limit-name "storage-size-gbs"
```

## Flex Shapes for Cost Optimization

Flex shapes let you right-size instances. Key patterns:

```bash
# Start small, scale up without re-provisioning
oci compute instance update \
  --instance-id $INST_ID \
  --shape-config '{"ocpus":1,"memoryInGBs":8}'   # scale down
  # Must be stopped first for some shapes

# Find cheapest flex configuration
# E4.Flex: ~$0.0255/OCPU/hr + $0.0015/GB/hr
# A1.Flex: ~$0.01/OCPU/hr + $0.0015/GB/hr (Arm — cheapest for CPU-light workloads)

# Compare shapes
oci compute shape list --compartment-id $C --all \
  --query 'data[?contains(shape, `Flex`) && contains(shape, `A1`)]'
```

## Reserved Capacity

Reserve VM capacity to guarantee availability (useful for large launches).

```bash
# Create capacity reservation
oci compute capacity-reservation create \
  --compartment-id $C \
  --availability-domain "kWVD:US-ASHBURN-AD-1" \
  --display-name "prod-reservation" \
  --instance-reservation-configs '[{
    "instanceShape": "VM.Standard.E4.Flex",
    "instanceShapeConfig": {"ocpus":4,"memoryInGBs":32},
    "reservedCount": 10
  }]'

# Launch instance using reservation
oci compute instance launch \
  --capacity-reservation-id $RESERVATION_ID \
  ... (other flags)
```

## Universal Credits Model

OCI billing uses Universal Credits — a pool of $ you can spend on any OCI service.

Key facts:
- Commit upfront for discounts (1-year or 3-year)
- Universal Credits can be used for any service (compute, database, storage, etc.)
- BYOL (Bring Your Own License) — use existing Oracle DB/middleware licenses on OCI

```bash
# Check subscription details
oci osub-subscription subscription list \
  --compartment-id <tenancy-ocid> \
  --all --output table
```

## Tagging for Cost Allocation

Consistent tagging is critical for cost attribution.

```bash
# Create tag namespace
oci iam tag-namespace create \
  --compartment-id $C \
  --name "CostCenter" \
  --description "Cost allocation tags"

# Create tag key
oci iam tag create \
  --tag-namespace-id $NS_ID \
  --name "Project" \
  --description "Project name for billing" \
  --is-cost-tracking true   # Makes it appear in cost reports

# Create tag key with allowed values (enum)
oci iam tag create \
  --tag-namespace-id $NS_ID \
  --name "Environment" \
  --validator '{"validatorType":"ENUM","values":["prod","staging","dev","test"]}' \
  --is-cost-tracking true

# Apply tags when creating resources
oci compute instance launch \
  --defined-tags '{"CostCenter":{"Project":"WebApp","Environment":"prod"}}' \
  --freeform-tags '{"Owner":"platform-team","ManagedBy":"terraform"}' \
  ... (other flags)

# Tag existing resource
oci compute instance update \
  --instance-id $INST_ID \
  --defined-tags '{"CostCenter":{"Project":"WebApp","Environment":"prod"}}'

# Default tags (auto-apply to resources in compartment)
oci iam tag-default create \
  --compartment-id $C \
  --tag-definition-id $TAG_ID \
  --value "dev"
```

## Cost Advisor

Cost Advisor analyzes usage patterns and recommends savings.

```bash
# List Cost Advisor recommendations
oci optimizer recommendation list \
  --compartment-id <tenancy-ocid> \
  --compartment-id-in-subtree true \
  --all \
  --output table \
  --query 'data[*].{"Name":"name","Status":"status","Savings":"estimated-cost-saving","Importance":"importance"}'

# Get details on a specific recommendation
oci optimizer recommendation get \
  --recommendation-id $REC_ID

# List resource actions for a recommendation (what to actually do)
oci optimizer resource-action list \
  --recommendation-id $REC_ID \
  --all --output table

# Implement (e.g., resize instance to recommended shape)
# Cost Advisor provides the specific CLI command in the action details
```

Common Cost Advisor recommendations:
- **Idle compute** — instances with <5% CPU utilization for 14+ days → recommend stopping or downsizing
- **Oversized instances** — recommend smaller flex shape config
- **Unattached block volumes** — orphaned block volumes with no attached instance
- **Old snapshots** — volume backups older than X days consuming storage
- **Underutilized DB** — ADB with consistently low ECPU usage → recommend reducing compute

## FinOps Best Practices

```bash
# 1. Always tag resources at creation (use tag defaults)
# 2. Set budgets before deploying anything
# 3. Enable Cost Analysis scheduled reports

# Scheduled cost report to Object Storage
oci usage-api scheduled-run create \
  --compartment-id <tenancy-ocid> \
  --schedule-recurrences "MONTHLY" \
  --name "monthly-cost-report" \
  --query-properties '{
    "queryType": "COST",
    "granularity": "MONTHLY",
    "groupBy": ["service","compartmentId"]
  }' \
  --result-location '{
    "locationType": "OBJECT_STORAGE",
    "bucketName": "cost-reports",
    "namespace": "'$NAMESPACE'"
  }'

# 4. Use Arm (A1.Flex) for cost-insensitive workloads
# 5. Use preemptible instances for batch/fault-tolerant workloads (not yet GA in all regions)
# 6. Archive cold Object Storage data (10x cheaper than Standard)
# 7. Stop dev instances on weekends (save ~30% monthly)

# Quick cost check: top spending services this month
oci usage-api usage-summary request-summarized-usages \
  --tenant-id <tenancy-ocid> \
  --time-usage-started "$(date -u '+%Y-%m-01T00:00:00Z')" \
  --time-usage-ended "$(date -u '+%Y-%m-%dT23:59:59Z')" \
  --granularity "MONTHLY" \
  --query-type "COST" \
  --group-by '["service"]' | \
  jq '.data.items | sort_by(.computedAmount) | reverse | .[:10] | .[] | {service: .grouping.service, cost: .computedAmount}'
```

## Gotchas & Tips

- **Cost Analysis data lag** — usage data appears in Cost Analysis with a 24–48 hour delay. Don't expect today's spending to show up in real-time.
- **Budget targets** — budgets on compartments only track costs for resources IN that compartment (not sub-compartments unless `target-type=COMPARTMENT` and targeting the root). Use tenancy-level budgets for total visibility.
- **Usage Reports vs Cost Analysis** — Usage Reports (CSV) are more detailed (line-item granularity). Cost Analysis is easier for aggregated views. Use both.
- **Always Free shape quotas** — Always Free A1 (4 OCPU/24 GB) is tracked at tenancy level across ALL compartments. Monitor with `oci limits resource-availability`.
- **Outbound data costs** — OCI charges for egress to internet (first 10 TB free). Inter-region traffic also costs. Use Service Gateways to avoid egress for OCI-to-OCI traffic.
- **Defined tags for cost-tracking** — only `is-cost-tracking = true` tags appear in Usage Reports and Cost Analysis grouping. Set this when creating tag keys.
- **Tag enforcement** — use Tag Defaults + Security Zones to enforce mandatory tags. Without enforcement, developers will skip tagging.
- **BYOL savings** — running Oracle Database SE/EE on OCI with BYOL can save 40–60% vs LICENSE_INCLUDED. Requires valid Oracle license agreement.
