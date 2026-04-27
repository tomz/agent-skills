---
name: azure-databricks
description: Azure Databricks — workspace provisioning, Unity Catalog, clusters, SQL warehouses, Delta Lake, MLflow, Jobs, CLI, and security
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

# Azure Databricks

Azure Databricks is a fully managed Apache Spark platform on Azure, tightly integrated
with Azure Active Directory / Entra ID, ADLS Gen2, Azure Key Vault, and Azure Monitor.
Billed in **DBUs (Databricks Units)** — rate depends on cluster type and SKU tier
(Standard, Premium, Enterprise).

---

## Workspace Provisioning

### az CLI
```bash
# Register provider (once per subscription)
az provider register --namespace Microsoft.Databricks

# Create workspace — standard deployment (public)
az databricks workspace create \
  --resource-group my-rg \
  --name my-databricks-ws \
  --location eastus \
  --sku premium          # standard | premium | trial

# Create with VNet injection (no-public-ip, private)
az databricks workspace create \
  --resource-group my-rg \
  --name my-databricks-ws \
  --location eastus \
  --sku premium \
  --virtual-network /subscriptions/$SUB/resourceGroups/net-rg/providers/Microsoft.Network/virtualNetworks/my-vnet \
  --private-subnet-name dbricks-private \
  --public-subnet-name  dbricks-public \
  --no-public-ip        # disables public IPs on cluster nodes (secure cluster connectivity)

# Show workspace URL
az databricks workspace show \
  --resource-group my-rg \
  --name my-databricks-ws \
  --query workspaceUrl -o tsv
```

### VNet Injection Notes
- Requires two delegated subnets (min /26 each) — public and private — in the same VNet.
- `--no-public-ip` (Secure Cluster Connectivity / SCC) routes all traffic through a
  NAT gateway; cluster nodes have no public IPs. Preferred for production.
- Once created, VNet injection settings **cannot be changed**.

---

## Databricks CLI

### Install & Auth
```bash
pip install databricks-cli    # legacy v0.x
# OR
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
# Installs databricks CLI v0.200+ (Go binary)

# Auth — Personal Access Token (legacy CLI)
databricks configure --token
# Prompts for host (https://<workspace-url>) and PAT

# Auth — OAuth M2M (CLI v0.200+, recommended for automation)
databricks auth login \
  --host https://<workspace-url> \
  --client-id $SP_CLIENT_ID \
  --client-secret $SP_CLIENT_SECRET

# Auth — Azure Service Principal via environment
export DATABRICKS_HOST=https://<workspace-url>
export ARM_CLIENT_ID=<sp-app-id>
export ARM_CLIENT_SECRET=<sp-secret>
export ARM_TENANT_ID=<tenant-id>
databricks clusters list     # auto-discovers auth from env

# Multiple profiles
databricks configure --token --profile prod
databricks clusters list --profile prod
```

### Common CLI Commands (v0.200+)
```bash
# Clusters
databricks clusters list
databricks clusters get --cluster-id <id>
databricks clusters start --cluster-id <id>
databricks clusters delete --cluster-id <id>

# Jobs
databricks jobs list
databricks jobs run-now --job-id <id>
databricks jobs get-run --run-id <id>

# Workspace (notebooks/files)
databricks workspace ls /Users/me@example.com
databricks workspace export /Users/me/notebook --format SOURCE -o notebook.py
databricks workspace import notebook.py /Users/me/notebook --format SOURCE --language PYTHON

# DBFS
databricks fs ls dbfs:/mnt/mydata
databricks fs cp local_file.csv dbfs:/tmp/local_file.csv
databricks fs rm dbfs:/tmp/local_file.csv

# Secrets
databricks secrets list-scopes
databricks secrets list --scope my-scope
databricks secrets put --scope my-scope --key my-secret --string-value "mysecretvalue"

# Unity Catalog
databricks unity-catalog catalogs list
databricks unity-catalog schemas list --catalog-name my_catalog
databricks unity-catalog tables list --catalog-name my_catalog --schema-name my_schema
```

---

## Unity Catalog

Unity Catalog (UC) provides centralized governance for all data assets across workspaces.

### Hierarchy
```
Metastore (one per region, assigned to workspaces)
└── Catalog
    └── Schema (Database)
        ├── Table (managed or external)
        ├── View
        └── Volume (unstructured files)
```

### Metastore Setup
```bash
# Assign metastore to workspace (done by Databricks account admin)
databricks unity-catalog metastores assign \
  --workspace-id <workspace-numeric-id> \
  --metastore-id <metastore-id> \
  --default-catalog-name main
```

### External Locations & Credentials
```sql
-- Create storage credential (uses managed identity or service principal)
CREATE STORAGE CREDENTIAL my_adls_cred
  WITH AZURE_MANAGED_IDENTITY (
    CONNECTOR_ID = '/subscriptions/.../resourceGroups/.../providers/Microsoft.Databricks/accessConnectors/my-connector'
  );

-- Create external location
CREATE EXTERNAL LOCATION my_ext_loc
  URL 'abfss://mycontainer@mystorageaccount.dfs.core.windows.net/'
  WITH (STORAGE CREDENTIAL my_adls_cred);

SHOW EXTERNAL LOCATIONS;
```

### Grants (Fine-Grained Access Control)
```sql
-- Grant catalog access
GRANT USE CATALOG ON CATALOG my_catalog TO `analysts@example.com`;
GRANT USE SCHEMA  ON SCHEMA  my_catalog.sales TO `analysts@example.com`;
GRANT SELECT      ON TABLE   my_catalog.sales.orders TO `analysts@example.com`;

-- Grant for service principal (use application ID format)
GRANT SELECT ON TABLE my_catalog.sales.orders TO `app-id-xxxx`;

-- Create a view with column masking
CREATE OR REPLACE VIEW my_catalog.sales.orders_masked AS
SELECT
  order_id,
  CASE WHEN is_account_group_member('pii_access') THEN customer_name
       ELSE '***MASKED***' END AS customer_name,
  amount
FROM my_catalog.sales.orders_raw;
```

### Volumes (Unstructured Files in UC)
```sql
-- Create managed volume
CREATE VOLUME my_catalog.my_schema.raw_files;

-- Create external volume
CREATE EXTERNAL VOLUME my_catalog.my_schema.landing_zone
  URL 'abfss://landing@mystorageaccount.dfs.core.windows.net/incoming/'
  WITH (STORAGE CREDENTIAL my_adls_cred);
```

```python
# Read from volume in notebook
df = spark.read.csv("/Volumes/my_catalog/my_schema/raw_files/orders.csv", header=True)
dbutils.fs.ls("/Volumes/my_catalog/my_schema/landing_zone/")
```

**Gotcha**: Unity Catalog requires **Premium tier** workspace. Standard workspaces use
the legacy Hive metastore (workspace-local, no cross-workspace sharing).

---

## Clusters

### All-Purpose vs Job Clusters
| Feature          | All-Purpose (Interactive) | Job Cluster           |
|------------------|---------------------------|-----------------------|
| Created by       | User (UI or API)          | Job orchestrator      |
| Lifecycle        | Manual start/stop         | Terminates after job  |
| DBU rate         | Higher (interactive rate) | Lower (job rate)      |
| Use case         | Notebooks, exploration    | Scheduled workloads   |

### Cluster Config
```json
{
  "cluster_name": "my-cluster",
  "spark_version": "14.3.x-scala2.12",
  "node_type_id": "Standard_DS3_v2",
  "driver_node_type_id": "Standard_DS3_v2",
  "autoscale": {
    "min_workers": 2,
    "max_workers": 10
  },
  "azure_attributes": {
    "first_on_demand": 1,
    "availability": "SPOT_WITH_FALLBACK_AZURE",
    "spot_bid_max_price": -1
  },
  "spark_conf": {
    "spark.databricks.delta.preview.enabled": "true",
    "spark.sql.adaptive.enabled": "true"
  },
  "custom_tags": {"team": "data-engineering", "project": "sales"},
  "autotermination_minutes": 60
}
```

### Photon
Photon is Databricks' C++ native vectorized query engine — drop-in replacement for
Spark SQL and DataFrame operations. Enable at cluster creation:
```json
{"runtime_engine": "PHOTON"}
```
Photon accelerates: filter/project/aggregation/join/MERGE. Does NOT accelerate
RDD-based code or UDFs (use Pandas UDFs with Arrow for best Photon compat).

### Cluster Policies
Policies restrict what users can configure — enforce tagging, limit node types, cap
autoscale, enforce autotermination:
```bash
databricks cluster-policies list
databricks cluster-policies create --json @policy.json
```

**Gotcha**: Cluster start time is 5-8 minutes for standard clusters. Use **cluster
pools** (pre-warmed instances) to reduce startup to <30 seconds for job clusters.

---

## Databricks SQL

### SQL Warehouses
| Type       | Cold Start | Min Clusters | Best For              |
|------------|-----------|--------------|----------------------|
| Serverless | ~5 sec    | 0            | Ad-hoc, dashboards   |
| Pro        | ~2 min    | 0 (scales to 0) | BI tools, moderate load |
| Classic    | ~3 min    | 1 minimum    | Steady-state workloads|

```bash
# Create SQL warehouse via CLI
databricks warehouses create --json '{
  "name": "my-sql-warehouse",
  "cluster_size": "Medium",
  "min_num_clusters": 1,
  "max_num_clusters": 3,
  "auto_stop_mins": 30,
  "warehouse_type": "PRO",
  "enable_photon": true,
  "channel": {"name": "CHANNEL_NAME_CURRENT"}
}'
```

**Gotcha — Serverless minimum cost**: Serverless SQL warehouses have a minimum
billing even when idle (a small "management" charge). Not truly free at zero usage.

### Query Patterns
```sql
-- Parameterized query (Databricks SQL native params)
SELECT * FROM catalog.sales.orders
WHERE order_date BETWEEN {{start_date}} AND {{end_date}}
  AND region = {{region}};

-- Query history for cost analysis
SELECT
  executed_by,
  statement_text,
  total_duration_ms / 1000.0 AS duration_sec,
  total_task_duration_ms
FROM system.query.history
WHERE executed_at > current_timestamp() - INTERVAL 1 DAY
ORDER BY total_duration_ms DESC
LIMIT 20;
```

---

## Delta Lake

### Core Operations
```python
from delta.tables import DeltaTable

# MERGE (upsert)
target = DeltaTable.forName(spark, "my_catalog.sales.orders")
target.alias("t").merge(
    source_df.alias("s"),
    "t.order_id = s.order_id"
).whenMatchedUpdateAll(
).whenNotMatchedInsertAll(
).execute()

# Time travel
df_v5  = spark.read.format("delta").option("versionAsOf", 5).table("my_catalog.sales.orders")
df_old = spark.read.format("delta").option("timestampAsOf", "2024-06-01").table("my_catalog.sales.orders")

# Describe history
spark.sql("DESCRIBE HISTORY my_catalog.sales.orders").show(truncate=False)
```

### OPTIMIZE & ZORDER
```sql
-- Compact small files
OPTIMIZE my_catalog.sales.orders;

-- ZORDER (co-locate data for common filter columns — not sorting, it's clustering)
OPTIMIZE my_catalog.sales.orders ZORDER BY (customer_id, order_date);

-- Liquid Clustering (replaces ZORDER in Delta 3.x+ / DBR 13.3+)
-- Set at table creation — no manual OPTIMIZE needed, auto-clusters incrementally
CREATE TABLE my_catalog.sales.orders
CLUSTER BY (customer_id, order_date)
AS SELECT * FROM staging.orders;

-- Or alter existing table
ALTER TABLE my_catalog.sales.orders CLUSTER BY (customer_id, order_date);
```

### Vacuum & Change Data Feed
```sql
-- Vacuum old versions (default 7-day retention; shorter = more aggressive cleanup)
VACUUM my_catalog.sales.orders RETAIN 168 HOURS;

-- Enable Change Data Feed for incremental processing
ALTER TABLE my_catalog.sales.orders SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');

-- Read CDF
spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 10) \
    .table("my_catalog.sales.orders")
```

**Gotcha — ZORDER diminishing returns**: ZORDER on high-cardinality columns with many
small files is ineffective. Combine OPTIMIZE (file compaction) with ZORDER (locality),
and prefer Liquid Clustering for new tables.

---

## Jobs & Workflows

### Job Definition (JSON)
```json
{
  "name": "daily_sales_pipeline",
  "tasks": [
    {
      "task_key": "ingest",
      "notebook_task": {
        "notebook_path": "/Shared/pipelines/ingest_orders",
        "base_parameters": {"date": "{{job.start_time.iso_date}}"}
      },
      "job_cluster_key": "main_cluster",
      "timeout_seconds": 1800
    },
    {
      "task_key": "transform",
      "depends_on": [{"task_key": "ingest"}],
      "python_wheel_task": {
        "package_name": "sales_transforms",
        "entry_point": "run_transforms",
        "parameters": ["--date", "{{job.start_time.iso_date}}"]
      },
      "job_cluster_key": "main_cluster"
    }
  ],
  "job_clusters": [{
    "job_cluster_key": "main_cluster",
    "new_cluster": {
      "spark_version": "14.3.x-scala2.12",
      "node_type_id": "Standard_DS3_v2",
      "num_workers": 4
    }
  }],
  "schedule": {
    "quartz_cron_expression": "0 0 6 * * ?",
    "timezone_id": "America/New_York",
    "pause_status": "UNPAUSED"
  },
  "max_concurrent_runs": 1,
  "email_notifications": {
    "on_failure": ["oncall@example.com"]
  }
}
```

```bash
databricks jobs create --json @job.json
databricks jobs run-now --job-id 12345 \
  --notebook-params '{"date": "2024-01-15"}'
```

---

## MLflow

### Experiment Tracking
```python
import mlflow

mlflow.set_experiment("/Users/me@example.com/sales_forecasting")

with mlflow.start_run(run_name="xgboost_v3"):
    mlflow.log_params({"n_estimators": 100, "max_depth": 6, "learning_rate": 0.1})
    # ... train model ...
    mlflow.log_metrics({"rmse": 142.3, "mae": 98.7})
    mlflow.xgboost.log_model(model, "model", registered_model_name="sales_forecast_model")
```

### Model Registry & Serving
```python
# Transition model stage (legacy) or set alias (UC-backed registry)
client = mlflow.tracking.MlflowClient()

# Unity Catalog model registry (recommended)
# Format: <catalog>.<schema>.<model_name>
mlflow.set_registry_uri("databricks-uc")
mlflow.register_model("runs:/<run-id>/model", "my_catalog.ml.sales_forecast")

# Set alias for deployment
client.set_registered_model_alias("my_catalog.ml.sales_forecast", "champion", version=3)

# Load by alias
model = mlflow.pyfunc.load_model("models:/my_catalog.ml.sales_forecast@champion")
```

```bash
# Deploy as REST endpoint (Model Serving)
databricks serving-endpoints create --json '{
  "name": "sales-forecast-endpoint",
  "config": {
    "served_entities": [{
      "entity_name": "my_catalog.ml.sales_forecast",
      "entity_version": "3",
      "workload_size": "Small",
      "scale_to_zero_enabled": true
    }]
  }
}'
```

---

## Security

### Secret Scopes
```bash
# Databricks-backed secret scope
databricks secrets create-scope my-scope --initial-manage-principal users

# Azure Key Vault-backed scope (read-only from Databricks side)
databricks secrets create-scope kv-scope \
  --scope-backend-type AZURE_KEYVAULT \
  --resource-id /subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$KV_NAME \
  --dns-name https://$KV_NAME.vault.azure.net/

# Use in notebook
dbutils.secrets.get(scope="kv-scope", key="sql-password")
```

**Gotcha — Key Vault-backed scopes**: Secrets are read-only (you can't write back to KV
from Databricks). The Databricks workspace managed identity must have Key Vault Secrets
User role on the Key Vault.

### IP Access Lists
```bash
# Enable IP access list feature
databricks ip-access-lists create \
  --label "Corporate VPN" \
  --list-type ALLOW \
  --ip-addresses '["203.0.113.0/24", "198.51.100.42"]'
```

### Table ACLs (Legacy — use Unity Catalog instead)
```sql
-- Only available with Premium tier + legacy Hive metastore
GRANT SELECT ON TABLE default.orders TO `user@example.com`;
DENY  SELECT ON TABLE default.pii    TO `analyst_group`;
```

---

## Gotchas & Operational Tips

1. **DBU cost varies widely**: All-purpose clusters (interactive) cost 2-4× more DBU/hr
   than job clusters for the same hardware. Always use job clusters for scheduled work.

2. **Cluster start time**: Standard clusters take 5-8 minutes. For latency-sensitive jobs,
   use cluster pools (pre-warmed) or keep a small always-on cluster for critical paths.

3. **Unity Catalog requires Premium**: Standard tier workspaces cannot use UC. Plan your
   tier before provisioning — downgrading isn't supported.

4. **VNet injection is permanent**: The VNet, subnets, and NSG rules cannot be changed
   after workspace creation. Get the network design right before provisioning.

5. **MERGE performance**: Large MERGE operations on Delta can be slow without ZORDER/Liquid
   Clustering on the join key. Partition by date + cluster by ID for typical SCD patterns.

6. **Serverless SQL minimum cost**: Even when scaled to zero, serverless warehouses incur
   a small management fee (~$0.05-0.10/hr). Disable warehouses not in use.

7. **VACUUM and time travel**: VACUUM with < 7-day retention breaks time travel. If you
   have downstream consumers using time travel (e.g., CDF readers), coordinate retention.

8. **Spot instance interruptions**: With `SPOT_WITH_FALLBACK_AZURE`, Databricks falls back
   to on-demand if spot isn't available. Monitor spot eviction rates in cluster event logs.

9. **Auto Loader vs COPY INTO**: Use Auto Loader (`cloudFiles`) for streaming ingestion
   with checkpointing; use COPY INTO for one-time or idempotent batch loads.

10. **PAT expiry**: Personal Access Tokens expire (default 90 days for managed accounts).
    Use OAuth M2M (service principal) for production automation — tokens are short-lived
    and auto-refreshed.
