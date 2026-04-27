---
name: azure-fabric
description: Microsoft Fabric unified analytics platform — OneLake, lakehouses, warehouses, Data Factory, Real-Time Intelligence, Power BI, capacities, and Spark
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

# Microsoft Fabric

Microsoft Fabric is a unified SaaS analytics platform that combines OneLake storage,
lakehouses, data warehouses, data engineering, data science, real-time analytics, and
Power BI into a single licensed product billed through **capacity units (CUs)**.

---

## Architecture Overview

```
Fabric Tenant
└── Capacity (F SKU — F2 to F2048)
    └── Workspace(s)
        ├── Lakehouse       → Delta tables + Files section + SQL analytics endpoint
        ├── Warehouse       → T-SQL warehouse with no physical indexes needed
        ├── Notebook        → Spark compute
        ├── Dataflow Gen2   → Power Query M / low-code ETL
        ├── Pipeline        → ADF-compatible orchestration
        ├── KQL Database    → Real-Time Intelligence
        ├── Eventstream     → Real-time event ingestion
        ├── Semantic Model  → Power BI dataset (Direct Lake capable)
        └── Report          → Power BI report
```

**OneLake** is the single logical data lake underpinning everything — one per tenant,
structured as `onelake://<workspace>/<item>/...`.

---

## Capacities & Licensing

### SKUs
| SKU  | CUs  | Approx equiv | Notes                        |
|------|------|--------------|------------------------------|
| F2   | 2    | 0.25× P1     | Smallest production SKU      |
| F4   | 4    |              |                              |
| F8   | 8    | 1× P1        |                              |
| F64  | 64   | 8× P1        | First SKU w/ full feature set|
| F256 | 256  |              |                              |
| F2048| 2048 |              | Largest                      |

- **Trial capacity** is free for 60 days (~F64 equivalent); one per tenant per year.
- CUs are shared across all workspaces assigned to the capacity.
- Capacity can be **paused** (billing stops, items still exist).
- **Bursting**: short spikes can exceed CU limit; sustained overuse causes throttling.

### Manage via Portal or CLI
```bash
# List capacities (Fabric REST API — no az fabric CLI yet for most operations)
curl -sX GET \
  "https://api.fabric.microsoft.com/v1/capacities" \
  -H "Authorization: Bearer $FABRIC_TOKEN" | jq '.value[] | {id, displayName, sku, state}'

# Pause / resume a capacity (requires admin or capacity admin role)
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/capacities/<capacityId>/suspend" \
  -H "Authorization: Bearer $FABRIC_TOKEN"

curl -sX POST \
  "https://api.fabric.microsoft.com/v1/capacities/<capacityId>/resume" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

### CU Consumption Patterns
- Spark jobs consume CUs per vCore-second while running.
- Warehouse queries consume CUs based on compute used.
- Dataflows Gen2 have a **high watermark**: CU usage is smoothed over time.
- Power BI Direct Lake queries consume CUs from the workspace's capacity.

---

## OneLake

OneLake is ADFS Gen2-compatible. Access paths:

```
ABFS (Azure Blob Filesystem):
  abfss://<workspace-name>@onelake.dfs.fabric.microsoft.com/<lakehouse-name>.Lakehouse/Tables/
  abfss://<workspace-name>@onelake.dfs.fabric.microsoft.com/<lakehouse-name>.Lakehouse/Files/

OneLake path (internal reference):
  onelake://<workspace-name>/<lakehouse-name>.Lakehouse/Tables/<table-name>/
```

### OneLake File Explorer
- Windows desktop app; mounts OneLake as a drive (O:\ or similar).
- Real-time sync for Files section; Tables are read via SQL endpoint.

### Shortcuts
Shortcuts create virtual pointers to external or internal data without copying it.

| Source        | Supported? | Auth method                          |
|---------------|-----------|--------------------------------------|
| ADLS Gen2     | ✅         | Account key, SAS, or workspace identity |
| Amazon S3     | ✅         | Access key + secret, or IAM role     |
| Google Cloud Storage | ✅  | Service account key (JSON)           |
| Another Fabric lakehouse | ✅ | Workspace identity (no extra auth)  |
| S3-compatible | ✅ (GA)    | Access key + secret                  |

```python
# Create a shortcut via Fabric REST API
import requests, json

payload = {
    "name": "my_s3_shortcut",
    "path": "Tables",          # or "Files"
    "target": {
        "type": "AmazonS3",
        "amazonS3": {
            "location": "https://s3.amazonaws.com/my-bucket/path/",
            "subpath": "delta_tables/orders"
        }
    }
}
resp = requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WS_ID}/lakehouses/{LH_ID}/shortcuts",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    json=payload
)
```

**Gotcha**: Shortcuts are **read-only** from the Delta Lake perspective — you cannot
INSERT/UPDATE/DELETE through a shortcut table in the warehouse SQL endpoint.

---

## Lakehouses

### Create & Explore
```bash
# Via Fabric REST API
curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/lakehouses" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"displayName": "sales_lakehouse", "description": "Sales data lakehouse"}'
```

### Delta Lake Format
All managed tables in the lakehouse are stored as Delta Lake on OneLake.

```python
# In a Fabric notebook (PySpark)
df = spark.read.format("delta").load(
    "abfss://MyWorkspace@onelake.dfs.fabric.microsoft.com/sales_lakehouse.Lakehouse/Tables/orders"
)

# Write a Delta table (creates managed table visible in lakehouse explorer)
df.write.format("delta").mode("overwrite").saveAsTable("sales_lakehouse.orders")

# Time travel
df_v0 = spark.read.format("delta").option("versionAsOf", 0).load("Tables/orders")
df_ts  = spark.read.format("delta").option("timestampAsOf", "2024-01-01").load("Tables/orders")
```

### V-Order Optimization
Fabric writes Delta tables with **V-Order** by default — a Microsoft-specific Delta
writer optimization that improves read performance for Power BI Direct Lake and
Fabric engines. It adds ~10-15% write overhead but cuts read time significantly.

```python
# V-Order is on by default in Fabric notebooks. Disable if needed:
spark.conf.set("spark.sql.parquet.vorder.enabled", "false")
```

**Gotcha**: V-Order parquet files are valid standard Delta/Parquet but engines outside
Fabric (external Spark clusters, AWS Athena) don't benefit from the optimization — they
just see normal Parquet.

---

## Spark in Fabric

### Runtime Versions (as of early 2026)
- **Runtime 1.3** — Spark 3.5, Delta 3.x, Python 3.11 (recommended)
- **Runtime 1.2** — Spark 3.4, Delta 2.4, Python 3.10

### Workspace Spark Settings
Configure under Workspace Settings → Data Engineering/Science:

```json
{
  "sparkSettings": {
    "defaultSparkRuntimeVersion": "1.3",
    "customizeComputeEnabled": true
  }
}
```

### Environments (Library Management)
Create a Fabric **Environment** item to pin library versions, then attach it to notebooks.

```bash
# Install libraries in an Environment via REST
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/environments/$ENV_ID/staging/libraries" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pypiPackages": [{"name": "scikit-learn", "version": "1.4.0"}, {"name": "xgboost"}]}'

# Publish the environment (triggers build — takes a few minutes)
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/environments/$ENV_ID/staging/publish" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

### Notebook Patterns
```python
# Read from default lakehouse (attached in notebook settings)
df = spark.read.format("delta").table("orders")

# Read from another lakehouse in same workspace
df = spark.read.format("delta").load(
    "abfss://MyWorkspace@onelake.dfs.fabric.microsoft.com/other_lakehouse.Lakehouse/Tables/customers"
)

# Write to Files section (unmanaged / raw zone)
df.write.parquet("Files/raw/orders/2024/01/")

# Use mssparkutils (Fabric's dbutils equivalent)
mssparkutils.fs.ls("abfss://MyWorkspace@onelake.dfs.fabric.microsoft.com/sales_lakehouse.Lakehouse/Files/")
mssparkutils.notebook.run("child_notebook", timeout=600, arguments={"date": "2024-01-15"})
```

---

## Data Warehouse

The Fabric Warehouse is a **serverless, fully managed** T-SQL engine layered over Delta
Lake in OneLake. No indexes, no statistics management, no traditional DBA tasks.

### T-SQL Patterns
```sql
-- Cross-database query (within same workspace — no linked server needed)
SELECT o.order_id, c.customer_name
FROM sales_warehouse.dbo.orders o
JOIN sales_lakehouse.dbo.customers c    -- lakehouse SQL endpoint
    ON o.customer_id = c.customer_id;

-- COPY INTO (load from ADLS / OneLake Files section)
COPY INTO dbo.orders
FROM 'https://onelake.dfs.fabric.microsoft.com/MyWorkspace/sales_lakehouse.Lakehouse/Files/raw/orders/'
WITH (
    FILE_TYPE = 'PARQUET',
    CREDENTIAL = (IDENTITY = 'Managed Identity')
);

-- External table over Files section
CREATE EXTERNAL TABLE dbo.orders_ext (
    order_id INT,
    order_date DATE,
    amount DECIMAL(18,2)
)
WITH (
    LOCATION = 'Files/raw/orders/',
    DATA_SOURCE = lakehouse_files_ds,
    FILE_FORMAT = parquet_ff
);

-- DDL differences from Azure SQL: no IDENTITY workaround needed, use sequences
CREATE SEQUENCE dbo.order_seq START WITH 1 INCREMENT BY 1;
```

**Gotcha**: The Warehouse does not support traditional clustered indexes or columnstore
indexes — the engine decides physical layout. You cannot force distribution (unlike
Synapse Dedicated).

---

## Data Factory in Fabric

Fabric Data Factory uses the same pipeline canvas as ADF but runs on Fabric compute.

### Pipelines
```bash
# Create a pipeline via REST
curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/datapipelines" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"displayName": "ingest_orders_pipeline"}'
```

### Dataflows Gen2
Dataflows Gen2 use Power Query M for transformation and stage data through an
**internal staging lakehouse** before writing to destination.

Key differences from ADF Mapping Data Flows:
- Low-code / no-code Power Query editor
- Uses Fabric compute (no IR to manage)
- Staging adds latency but enables larger datasets than Power Query Online

**Gotcha**: Dataflows Gen2 always stage data internally — you pay CU cost for the
staging write AND the final destination write. For simple copies, a pipeline with
Copy Activity is cheaper.

### Copy Activity — Common Patterns
```json
{
  "name": "CopyFromBlobToLakehouse",
  "type": "Copy",
  "inputs": [{"referenceName": "BlobDataset", "type": "DatasetReference"}],
  "outputs": [{"referenceName": "LakehouseTableDataset", "type": "DatasetReference"}],
  "typeProperties": {
    "source": {"type": "ParquetSource"},
    "sink": {
      "type": "LakehouseTableSink",
      "tableOption": "autoCreate",
      "writeBehavior": "Append"
    }
  }
}
```

---

## Real-Time Intelligence

### KQL Databases
```kql
// Create table
.create table Telemetry (
    Timestamp: datetime,
    DeviceId: string,
    Temperature: real,
    Humidity: real
)

// Ingest from eventstream (configured via UI) or from storage
.ingest into table Telemetry
    (h'https://mystorageaccount.blob.core.windows.net/mycontainer/data.json.gz;impersonate')
    with (format='json', ingestionMappingReference='TelemetryMapping')

// Common analytics patterns
Telemetry
| where Timestamp > ago(1h)
| summarize avg_temp = avg(Temperature) by bin(Timestamp, 5m), DeviceId
| render timechart

// Join with reference data
Telemetry
| join kind=leftouter Devices on DeviceId
| where Location == "Floor1"
| summarize count() by bin(Timestamp, 1m)
```

### Eventstreams
Eventstreams ingest from: Azure Event Hubs, IoT Hub, Kafka, Azure Service Bus,
custom apps (REST), and can route to KQL databases, lakehouses, or other eventstreams.

**Gotcha**: Eventstream source/destination edits require stopping and restarting the
stream. Any events arriving during restart are buffered up to the source retention period.

### Real-Time Dashboards
- KQL-based dashboards with auto-refresh (minimum 30 seconds).
- Tiles share a time range parameter.
- Cannot embed in Power BI reports (separate artifact).

---

## Power BI Integration

### Direct Lake Mode
Direct Lake reads Delta parquet files from OneLake directly without importing data
into the semantic model — combines Import speed with DirectQuery freshness.

```
Direct Lake prerequisites:
- Lakehouse or warehouse in same workspace (or via shortcut)
- Tables must be in Delta format with V-Order (default for managed tables)
- Model must use Fabric semantic model (not legacy PBIX import model)
```

**Fallback to DirectQuery**: Direct Lake falls back to DirectQuery when:
1. Table is not in Delta format
2. V-Order parquet is missing (e.g., shortcut pointing to non-Fabric Delta)
3. Column/row-level security forces row evaluation per-user
4. Table exceeds the capacity's Direct Lake guardrails (row count limits vary by SKU)

```python
# Check if Direct Lake is falling back (in DAX Studio or SSMS)
# Run DMV query against the semantic model's XMLA endpoint:
SELECT * FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS
-- If IsResident = false and large scans, fallback is occurring
```

### XMLA Endpoint
```bash
# Connect via XMLA (PowerShell)
$server = "powerbi://api.powerbi.com/v1.0/myorg/MyWorkspace"
# Use tabular editor or SSMS with Analysis Services connection
```

---

## Git Integration

### Connect Workspace to Git
1. Workspace Settings → Git integration → Connect
2. Choose Azure DevOps or GitHub
3. Select repo, branch, folder (e.g., `/fabric/workspaces/sales/`)

### Supported Item Types (as of early 2026)
Notebooks, Dataflows Gen2, Pipelines, Environments, Spark Job Definitions,
Lakehouses (schema only, not data), Warehouses (schema only), Reports, Semantic Models.

**Gotcha**: Data is NOT committed to git — only item definitions (JSON/SQL/etc.). The
lakehouse Tables and Files sections live only in OneLake.

### Deployment Pipelines
```
Dev Workspace → Test Workspace → Prod Workspace
```
- Configure deployment rules to substitute linked service names, connection IDs, etc.
- Deployment is item-by-item; pipelines and notebooks can override parameter values per stage.

```bash
# Trigger deployment via REST
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/pipelines/$PIPELINE_ID/deploy" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sourceStageOrder": 0,
    "targetStageOrder": 1,
    "items": [{"type": "Notebook", "sourceItemId": "<id>"}],
    "note": "Deploy v1.2 notebooks"
  }'
```

---

## Gotchas & Operational Tips

1. **Capacity throttling is silent**: When CU consumption exceeds capacity for sustained
   periods, Fabric queues or rejects requests. Monitor via the **Fabric Capacity Metrics app**
   (a Fabric-provided report available in AppSource).

2. **Direct Lake fallback is invisible by default**: Your Power BI report will still work
   but will be slower. Enable query diagnostics in Power BI Desktop to detect fallback.

3. **Shortcuts are read-only in SQL endpoint**: You can read shortcut tables via the
   lakehouse SQL analytics endpoint and in notebooks, but cannot INSERT/UPDATE/DELETE.

4. **Warehouse vs Lakehouse SQL endpoint**: The lakehouse has a read-only SQL endpoint
   (auto-generated); the warehouse is fully writable T-SQL. Both appear in the workspace.

5. **Spark session startup**: First notebook cell takes 2-3 minutes for Spark session
   creation (cold start). Subsequent cells in the same session are fast.

6. **Pipeline triggers and time zones**: All schedule triggers use UTC. There's no
   timezone-aware cron in Fabric pipelines yet (as of early 2026).

7. **OneLake ACL inheritance**: Permissions flow from workspace → item → table/file.
   There's no row-level OneLake ACL — use the SQL endpoint's row-level security for that.

8. **Item names in OneLake paths**: Spaces in workspace/lakehouse names become `%20` in
   ABFS paths. Prefer underscores in names to avoid encoding issues in Spark code.

9. **Delta table OPTIMIZE**: Fabric does not auto-run OPTIMIZE. Small-file accumulation
   from streaming or frequent micro-batch writes will degrade performance. Run manually:
   ```sql
   OPTIMIZE sales_lakehouse.dbo.orders;        -- from warehouse
   -- or in a notebook:
   spark.sql("OPTIMIZE sales_lakehouse.orders") -- PySpark
   ```

10. **F2/F4 limitations**: Trial and very small SKUs lack some features (e.g., XMLA
    write endpoints require F64+, Real-Time Intelligence requires F2+, but some advanced
    features need F8+ or higher).
