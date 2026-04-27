---
name: azure-fabric
description: Microsoft Fabric unified analytics platform — OneLake, lakehouses, warehouses, Data Factory, Real-Time Intelligence, Power BI, capacities, Spark engineering, and data science
license: MIT
version: 2.0.0
updated: 2026-04-27
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
        ├── Notebook        → Spark compute (PySpark, Scala, SparkSQL, R)
        ├── Spark Job Def   → Headless Spark batch jobs (jar/py)
        ├── Environment     → Library + Spark config management
        ├── Dataflow Gen2   → Power Query M / low-code ETL
        ├── Pipeline        → ADF-compatible orchestration
        ├── KQL Database    → Real-Time Intelligence
        ├── Eventstream     → Real-time event ingestion
        ├── Eventhouse      → KQL database + eventstream bundle
        ├── ML Model        → MLflow-registered model
        ├── ML Experiment   → MLflow experiment tracking
        ├── Semantic Model  → Power BI dataset (Direct Lake capable)
        └── Report          → Power BI report
```

**OneLake** is the single logical data lake underpinning everything — one per tenant,
structured as `onelake://<workspace>/<item>/...`.

---

## Capacities & Licensing

### SKUs
| SKU  | CUs  | Spark vCores (max) | Notes                        |
|------|------|---------------------|------------------------------|
| F2   | 2    | 8                   | Smallest production SKU      |
| F4   | 4    | 16                  |                              |
| F8   | 8    | 32                  | ≈ 1× P1                     |
| F16  | 16   | 64                  |                              |
| F32  | 32   | 128                 |                              |
| F64  | 64   | 256                 | Full feature set (XMLA r/w)  |
| F128 | 128  | 512                 |                              |
| F256 | 256  | 1024                |                              |
| F512 | 512  | 2048                |                              |
| F1024| 1024 | 4096                |                              |
| F2048| 2048 | 8192                | Largest                      |

- **Trial capacity** is free for 60 days (~F64 equivalent); one per tenant per year.
- CUs are shared across all workspaces assigned to the capacity.
- Capacity can be **paused** (billing stops, items still exist).
- **Bursting**: short spikes can exceed CU limit; sustained overuse causes throttling then rejection.
- **Spark CU consumption**: 1 Spark vCore ≈ 0.5 CU/hour. An F64 capacity can run up to 256 concurrent Spark vCores.

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
| Dataverse     | ✅         | Entra ID linked service              |

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

### Lakehouse Structure
```
sales_lakehouse.Lakehouse/
├── Tables/            ← Managed Delta tables (visible in SQL endpoint + Explorer)
│   ├── orders/        ← Delta table (_delta_log/ + parquet files)
│   ├── customers/
│   └── products/
└── Files/             ← Unmanaged files (raw zone, staging, CSV, Parquet, images, etc.)
    ├── raw/
    ├── staging/
    └── reference/
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

## Spark in Fabric — Comprehensive Guide

### Runtime Versions (as of early 2026)
| Runtime | Spark | Delta Lake | Python | Java | Scala | R     | OS              | Status         |
|---------|-------|-----------|--------|------|-------|-------|-----------------|----------------|
| 2.0     | 4.0   | 4.0       | 3.12   | 21   | 2.13  | 4.5.2 | Azure Linux 3.0 | Public Preview |
| 1.3     | 3.5   | 3.2       | 3.11   | 11   | 2.12  | 4.3   | —               | GA (Recommended) |
| 1.2     | 3.4   | 2.4       | 3.10   | 11   | 2.12  | 4.3   | —               | GA             |
| 1.1     | 3.3   | 2.2       | 3.10   | 8    | 2.12  | 4.2   | —               | Deprecated     |

### Runtime 2.0 — Key Changes (Public Preview)

**Major version jumps**: Spark 3.x → 4.0, Delta 2.x/3.x → 4.0, Java 11 → 21, Scala 2.12 → 2.13, Python 3.10/3.11 → 3.12.

#### Spark 4.0 Highlights
- **VARIANT data type** — native semi-structured data support (replaces string-based JSON columns)
- **SQL user-defined functions** — define reusable SQL functions without Python/Scala
- **Session variables** — `DECLARE VARIABLE`, `SET VARIABLE` for session-scoped state
- **Pipe syntax** — `table |> SELECT ... |> WHERE ...` for readable query chaining
- **String collation** — locale-aware string comparison and sorting
- **PySpark native plotting API** — built-in `df.plot()` without matplotlib
- **Python Data Source API** — custom data sources in pure Python
- **Python UDTFs** — user-defined table functions in Python
- **Unified profiling** — profile PySpark UDFs with Spark's built-in profiler
- **Structured Streaming Arbitrary State API v2** — more flexible stateful processing
- **Structured Streaming State Data Source** — debug and inspect streaming state
- **SparkR is deprecated** — may be removed in a future version

#### Delta Lake 4.0 Highlights
- Cross-format interoperability improvements
- Performance optimizations for large tables
- Parallel Delta snapshot loading (reduces query startup for large tables)

**⚠ Important**: Delta Lake 4.0-specific features are **experimental** and only work in
Spark experiences (Notebooks, Spark Job Definitions). If you share Delta tables across
multiple Fabric workloads (warehouse, Power BI, etc.), do NOT enable Delta 4.0-only features.
Check [Delta Lake interoperability docs](https://learn.microsoft.com/en-us/fabric/fundamentals/delta-lake-interoperability) first.

#### Performance — Native Execution Engine
- **Up to 6× faster** than open-source Spark on TPC-DS benchmarks
- Vectorized processing — accelerates queries without code changes
- **Vectorized CSV parsing** — much faster CSV ingestion
- Vectorized JSON parsing and Streaming support planned for future updates
- Enable at environment level so all jobs inherit the performance boost

```python
# Enable Native Execution Engine (Runtime 2.0)
spark.conf.set("spark.native.enabled", "true")
# Or set at Environment level (recommended — applies to all notebooks/jobs)
```

#### Compute Management (Runtime 2.0)
- **Resource Profiles** — predefined resource allocations for sessions (control cost/performance)
- **Custom Live Pools (preview)** — pre-warmed Spark pools to eliminate cold start

#### Migration Notes (1.3 → 2.0)
- **Scala 2.12 → 2.13**: Recompile any custom Scala JARs. Binary incompatible.
- **Java 11 → 21**: Review deprecated Java APIs. Most code works, but reflection-heavy libraries may need updates.
- **Python 3.11 → 3.12**: Check third-party library compatibility (most major libs support 3.12).
- **WASB protocol deprecated**: Use `abfss://` (ABFS) instead of `wasbs://` for GPv2 storage accounts.
- **SparkR deprecated**: Migrate R workloads to Python or consider alternatives.
- **Delta 4.0 experimental features**: Don't enable on tables shared with non-Spark Fabric workloads.
- **Initial session startup may be slower** during preview — use Custom Live Pools to mitigate.

### Spark Compute Model
Fabric Spark is **serverless** — no persistent clusters to manage. Sessions are created
on demand with configurable node sizes.

**Node types (Spark pools):**
| Node Size | Driver Memory | Executor Memory | Cores/Node |
|-----------|--------------|-----------------|------------|
| Small     | 28 GB        | 28 GB           | 4          |
| Medium    | 56 GB        | 56 GB           | 8          |
| Large     | 112 GB       | 112 GB          | 16         |
| XLarge    | 224 GB       | 224 GB          | 32         |
| XXLarge   | 432 GB       | 432 GB          | 64         |

**Executor allocation:**
- **Default**: Dynamic allocation enabled. Min 1 executor, max varies by capacity.
- Executors scale up/down within the session as shuffle stages demand.
- Idle sessions auto-terminate after configurable timeout (default 20 min).

### Workspace Spark Settings
Configure under Workspace Settings → Data Engineering/Science:

```python
# Or set per-session in notebooks:
spark.conf.set("spark.executor.instances", "4")
spark.conf.set("spark.executor.cores", "4")
spark.conf.set("spark.executor.memory", "28g")
spark.conf.set("spark.driver.memory", "28g")
spark.conf.set("spark.dynamicAllocation.enabled", "true")
spark.conf.set("spark.dynamicAllocation.minExecutors", "1")
spark.conf.set("spark.dynamicAllocation.maxExecutors", "10")
```

### Environments (Library & Configuration Management)
Create a Fabric **Environment** item to manage:
- **PyPI packages** — pinned versions, no wheel upload needed for public packages
- **Custom libraries** — upload `.whl`, `.jar`, `.tar.gz` files
- **Conda packages** — via `environment.yml`
- **Spark properties** — set defaults per environment
- **R packages** — install from CRAN

```bash
# Install libraries in an Environment via REST
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/environments/$ENV_ID/staging/libraries" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pypiPackages": [{"name": "scikit-learn", "version": "1.4.0"}, {"name": "xgboost"}]}'

# Publish the environment (triggers build — takes 3-10 minutes)
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/environments/$ENV_ID/staging/publish" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

```yaml
# environment.yml example for conda-based environments
name: my_env
channels:
  - conda-forge
  - defaults
dependencies:
  - numpy=1.26.*
  - pandas=2.1.*
  - scipy
  - pip:
    - great-expectations==0.18.*
    - delta-spark==3.2.*
```

**Gotcha**: After publishing an Environment, existing running sessions do NOT pick up
changes. You must restart the session (or start a new notebook session).

### Inline Library Installation (Quick & Dirty)
```python
# Install in current session only — lost on session restart
%pip install great-expectations==0.18.0
%pip install /lakehouse/default/Files/libs/my_custom_lib-1.0.0-py3-none-any.whl

# Conda (slower, but available)
%conda install -c conda-forge prophet
```

### High-Concurrency Mode
Enables **session sharing** across multiple notebooks and users on the same Spark cluster.
Reduces cold-start overhead by reusing live sessions.

```python
# Enable via workspace settings or per-session
# In workspace settings → Data Engineering → High Concurrency Mode: ON

# Notebooks share the same SparkContext but get isolated SparkSessions
# Variables and imports in one notebook are NOT visible to others
```

**When to use**: Development scenarios with multiple users iterating on notebooks.
**When NOT to use**: Production jobs where isolation matters, or jobs needing full
cluster resources.

### Native Execution Engine
Fabric's vectorized execution engine that accelerates Spark queries on Delta tables.
Available on Runtime 1.2+; **up to 6× faster on Runtime 2.0** (TPC-DS benchmarks).

```python
# Enable native execution engine
spark.conf.set("spark.native.enabled", "true")

# Check if a query used native execution
df = spark.sql("SELECT * FROM orders WHERE amount > 100")
df.explain(True)  # Look for "NativeExecution" in the physical plan
```

**Supported**: Scan, filter, project, aggregation, join (hash/sort-merge), sort, window.
Runtime 2.0 adds vectorized CSV parsing; vectorized JSON and Streaming support planned.
**Not supported**: UDFs (Python/Scala), complex types (struct/array/map operations),
some date/timestamp edge cases. Falls back to JVM Spark transparently.

### Notebook Patterns

#### Basic Read/Write
```python
# Read from default lakehouse (attached in notebook metadata)
df = spark.read.format("delta").table("orders")

# Equivalent using spark.sql
df = spark.sql("SELECT * FROM sales_lakehouse.orders")

# Read from another lakehouse in same workspace
df = spark.read.format("delta").load(
    "abfss://MyWorkspace@onelake.dfs.fabric.microsoft.com/other_lakehouse.Lakehouse/Tables/customers"
)

# Write to managed table
df.write.format("delta").mode("overwrite").saveAsTable("orders_enriched")

# Write to Files section (unmanaged / raw zone)
df.write.parquet("Files/raw/orders/2024/01/")

# Append with schema evolution
df.write.format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("orders")
```

#### Merge / Upsert (Delta MERGE INTO)
```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "orders")
source = spark.read.format("delta").table("orders_staging")

target.alias("t") \
    .merge(source.alias("s"), "t.order_id = s.order_id") \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .whenNotMatchedBySourceDelete() \
    .execute()
```

```sql
-- Spark SQL equivalent
MERGE INTO orders AS t
USING orders_staging AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

#### Schema Management
```python
# Schema enforcement (default) — rejects writes with mismatched schema
df.write.format("delta").mode("append").saveAsTable("orders")  # Fails if schema differs

# Schema evolution — allows adding new columns
df.write.format("delta").mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("orders")

# Overwrite schema entirely
df.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable("orders")

# Add/rename/drop columns via ALTER TABLE
spark.sql("ALTER TABLE orders ADD COLUMN (region STRING AFTER customer_id)")
spark.sql("ALTER TABLE orders RENAME COLUMN region TO sales_region")
spark.sql("ALTER TABLE orders DROP COLUMN sales_region")

# Set column comments
spark.sql("ALTER TABLE orders ALTER COLUMN amount COMMENT 'Order total in USD'")
```

#### Partitioning Strategies
```python
# Partition by date — good for time-series, daily ingestion
df.write.format("delta") \
    .partitionBy("order_date") \
    .mode("overwrite") \
    .saveAsTable("orders_partitioned")

# Liquid clustering (Delta 3.x / Runtime 1.3) — PREFERRED over partitioning
# Self-optimizing, no need to choose partition columns upfront
spark.sql("""
    CREATE TABLE orders_clustered
    USING DELTA
    CLUSTER BY (customer_id, order_date)
    AS SELECT * FROM orders
""")

# Alter clustering keys (non-breaking — applies incrementally)
spark.sql("ALTER TABLE orders_clustered CLUSTER BY (region, order_date)")
```

**When to partition**: Only when partition column has low cardinality (<1000 values) and
queries almost always filter on it. Otherwise prefer **liquid clustering**.

**Gotcha**: Over-partitioned tables (e.g., partition by high-cardinality column) create
thousands of tiny files → severe small-file problem → slow reads and costly OPTIMIZE.

#### Z-ORDER (for Runtime 1.2 / Delta 2.x)
```sql
-- Z-ORDER co-locates data by column values within partitions
OPTIMIZE orders ZORDER BY (customer_id, order_date);

-- Not needed with liquid clustering (Runtime 1.3) — clustering replaces Z-ORDER
```

### Delta Table Maintenance

#### OPTIMIZE (Compaction)
```python
# Compact small files into larger ones (target ~1 GB per file)
spark.sql("OPTIMIZE orders")

# With predicate — only compact files in matching partitions
spark.sql("OPTIMIZE orders WHERE order_date >= '2024-01-01'")

# In a notebook using Delta API
from delta.tables import DeltaTable
dt = DeltaTable.forName(spark, "orders")
dt.optimize().executeCompaction()

# With Z-ORDER
dt.optimize().executeZOrderBy("customer_id")
```

#### VACUUM (Remove Old Files)
```python
# Remove files no longer referenced by Delta log (default retention: 7 days)
spark.sql("VACUUM orders")

# Reduce retention (DANGEROUS — breaks time travel for vacuumed versions)
spark.sql("SET spark.databricks.delta.retentionDurationCheck.enabled = false")
spark.sql("VACUUM orders RETAIN 24 HOURS")

# DeltaTable API
dt = DeltaTable.forName(spark, "orders")
dt.vacuum(retentionHours=168)  # 7 days
```

**Gotcha**: VACUUM is not automatic in Fabric. Schedule it via pipeline or cron notebook.
Without regular VACUUM, OneLake storage costs grow unbounded from historical file versions.

#### ANALYZE TABLE (Statistics)
```python
# Collect statistics for query optimization
spark.sql("ANALYZE TABLE orders COMPUTE STATISTICS FOR ALL COLUMNS")

# For specific columns
spark.sql("ANALYZE TABLE orders COMPUTE STATISTICS FOR COLUMNS order_id, customer_id, amount")
```

### Medallion Architecture in Fabric

```
Files/raw/          → Bronze (raw ingestion, append-only)
Tables/bronze_*     → Bronze Delta tables (schema-enforced raw)
Tables/silver_*     → Silver (cleansed, deduplicated, typed)
Tables/gold_*       → Gold (aggregated, business-ready, Direct Lake-ready)
```

```python
# Bronze → Silver transformation
bronze_df = spark.read.format("delta").table("bronze_orders")

silver_df = (
    bronze_df
    .dropDuplicates(["order_id"])
    .withColumn("order_date", F.to_date("order_date_str", "yyyy-MM-dd"))
    .withColumn("amount", F.col("amount").cast("decimal(18,2)"))
    .filter(F.col("order_id").isNotNull())
    .withColumn("_ingested_at", F.current_timestamp())
)

silver_df.write.format("delta").mode("overwrite").saveAsTable("silver_orders")

# Silver → Gold aggregation
gold_df = spark.sql("""
    SELECT
        date_trunc('month', order_date) AS month,
        region,
        COUNT(*) AS order_count,
        SUM(amount) AS total_revenue,
        AVG(amount) AS avg_order_value
    FROM silver_orders
    WHERE order_date >= '2024-01-01'
    GROUP BY 1, 2
""")

gold_df.write.format("delta").mode("overwrite").saveAsTable("gold_monthly_revenue")
```

### mssparkutils — Fabric's Utility Library

```python
# Filesystem operations
mssparkutils.fs.ls("Files/raw/")
mssparkutils.fs.cp("Files/raw/data.csv", "Files/staging/data.csv")
mssparkutils.fs.mv("Files/staging/data.csv", "Files/archive/data.csv")
mssparkutils.fs.rm("Files/temp/", recurse=True)
mssparkutils.fs.mkdirs("Files/output/2024/01/")
mssparkutils.fs.head("Files/raw/sample.csv", maxBytes=1024)
mssparkutils.fs.put("Files/config/params.json", '{"key": "value"}', overwrite=True)

# Cross-workspace access
mssparkutils.fs.ls(
    "abfss://OtherWorkspace@onelake.dfs.fabric.microsoft.com/other_lakehouse.Lakehouse/Files/"
)

# Notebook orchestration
mssparkutils.notebook.run("child_notebook", timeout=600, arguments={"date": "2024-01-15"})
mssparkutils.notebook.runMultiple([
    {"name": "etl_orders", "timeout": 600, "args": {"date": "2024-01-15"}},
    {"name": "etl_customers", "timeout": 600},
])
# runMultiple executes notebooks in parallel (respects DAG if dependencies specified)

# Notebook exit with return value
mssparkutils.notebook.exit("success:42")  # returns string to caller

# Credentials / secrets
token = mssparkutils.credentials.getToken("https://storage.azure.com/")
secret = mssparkutils.credentials.getSecret("https://my-kv.vault.azure.net/", "my-secret")
conn_str = mssparkutils.credentials.getConnectionStringOrCred("MyConnection")

# Runtime context
workspace_id = mssparkutils.runtime.context.get("currentWorkspaceId")
lakehouse_id = mssparkutils.runtime.context.get("lakehouseId")

# Lakehouse management (attach/detach within notebook)
mssparkutils.lakehouse.list()
mssparkutils.lakehouse.get("sales_lakehouse")
```

### Spark SQL — Common Patterns in Fabric

```sql
-- Show all tables in default lakehouse
SHOW TABLES;

-- Describe table schema with column comments
DESCRIBE EXTENDED orders;

-- Table properties (Delta metadata)
SHOW TBLPROPERTIES orders;

-- Table history (Delta log)
DESCRIBE HISTORY orders;

-- Convert Parquet files to managed Delta table
CREATE TABLE bronze_orders
USING DELTA
AS SELECT * FROM parquet.`Files/raw/orders/`;

-- Create table with explicit schema
CREATE TABLE IF NOT EXISTS orders (
    order_id LONG NOT NULL,
    customer_id LONG,
    order_date DATE,
    amount DECIMAL(18,2),
    status STRING,
    _loaded_at TIMESTAMP
)
USING DELTA
CLUSTER BY (customer_id, order_date)
TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact' = 'true'
);

-- Insert overwrite by partition (partial overwrite)
INSERT OVERWRITE TABLE orders
PARTITION (order_date = '2024-01-15')
SELECT order_id, customer_id, '2024-01-15' AS order_date, amount, status, current_timestamp()
FROM staging_orders;

-- Delete with time-travel rollback safety
DELETE FROM orders WHERE status = 'cancelled' AND order_date < '2023-01-01';

-- Restore table to previous version
RESTORE TABLE orders TO VERSION AS OF 42;
RESTORE TABLE orders TO TIMESTAMP AS OF '2024-01-15 08:00:00';

-- Clone a table (zero-copy shallow clone)
CREATE TABLE orders_clone SHALLOW CLONE orders;
-- Deep clone (full copy)
CREATE TABLE orders_backup DEEP CLONE orders;

-- Cross-lakehouse query
SELECT a.*, b.customer_name
FROM sales_lakehouse.orders a
JOIN crm_lakehouse.customers b ON a.customer_id = b.id;
```

### Spark Structured Streaming in Fabric

```python
from pyspark.sql import functions as F
from pyspark.sql.types import *

# Read streaming from Files (auto-loader pattern — new files trigger processing)
schema = StructType([
    StructField("order_id", LongType()),
    StructField("customer_id", LongType()),
    StructField("amount", DoubleType()),
    StructField("event_time", TimestampType()),
])

stream_df = (
    spark.readStream
    .format("cloudFiles")           # Auto Loader (Fabric-supported)
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "Files/_checkpoints/orders_schema/")
    .option("cloudFiles.inferColumnTypes", "true")
    .schema(schema)
    .load("Files/raw/orders_stream/")
)

# Transform
enriched = (
    stream_df
    .withColumn("_ingested_at", F.current_timestamp())
    .withColumn("amount_usd", F.col("amount").cast("decimal(18,2)"))
    .withWatermark("event_time", "1 hour")
)

# Write to Delta table (streaming sink)
query = (
    enriched.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "Files/_checkpoints/orders_bronze/")
    .trigger(availableNow=True)      # Process all available, then stop (micro-batch)
    # .trigger(processingTime="5 minutes")  # Continuous micro-batch
    .toTable("bronze_orders")
)
query.awaitTermination()

# Stream-to-stream: read from Delta, aggregate, write to Gold
orders_stream = spark.readStream.format("delta").table("silver_orders")

agg_stream = (
    orders_stream
    .withWatermark("order_date", "1 day")
    .groupBy(F.window("order_date", "1 day"), "region")
    .agg(
        F.count("*").alias("order_count"),
        F.sum("amount").alias("total_revenue"),
    )
    .select("window.start", "region", "order_count", "total_revenue")
)

agg_query = (
    agg_stream.writeStream
    .format("delta")
    .outputMode("complete")
    .option("checkpointLocation", "Files/_checkpoints/gold_daily_revenue/")
    .trigger(availableNow=True)
    .toTable("gold_daily_revenue")
)
```

**Trigger modes:**
| Trigger | Use case |
|---------|----------|
| `availableNow=True` | Batch-like: process all new files, then stop. Best for scheduled pipeline runs. |
| `processingTime="5 minutes"` | Continuous: micro-batch every N interval. Session must stay alive. |
| `once=True` | Legacy — prefer `availableNow`. |

**Gotcha**: Streaming notebooks keep the Spark session alive and consume CUs continuously
when using `processingTime`. For cost control, prefer `availableNow=True` triggered by a
pipeline on a schedule.

### Spark Job Definitions (Headless Batch)

For production Spark jobs that don't need a notebook UI:

```bash
# Create a Spark Job Definition via REST
curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/sparkjobdefinitions" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "daily_etl_job",
    "definition": {
      "format": "SparkJobDefinitionV1",
      "parts": [{
        "path": "SparkJobDefinitionV1.json",
        "payload": "{\"executableFile\": \"Files/jobs/daily_etl.py\", \"defaultLakehouse\": {\"name\": \"sales_lakehouse\"}, \"language\": \"python\", \"args\": [\"--date\", \"2024-01-15\"]}"
      }]
    }
  }'
```

```python
# daily_etl.py — standalone Spark script
import sys
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("daily_etl").getOrCreate()

date_param = sys.argv[sys.argv.index("--date") + 1] if "--date" in sys.argv else None

df = spark.read.format("delta").table("bronze_orders")
if date_param:
    df = df.filter(f"order_date = '{date_param}'")

result = df.groupBy("region").agg({"amount": "sum"})
result.write.format("delta").mode("overwrite").saveAsTable("gold_revenue_by_region")

spark.stop()
```

### Spark Performance Tuning in Fabric

#### Auto-Optimize Settings
```python
# Per-table auto-optimize (recommended for all tables)
spark.sql("""
    ALTER TABLE orders SET TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact' = 'true'
    )
""")
# optimizeWrite: coalesces small writes into ~128 MB files
# autoCompact: triggers mini-OPTIMIZE after writes when small files accumulate
```

#### Adaptive Query Execution (AQE)
```python
# AQE is ON by default in Fabric. Key settings:
spark.conf.set("spark.sql.adaptive.enabled", "true")                    # default
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true") # default
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")           # default

# Adjust shuffle partition count (default 200 — AQE coalesces automatically)
spark.conf.set("spark.sql.shuffle.partitions", "auto")  # let AQE decide
```

#### Broadcast Joins
```python
from pyspark.sql import functions as F

# Force broadcast for small dimension tables (<100 MB)
small_df = spark.read.format("delta").table("dim_products")  # ~50 MB
large_df = spark.read.format("delta").table("fact_orders")   # 100 GB

result = large_df.join(F.broadcast(small_df), "product_id")

# Adjust broadcast threshold (default 10 MB)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100m")  # 100 MB
```

#### Caching
```python
# Cache hot tables in memory (Spark executor memory)
df = spark.read.format("delta").table("dim_customers")
df.cache()
df.count()  # Materialize cache

# Uncache when done
df.unpersist()

# CACHE TABLE (SQL)
spark.sql("CACHE TABLE dim_customers")
spark.sql("UNCACHE TABLE dim_customers")
```

**Gotcha**: Caching uses executor memory. On small capacity SKUs (F2/F4), caching large
tables can cause OOM. Use caching sparingly and only for repeatedly-read small tables.

#### Common Performance Anti-Patterns
| Anti-pattern | Fix |
|-------------|-----|
| `collect()` on large DataFrame | Use `take(n)`, `head(n)`, or write to table |
| Python UDFs in hot path | Use Spark SQL functions or Pandas UDFs |
| `count()` just to check emptiness | Use `df.head(1)` or `df.isEmpty()` |
| Writing many small files | Enable `optimizeWrite`, use `coalesce()` before write |
| Reading all columns (`SELECT *`) | Project only needed columns |
| Unbounded `crossJoin` | Add explicit join condition or filter |
| Repartition to 1 for single-file output | Use `coalesce(1)` instead (no shuffle) |
| Skewed join on high-cardinality key | Enable AQE skew join, or salt the key |

### Spark 4.0 New Syntax Examples (Runtime 2.0)

```sql
-- VARIANT data type — store semi-structured data natively
CREATE TABLE events (
    event_id LONG,
    event_time TIMESTAMP,
    payload VARIANT    -- replaces storing JSON as STRING
);
INSERT INTO events VALUES (1, current_timestamp(), PARSE_JSON('{"action": "click", "page": "/home"}'));
SELECT event_id, payload:action::STRING AS action FROM events;  -- schema-on-read

-- Pipe syntax — readable query chaining (left-to-right)
FROM orders
|> WHERE amount > 100
|> SELECT customer_id, amount
|> ORDER BY amount DESC
|> LIMIT 10;

-- Session variables
DECLARE VARIABLE batch_date DATE DEFAULT CURRENT_DATE();
SET VARIABLE batch_date = DATE '2024-06-15';
SELECT * FROM orders WHERE order_date = batch_date;

-- SQL user-defined functions
CREATE FUNCTION discount(amount DECIMAL(18,2), pct DECIMAL(5,2))
RETURNS DECIMAL(18,2)
RETURN amount * (1 - pct / 100);
SELECT order_id, discount(amount, 10) AS discounted FROM orders;
```

```python
# PySpark native plotting (Spark 4.0) — no matplotlib needed
df = spark.read.format("delta").table("gold_monthly_revenue")
pdf = df.toPandas()
# In Runtime 2.0 notebooks, df.plot() works natively:
df.select("month", "total_revenue").toPandas().plot(x="month", y="total_revenue", kind="bar")

# Python UDTF — return multiple rows from a function
from pyspark.sql.functions import udtf

@udtf(returnType="word: string, count: int")
class WordCount:
    def eval(self, text: str):
        for word in text.split():
            yield word.lower(), 1

spark.sql("SELECT * FROM WordCount('Hello World Hello')")
# Returns: (hello, 1), (world, 1), (hello, 1)

# Python Data Source API — custom data source in pure Python
from pyspark.sql.datasource import DataSource, DataSourceReader

class MyApiSource(DataSource):
    @classmethod
    def name(cls): return "myapi"
    def reader(self, schema): return MyApiReader(self.options)

class MyApiReader(DataSourceReader):
    def __init__(self, options): self.url = options["url"]
    def read(self, partition):
        import requests
        for row in requests.get(self.url).json():
            yield tuple(row.values())

# Structured Streaming State Data Source — inspect streaming state
state_df = spark.read.format("statestore") \
    .option("checkpointLocation", "Files/_checkpoints/my_stream/") \
    .load()
state_df.show()  # Debug what's in the streaming state store
```

### Data Science & MLflow in Fabric

```python
import mlflow
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# MLflow auto-logging (enabled by default in Fabric notebooks)
mlflow.autolog()

# Load data from lakehouse
df = spark.read.format("delta").table("silver_customers").toPandas()
X = df[["age", "income", "tenure"]]
y = df["churn"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

# Train with MLflow tracking
with mlflow.start_run(run_name="rf_churn_v1") as run:
    model = RandomForestClassifier(n_estimators=100, max_depth=10)
    model.fit(X_train, y_train)
    accuracy = model.score(X_test, y_test)
    mlflow.log_metric("accuracy", accuracy)
    mlflow.sklearn.log_model(model, "model")
    print(f"Run ID: {run.info.run_id}, Accuracy: {accuracy:.4f}")

# Register model
mlflow.register_model(f"runs:/{run.info.run_id}/model", "churn_predictor")

# Load and predict with registered model
loaded_model = mlflow.pyfunc.load_model("models:/churn_predictor/latest")
predictions = loaded_model.predict(X_test)
```

#### PREDICT function (T-SQL inference)
```sql
-- Use MLflow model directly in warehouse T-SQL queries
SELECT *, PREDICT('churn_predictor', 1) AS churn_prediction
FROM customers;
```

### Notebook Scheduling
```python
# Notebooks can be scheduled directly (without a Pipeline) via REST
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/items/$NOTEBOOK_ID/jobs/instances?jobType=RunNotebook" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json"

# Or embed in a Pipeline as a Notebook Activity with parameters:
# Pipeline → Add Activity → Notebook → Settings → Base parameters
```

### Fabric Spark vs Databricks — Key Differences
| Feature | Fabric Spark | Databricks |
|---------|-------------|------------|
| Cluster management | Serverless (no clusters) | Interactive + job clusters |
| Spark version | 4.0 (RT 2.0) / 3.5 (RT 1.3) | Spark 3.5+ (DBR 15+) |
| Delta version | 4.0 (RT 2.0) / 3.2 (RT 1.3) | Delta 3.x (Unity Catalog) |
| Unity Catalog | No (OneLake + workspace RBAC) | Yes |
| Photon/Native engine | Native Execution Engine (up to 6×) | Photon |
| dbutils | mssparkutils (similar API) | dbutils |
| Auto Loader | cloudFiles format | cloudFiles (identical API) |
| Repos/Git | Workspace Git integration | Repos + Asset Bundles |
| Billing | CU-based (shared capacity) | DBU-based (per cluster) |
| Cold start | 2-3 min (Custom Live Pools help) | 3-8 min (cluster spin-up) |
| Liquid clustering | ✅ | ✅ |
| Streaming | Structured Streaming | Structured Streaming + DLT |
| Scala version | 2.13 (RT 2.0) / 2.12 (RT 1.3) | 2.12 or 2.13 |
| Java version | 21 (RT 2.0) / 11 (RT 1.3) | 17 (DBR 14+) |

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

### KQL Databases & Eventhouses
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

### Direct Lake Guardrails by SKU
| SKU  | Max rows per table | Max model size | Max columns per table |
|------|-------------------|----------------|----------------------|
| F2   | 300M              | 10 GB          | 1,500                |
| F8   | 300M              | 10 GB          | 1,500                |
| F64  | 1.5B              | 25 GB          | 1,500                |
| F128 | 3B                | 50 GB          | 1,500                |
| F256 | 6B                | 100 GB         | 1,500                |

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

## Fabric REST API Patterns

```python
import requests

# Get a Fabric token (service principal or user)
# Via Azure CLI:
# az account get-access-token --resource https://api.fabric.microsoft.com --query accessToken -o tsv

BASE = "https://api.fabric.microsoft.com/v1"
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# List workspaces
r = requests.get(f"{BASE}/workspaces", headers=headers)
workspaces = r.json()["value"]

# List items in a workspace
r = requests.get(f"{BASE}/workspaces/{ws_id}/items", headers=headers)
items = r.json()["value"]

# Run a notebook on demand
r = requests.post(
    f"{BASE}/workspaces/{ws_id}/items/{notebook_id}/jobs/instances?jobType=RunNotebook",
    headers=headers,
)
job_id = r.json()["id"]

# Poll job status
r = requests.get(
    f"{BASE}/workspaces/{ws_id}/items/{notebook_id}/jobs/instances/{job_id}",
    headers=headers,
)
status = r.json()["status"]  # InProgress, Completed, Failed
```

---

## Semantic Link (Python SDK for Fabric)

```python
import sempy.fabric as fabric

# List datasets in workspace
datasets = fabric.list_datasets()

# Read Power BI table into Pandas (via XMLA)
df = fabric.read_table("My Semantic Model", "Sales")

# Evaluate DAX
result = fabric.evaluate_dax(
    "My Semantic Model",
    "EVALUATE SUMMARIZE(Sales, Sales[Region], \"Total\", SUM(Sales[Amount]))"
)

# List tables in a semantic model
tables = fabric.list_tables("My Semantic Model")

# Refresh a semantic model
fabric.refresh_dataset("My Semantic Model")
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

9. **Delta table OPTIMIZE is not automatic**: Small-file accumulation from streaming or
   frequent micro-batch writes will degrade performance. Schedule OPTIMIZE via pipeline:
   ```sql
   OPTIMIZE sales_lakehouse.dbo.orders;        -- from warehouse
   ```
   ```python
   spark.sql("OPTIMIZE sales_lakehouse.orders") # PySpark
   ```

10. **VACUUM is not automatic either**: Without VACUUM, historical Delta file versions
    accumulate and increase OneLake storage costs. Schedule weekly VACUUM.

11. **F2/F4 limitations**: Very small SKUs lack some features (XMLA write requires F64+).

12. **Environment publish is slow**: 3-10 minutes. Don't add library installs to
    notebook cells that run on every execution — use Environments instead.

13. **Spark session timeout**: Default 20 minutes idle timeout. Long-running streaming
    jobs need `processingTime` trigger to keep the session alive, or use pipeline scheduling.

14. **No Delta Sharing server**: Fabric supports reading external Delta Sharing sources
    but does not act as a Delta Sharing server. To share OneLake data externally, use
    shortcuts or direct ABFS access with SAS tokens.

15. **Notebook output size limit**: Rich display output (charts, dataframes) is capped at
    ~20 MB per cell. For large result sets, write to table and visualize via Power BI.
