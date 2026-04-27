---
name: gcp-costs
description: GCP cost management — billing accounts, budgets, BigQuery export, Recommender, CUDs, Spot pricing, labels, reports
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Cost Management Skill

Understand, monitor, and optimize GCP spending.

---

## Billing Account Basics

```bash
# List billing accounts you have access to
gcloud billing accounts list

# Describe a billing account
gcloud billing accounts describe BILLING_ACCOUNT_ID

# List projects linked to a billing account
gcloud billing projects list --billing-account=BILLING_ACCOUNT_ID

# Link a project to a billing account
gcloud billing projects link my-project \
  --billing-account=BILLING_ACCOUNT_ID

# Unlink (disables all paid services in the project)
gcloud billing projects unlink my-project
```

---

## Budgets and Alerts

Create budgets to receive alerts before costs spiral. Budgets don't cap spending —
they notify.

```bash
# Create a budget (calendar month, alert at 50%, 90%, 100%)
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Monthly Budget $500" \
  --budget-amount=500USD \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100 \
  --calendar-period=MONTH \
  --all-projects

# Budget for a specific project
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Project Budget $200" \
  --budget-amount=200USD \
  --projects=projects/my-project \
  --threshold-rule=percent=80 \
  --threshold-rule=percent=100 \
  --calendar-period=MONTH

# Budget with Pub/Sub notification (for programmatic actions)
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Dev Auto-Alert" \
  --budget-amount=100USD \
  --threshold-rule=percent=100 \
  --calendar-period=MONTH \
  --notifications-rule-pubsub-topic=projects/my-project/topics/billing-alerts

# List budgets
gcloud billing budgets list --billing-account=BILLING_ACCOUNT_ID

# Update a budget amount
gcloud billing budgets update BUDGET_ID \
  --billing-account=BILLING_ACCOUNT_ID \
  --budget-amount=750USD

# Delete a budget
gcloud billing budgets delete BUDGET_ID \
  --billing-account=BILLING_ACCOUNT_ID
```

**Auto-cap spending via Pub/Sub:** Subscribe a Cloud Function to the billing Pub/Sub
topic and have it disable billing on the project when budget is exceeded.

```python
# Cloud Function to disable billing when budget exceeded
import json
import base64
from google.cloud import billing_v1

def stop_billing(event, context):
    data = json.loads(base64.b64decode(event['data']).decode('utf-8'))
    if data.get('costAmount', 0) >= data.get('budgetAmount', float('inf')):
        project_name = f"projects/{data['projectId']}"
        client = billing_v1.CloudBillingClient()
        client.update_project_billing_info(
            name=project_name,
            project_billing_info=billing_v1.ProjectBillingInfo(billing_account_name=""),
        )
        print(f"Billing disabled for {project_name}")
```

---

## BigQuery Billing Export

The most powerful way to analyze costs — export billing data to BigQuery and query it.

```bash
# Enable billing export (do this in the console or via API)
# Console: Billing → Billing export → BigQuery export → Enable

# Once enabled, query cost data:
bq query --use_legacy_sql=false --project_id=my-project '
SELECT
  service.description AS service,
  sku.description AS sku,
  SUM(cost) AS total_cost,
  SUM(usage.amount) AS total_usage,
  usage.unit
FROM `my-project.billing_dataset.gcp_billing_export_v1_BILLING_ACCOUNT_ID`
WHERE
  DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1, 2, 5
ORDER BY total_cost DESC
LIMIT 20
'
```

### Useful Billing Queries

```sql
-- Top services last 30 days
SELECT
  service.description,
  ROUND(SUM(cost), 2) AS cost_usd
FROM `my-project.billing.gcp_billing_export_v1_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1
ORDER BY 2 DESC;

-- Cost by label (e.g., env=production vs env=dev)
SELECT
  labels.value AS environment,
  ROUND(SUM(cost), 2) AS cost_usd
FROM `my-project.billing.gcp_billing_export_v1_*`
LEFT JOIN UNNEST(labels) AS labels ON labels.key = 'env'
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1
ORDER BY 2 DESC;

-- Daily cost trend
SELECT
  DATE(usage_start_time) AS date,
  ROUND(SUM(cost), 2) AS daily_cost
FROM `my-project.billing.gcp_billing_export_v1_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 1
ORDER BY 1;

-- BigQuery cost breakdown (slots, storage, queries)
SELECT
  sku.description,
  ROUND(SUM(cost), 2) AS cost,
  ROUND(SUM(usage.amount), 2) AS usage_amount,
  usage.unit
FROM `my-project.billing.gcp_billing_export_v1_*`
WHERE
  service.description = 'BigQuery'
  AND DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1, 4
ORDER BY 2 DESC;

-- Compute Engine cost by instance name
SELECT
  resource.name AS instance_name,
  ROUND(SUM(cost), 2) AS cost_usd
FROM `my-project.billing.gcp_billing_export_v1_*`,
  UNNEST(labels) AS label
WHERE
  service.description = 'Compute Engine'
  AND DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1
ORDER BY 2 DESC;
```

---

## Recommender (Rightsizing and Idle Resources)

```bash
# List VM rightsizing recommendations
gcloud recommender recommendations list \
  --project=my-project \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --format="table(name,stateInfo.state,primaryImpact.costProjection.cost.units)"

# List idle VM recommendations (VMs that should be deleted or stopped)
gcloud recommender recommendations list \
  --project=my-project \
  --location=us-central1-a \
  --recommender=google.compute.instance.IdleResourceRecommender

# List idle disk recommendations
gcloud recommender recommendations list \
  --project=my-project \
  --location=us-central1 \
  --recommender=google.compute.disk.IdleResourceRecommender

# List idle GKE cluster recommendations
gcloud recommender recommendations list \
  --project=my-project \
  --location=us-central1 \
  --recommender=google.container.DiagnosisRecommender

# Mark a recommendation as claimed (you're acting on it)
gcloud recommender recommendations mark-claimed RECOMMENDATION_ID \
  --project=my-project \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --etag=CURRENT_ETAG

# Mark as succeeded (after implementing)
gcloud recommender recommendations mark-succeeded RECOMMENDATION_ID \
  --project=my-project \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --etag=CURRENT_ETAG

# Mark as failed (if the recommendation caused issues)
gcloud recommender recommendations mark-failed RECOMMENDATION_ID \
  --project=my-project \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --etag=CURRENT_ETAG
```

---

## Committed Use Discounts (CUDs)

CUDs are 1-year or 3-year commitments for sustained resource usage. They provide
significant discounts (up to 57% for memory-optimized, up to 37% for general purpose).

### Types

| CUD Type | Applies To | Commitment |
|---------|-----------|-----------|
| **Resource-based** | Specific vCPUs and RAM | Exact resource amounts |
| **Spend-based** | Cloud SQL, Vertex AI, GKE Autopilot | Dollar spend |

```bash
# Purchase a resource-based commitment
gcloud compute commitments create my-commitment \
  --region=us-central1 \
  --plan=TWELVE_MONTH \
  --resources=vcpu=20,memory=80GB

# List active commitments
gcloud compute commitments list

# Describe a commitment
gcloud compute commitments describe my-commitment --region=us-central1
```

**Gotcha:** CUD commitments are billing account-level, not project-level. They apply
across all projects under the billing account automatically (if not project-scoped).

**Spend-based CUDs** are purchased in the console. Common use cases:
- Cloud SQL: 25–52% discount for 1–3 year commits
- Vertex AI: 17% discount for 1-year commit

---

## Sustained Use Discounts (SUDs)

SUDs are **automatic** — no action needed. The longer a VM runs in a month, the
bigger the discount. Full month = up to 30% off.

- Applies to: N1, N2, N2D, E2 VM families, Dataflow workers, and some Kubernetes nodes.
- Does **not** apply to: E2, N1 when using committed use discounts (they're separate).
- Does **not** apply to: Cloud SQL, App Engine, Cloud Run.

---

## Preemptible / Spot Pricing

Spot/Preemptible VMs are up to 91% cheaper than on-demand. Use them for:
- Batch jobs, data pipelines
- CI/CD workers
- Fault-tolerant distributed workloads

```bash
# Check preemptible pricing (pricing varies by machine type and region)
gcloud compute machine-types describe n2-standard-4 --zone=us-central1-a

# Create a spot VM
gcloud compute instances create my-spot-vm \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --machine-type=n2-standard-4 \
  --zone=us-central1-a \
  [other flags...]

# Check spot VM eviction rate (via Monitoring)
# metric: compute.googleapis.com/instance/preemptions
```

---

## Labels for Cost Allocation

Labels are key-value pairs applied to resources. Export billing data to BigQuery
and filter by label for per-team, per-env, per-project cost breakdown.

```bash
# Add labels to a VM
gcloud compute instances add-labels my-vm \
  --zone=us-central1-a \
  --labels=env=production,team=backend,cost-center=eng-001

# Add labels to a GCS bucket
gsutil label ch \
  -l env:production \
  -l team:data \
  gs://my-bucket

# Add labels to a BigQuery dataset
bq update --set_label env:production --set_label team:analytics \
  my-project:my_dataset

# Add labels to a Cloud SQL instance
gcloud sql instances patch my-db \
  --update-labels=env=production,team=backend

# Add labels to a GKE cluster
gcloud container clusters update my-cluster \
  --zone=us-central1-a \
  --update-labels=env=production,team=platform

# Remove a label
gcloud compute instances remove-labels my-vm \
  --zone=us-central1-a \
  --labels=temp-label
```

**Label convention recommendation:**
```
env: production | staging | dev
team: backend | frontend | data | platform | security
cost-center: eng-001 | data-002 | ops-003
project: my-product-name
```

---

## Billing Reports and Cost Tools

```bash
# View current month cost via billing API
gcloud billing accounts describe BILLING_ACCOUNT_ID

# Use the `gcloud` billing overview (limited)
gcloud billing accounts list

# For detailed reports: use BigQuery export or the console
# console.cloud.google.com/billing → Reports
# console.cloud.google.com/billing → Cost table
# console.cloud.google.com/billing → Cost breakdown
```

### Useful Cost Optimization Checks

```bash
# Find VMs in TERMINATED state still billing for reserved IPs/disks
gcloud compute instances list \
  --filter="status=TERMINATED" \
  --format="table(name,zone,status,disks[0].diskSizeGb)"

# Find unattached disks (still billing)
gcloud compute disks list \
  --filter="NOT users:*" \
  --format="table(name,zone,sizeGb,type)"

# Find unattached static IPs (billed at $0.010/hr when unused)
gcloud compute addresses list \
  --filter="status=RESERVED" \
  --format="table(name,region,address,status)"

# Find load balancers with no backends
gcloud compute backend-services list \
  --format="table(name,backends)" \
  --global

# Find old snapshots
gcloud compute snapshots list \
  --format="table(name,diskSizeGb,creationTimestamp)" \
  --sort-by=~creationTimestamp
```

---

## Cost Guardrails by Architecture

| Pattern | Savings |
|---------|---------|
| Use `e2-micro` or `f1-micro` for dev VMs | Up to 80% vs n1-standard |
| Set `min-instances=0` on Cloud Run dev services | 100% when idle |
| Use BigQuery `require_partition_filter` | Prevents expensive full scans |
| Lifecycle rules on GCS | Auto-move old data to cheaper storage class |
| Use GKE Spot node pools for batch workloads | Up to 91% savings |
| Enable idle resource recommendations | Find waste automatically |
| Auto-delete old Artifact Registry images | Reduces storage costs |
| Stop dev VMs at night with Cloud Scheduler + Cloud Functions | ~60% savings |

---

## Guardrails

- **Labels are not retroactive in BigQuery billing export** — apply them at resource creation.
- **CUD commitments are binding** — you pay whether or not you use the resources.
- **Unattached static IPs cost money** — release them when not in use.
- **Persistent disks continue billing when VMs are stopped** — delete disks if the VM won't restart.
- **BigQuery on-demand pricing charges per byte scanned** — always use `LIMIT` clauses and partition filters in dev.
- **Cloud Run min-instances > 0 means always-on billing** — use only for latency-sensitive production services.
- **Snapshot costs add up** — implement a retention policy; don't keep snapshots indefinitely.
- **Always set budget alerts before starting a new project** — it's the first thing to configure.
