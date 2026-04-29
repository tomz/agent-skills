---
name: azure-hdinsight-migration-spark-to-fabric
description: End-to-end migration playbook for moving Apache Spark workloads from Azure HDInsight (3.6/4.0/5.x) to Microsoft Fabric — assessment, code conversion, storage migration (WASB/ADLS → OneLake), Hive Metastore → Lakehouse, Livy/Oozie → Pipelines/Notebooks, security mapping (Ranger/ESP → Workspace RBAC + OneLake roles), cutover, and validation.
license: MIT
version: 1.0.0
updated: 2026-04-29
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
triggers: hdinsight, fabric, migration, migrate, spark migration, hdi to fabric, livy, oozie, wasb, adls to onelake, hive metastore, ranger, esp
requires: azure-hdinsight, azure-fabric
---
# Migrating HDInsight Spark → Microsoft Fabric Spark

A field guide for migrating production Apache Spark workloads from **Azure HDInsight**
(classic 3.6/4.0/5.x or HDInsight on AKS) to **Microsoft Fabric** (Runtime 1.3 / 2.0).
Covers assessment, code rewrites, storage cutover, orchestration, security, and validation.

> **Why this migration?** HDInsight 4.0 reached end-of-standard-support; 5.x is the
> last classic generation. Microsoft's recommended landing zones for HDInsight Spark
> are **Microsoft Fabric** (SaaS, OneLake, Direct Lake) or **Azure Databricks** (PaaS,
> Unity Catalog). This skill targets Fabric.

---

## 1. Decision Matrix — Is Fabric the Right Target?

| Workload Pattern | Recommended Target | Reason |
|------------------|-------------------|--------|
| Batch ETL → Power BI reporting | **Fabric** | Direct Lake, OneLake, single bill |
| Interactive notebooks for analysts | **Fabric** | Built-in notebooks, Git integration |
| ML training + MLflow | **Fabric** or Databricks | Both have MLflow; Databricks more mature |
| Streaming with Kafka source | **Fabric Eventstream + Spark** | Or keep Kafka, point Fabric at it |
| Heavy Scala JAR jobs, custom Spark internals | **Databricks** | More Spark version flexibility |
| Delta Sharing producer | **Databricks** | Fabric is consumer-only |
| Tight Hadoop-ecosystem coupling (HBase, Storm, Pig) | **Migrate components first** | Fabric has no HBase/Storm equivalent |
| Cost-sensitive, bursty workloads | **Fabric** | CU model > always-on HDI head nodes |
| Strict VNet isolation, custom kernel | **Databricks (VNet injection)** | Fabric private link is newer |

**Decision shortcut**: If the workload's output goes to Power BI and the team is
comfortable with SaaS, choose Fabric. If it's part of a broader data platform with
Unity Catalog ambitions or needs Spark version pinning, choose Databricks.

---

## 2. Assessment Phase

### 2.1 Inventory the HDInsight Cluster
Run on the head node (SSH) or via Ambari API.

```bash
# SSH to head node
ssh sshuser@<cluster>-ssh.azurehdinsight.net

# Cluster metadata
curl -s -u admin:$PASS \
  "https://<cluster>.azurehdinsight.net/api/v1/clusters/<cluster>" \
  | python3 -m json.tool > cluster_meta.json

# Spark / Hadoop component versions
hdp-select status | tee component_versions.txt
spark-submit --version 2>&1 | tee spark_version.txt
hive --version 2>&1 | tee hive_version.txt

# Storage accounts referenced (core-site.xml)
sudo grep -E "wasb|abfs" /etc/hadoop/conf/core-site.xml \
  | tee storage_refs.txt

# All Hive tables + their storage locations
beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
        -n admin -p $PASS \
        --silent=true --outputformat=tsv2 \
        -e "SHOW DATABASES;" > databases.tsv

while read db; do
  beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
          -n admin -p $PASS --silent=true --outputformat=tsv2 \
          -e "USE $db; SHOW TABLES;" \
    | awk -v db=$db 'NR>1 {print db"."$1}'
done < databases.tsv > all_tables.txt

# Per-table DDL + size estimate
while read fq; do
  db=$(echo $fq | cut -d. -f1); tb=$(echo $fq | cut -d. -f2)
  echo "=== $fq ===" >> table_inventory.txt
  beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
          -n admin -p $PASS --silent=true --outputformat=tsv2 \
          -e "DESCRIBE FORMATTED $db.$tb;" >> table_inventory.txt
done < all_tables.txt
```

### 2.2 Inventory the Spark Codebase
```bash
# Job repo audit (replace with your repo path)
cd /path/to/spark-jobs

# 1) Spark version assumptions in code
grep -RInE "spark\.version|SparkSession\.builder|spark-submit" --include="*.py" --include="*.scala" --include="*.sh"

# 2) WASB / DBFS / HDFS path usage
grep -RInE "wasbs?://|hdfs://|/user/hive/warehouse|adl://" --include="*.py" --include="*.scala" --include="*.sql"

# 3) Hive Metastore-bound APIs
grep -RInE "spark\.sql|spark\.table|saveAsTable|enableHiveSupport" --include="*.py" --include="*.scala"

# 4) Custom JARs / dependencies
find . -name "build.sbt" -o -name "pom.xml" -o -name "requirements.txt" -o -name "setup.py" | xargs ls -la

# 5) Livy batch invocations (orchestration code)
grep -RInE "/livy/batches|livy\.endpoint" --include="*.py" --include="*.sh" --include="*.json"

# 6) Oozie workflows
find . -name "workflow.xml" -o -name "coordinator.xml" -o -name "job.properties"

# 7) Kerberos / ESP code paths
grep -RInE "kinit|krb5|UserGroupInformation|hadoop\.security" --include="*.py" --include="*.scala" --include="*.sh"

# 8) HBase / Phoenix / Hive ACID dependencies
grep -RInE "org\.apache\.hbase|org\.apache\.phoenix|TBLPROPERTIES.*transactional" --include="*.py" --include="*.scala" --include="*.sql"
```

### 2.3 Categorize Each Job
Build a CSV: `job_name, type, spark_version, storage, hms_dependent, custom_jars, schedule, owner, complexity, notes`.

| Complexity | Definition | Migration Effort |
|------------|------------|------------------|
| **Low**    | PySpark, ADLS Gen2, no custom JAR, no UDF, batch | 1-2 hours |
| **Medium** | Custom Python deps, Hive ACID, partitioned tables, HMS lookups | 0.5-2 days |
| **High**   | Scala JARs, JNI/native code, HBase/Phoenix coupling, ESP+Ranger | 1-2 weeks |
| **Blocker**| Storm, Pig, MapReduce, Oozie with shell actions, custom kernels | Re-architect |

---

## 3. Compatibility Mapping

### 3.1 Spark / Delta Versions

| HDInsight Cluster | Spark | Delta | → Fabric Target |
|-------------------|-------|-------|-----------------|
| HDI 3.6           | 2.3   | n/a (Parquet only) | RT 1.3 (Spark 3.5) — significant code changes |
| HDI 4.0           | 2.4   | 0.x (preview)      | RT 1.3 (Spark 3.5) — significant code changes |
| HDI 5.0           | 3.1   | 1.x                | RT 1.3 (Spark 3.5) — moderate changes |
| HDI 5.1           | 3.3   | 2.x                | RT 1.3 (Spark 3.5) — minor changes |
| HDI on AKS Spark 3.4 | 3.4 | 2.4              | RT 1.3 (Spark 3.5) or RT 2.0 (Spark 4.0) |

**Rule of thumb**: Land on **Runtime 1.3 (Spark 3.5, Delta 3.2)** first — it's GA and
the most stable. Plan a follow-up upgrade to Runtime 2.0 (Spark 4.0) once it goes GA
and your code is verified.

### 3.2 Scala / Java

| HDInsight | Fabric RT 1.3 | Fabric RT 2.0 |
|-----------|---------------|---------------|
| Scala 2.11 (HDI 3.6) | **Recompile required** → 2.12 | **Recompile required** → 2.13 |
| Scala 2.12 (HDI 4.0/5.x) | Compatible | **Recompile required** → 2.13 |
| Java 8 (HDI 3.6/4.0) | **Rebuild** for Java 11 | **Rebuild** for Java 21 |
| Java 11 (HDI 5.x) | Compatible | **Test** on Java 21 |

### 3.3 Component Replacements (No 1:1 Equivalent)

| HDInsight Component | Fabric Replacement | Notes |
|---------------------|-------------------|-------|
| Hive Metastore (external) | **Lakehouse + SQL endpoint** | Tables become managed Delta in Lakehouse |
| Livy REST API | **Notebook REST API + Spark Job Definition** | Different auth (Entra ID), different payload |
| Oozie | **Fabric Pipelines** | Workflow.xml → pipeline JSON. Shell actions → Web Activity / Notebook |
| Ambari | **Workspace settings + Capacity Metrics app** | No per-node config |
| Ranger | **Workspace roles + OneLake item permissions + Lakehouse SQL RLS/CLS** | Coarser; plan policy redesign |
| ESP / Kerberos | **Entra ID auth (built-in)** | No more keytabs |
| Custom kernel installs | **Environment libraries** | YAML / wheel upload only |
| HDFS local | **OneLake (`abfss://...onelake.dfs.fabric.microsoft.com/...`)** | No local HDFS — design ephemeral |
| HBase | **Cosmos DB / KQL DB / Azure Table Storage** | No HBase in Fabric |
| Phoenix SQL on HBase | **Cosmos DB SQL or KQL** | Re-platform |
| Storm | **Eventstream + Spark Structured Streaming** | Re-architect |
| Pig | **PySpark / SparkSQL** | Hand-rewrite |

---

## 4. Storage Migration: WASB / ADLS Gen2 → OneLake

### 4.1 Pattern A — Shortcut (Recommended for Cutover)
Create a **Fabric shortcut** from your Lakehouse `Tables/` or `Files/` to the existing
ADLS Gen2 container. **No data movement.** Run jobs in parallel against both engines
during cutover, then flip writers and copy/delete the old data later.

```python
# Create shortcut to existing ADLS Gen2 path (Fabric REST)
import requests
payload = {
    "name": "legacy_orders",
    "path": "Tables",                          # or "Files"
    "target": {
        "type": "AdlsGen2",
        "adlsGen2": {
            "location": "https://mystorage.dfs.core.windows.net",
            "subpath": "/legacy/sales/orders/",
            "connectionId": "<conn-id-from-onelake-data-hub>"
        }
    }
}
requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WS_ID}/lakehouses/{LH_ID}/shortcuts",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    json=payload,
)
```

**Limitations**: Shortcut tables are **read-only via the SQL endpoint**. Writes must go
through Spark to the underlying ADLS path. Direct Lake against shortcut tables works
but may fall back to DirectQuery if V-Order is missing.

### 4.2 Pattern B — One-Time Copy into OneLake
Copy historical data into managed OneLake Delta tables, then point new writes at the
Lakehouse and decommission the old container.

**Option B1 — `azcopy` for raw files** (Files section):
```bash
# Get OneLake SAS-equivalent: use Entra ID auth via azcopy login
azcopy login --tenant-id <tenant>

azcopy copy \
  "https://mystorage.dfs.core.windows.net/container/legacy/" \
  "https://onelake.dfs.fabric.microsoft.com/<workspace>/<lakehouse>.Lakehouse/Files/legacy/" \
  --recursive=true \
  --preserve-permissions=false \
  --check-md5=FailIfDifferent
```

**Option B2 — Spark CTAS for tables** (best for converting Parquet/ORC → Delta with V-Order):
```python
# Run in a Fabric notebook attached to the destination Lakehouse
src = "abfss://container@mystorage.dfs.core.windows.net/legacy/sales/orders/"

# Read source (Parquet, ORC, or non-V-Order Delta from HDI)
src_df = spark.read.format("parquet").load(src)
# For ORC from Hive: spark.read.format("orc").load(src)
# For HDI Delta:    spark.read.format("delta").load(src)

# Write as managed Delta in Fabric Lakehouse (V-Order enabled by default)
(src_df.write
   .format("delta")
   .mode("overwrite")
   .option("overwriteSchema", "true")
   .saveAsTable("sales_lakehouse.orders"))
```

**Option B3 — Pipeline Copy Activity** (no-code, scales well, integrates with monitoring):
```
Pipeline → Copy data activity
   Source: ADLS Gen2 dataset → /legacy/sales/orders/ (Parquet)
   Sink:   Lakehouse table dataset → sales_lakehouse / orders
   Settings: Copy behavior = "Merge files", Max parallel copies = 16
```

### 4.3 Path Rewriting Cheat Sheet
Replace these patterns in your Spark code:

| Old (HDInsight)                                                       | New (Fabric)                                                                                          |
|-----------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| `wasbs://container@account.blob.core.windows.net/path/`               | `abfss://container@account.dfs.core.windows.net/path/` (if keeping ADLS) or OneLake ABFS (if cutover) |
| `abfss://container@account.dfs.core.windows.net/path/`                | `abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>.Lakehouse/Files/path/`              |
| `/user/hive/warehouse/sales.db/orders`                                | `abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>.Lakehouse/Tables/orders`            |
| `hdfs:///tmp/staging/`                                                | Use `Files/_tmp/` in default Lakehouse, or executor-local `/tmp/` for ephemeral data                  |
| `spark.sql("USE sales")` then `spark.table("orders")`                 | `spark.table("sales_lakehouse.orders")` (Lakehouse name = database)                                   |

Automate with a one-off script:
```bash
# Dry-run path rewrite across the codebase (review diff before applying)
WORKSPACE=MyWorkspace; LAKEHOUSE=sales_lakehouse
grep -RIn "wasbs://" --include="*.py" --include="*.scala" --include="*.sql" .

# Apply (commit-by-commit, not bulk!)
sed -i.bak \
  -e "s|wasbs://\([^@]*\)@\([^./]*\)\.blob\.core\.windows\.net|abfss://${WORKSPACE}@onelake.dfs.fabric.microsoft.com/${LAKEHOUSE}.Lakehouse/Files|g" \
  path/to/file.py
```

---

## 5. Hive Metastore → Fabric Lakehouse

HDInsight relies on an **external Hive Metastore** (Azure SQL DB) that holds table
definitions. Fabric Lakehouse uses its own catalog (no separate HMS). Migration steps:

### 5.1 Export DDL from HMS
```bash
# Per-database DDL dump (run on HDI head node)
beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
        -n admin -p $PASS --silent=true --outputformat=tsv2 \
        -e "SHOW DATABASES" | tail -n +2 > /tmp/dbs.txt

mkdir -p ddl_export
while read db; do
  beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
          -n admin -p $PASS --silent=true --outputformat=tsv2 \
          -e "USE $db; SHOW TABLES" | tail -n +2 > /tmp/tables_$db.txt

  while read tb; do
    beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
            -n admin -p $PASS --silent=true --outputformat=tsv2 \
            -e "SHOW CREATE TABLE $db.$tb" \
      > "ddl_export/${db}.${tb}.sql"
  done < /tmp/tables_$db.txt
done < /tmp/dbs.txt
```

### 5.2 Translate DDL to Fabric/Spark Delta
Hive DDL needs three transformations:

1. `STORED AS ORC` / `STORED AS PARQUET` → `USING DELTA`
2. `LOCATION 'wasbs://...'` → drop (let Fabric pick managed location) **or** rewrite to `abfss://...onelake...`
3. `TBLPROPERTIES ("transactional" = "true")` → drop (Delta is always transactional)
4. `CLUSTERED BY ... INTO N BUCKETS` → drop, replace with `CLUSTER BY (cols)` (Liquid Clustering)
5. `PARTITIONED BY (col)` → keep, OR replace with liquid clustering for high-cardinality columns

**Example translation**:
```sql
-- HDInsight Hive DDL (source)
CREATE TABLE sales.orders (
    order_id   BIGINT,
    customer_id BIGINT,
    amount     DECIMAL(18,2),
    order_date DATE
)
PARTITIONED BY (load_date STRING)
CLUSTERED BY (customer_id) INTO 16 BUCKETS
STORED AS ORC
LOCATION 'wasbs://datalake@hdistore.blob.core.windows.net/warehouse/sales/orders'
TBLPROPERTIES ("transactional" = "true");

-- Fabric Lakehouse DDL (target — run in notebook)
CREATE TABLE sales_lakehouse.orders (
    order_id   BIGINT,
    customer_id BIGINT,
    amount     DECIMAL(18,2),
    order_date DATE,
    load_date  STRING
)
USING DELTA
CLUSTER BY (customer_id, order_date)         -- Liquid Clustering replaces buckets + partitioning
TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact'   = 'true'
);
```

### 5.3 Programmatic Translation Helper
```python
# Drop into a Fabric notebook to bulk-translate DDL files
import re, pathlib

DDL_DIR = pathlib.Path("/lakehouse/default/Files/ddl_export")
OUT_DIR = pathlib.Path("/lakehouse/default/Files/ddl_fabric")
OUT_DIR.mkdir(exist_ok=True)
LAKEHOUSE = "sales_lakehouse"

def translate(ddl: str, lh: str) -> str:
    # 1) STORED AS X → USING DELTA
    ddl = re.sub(r"STORED\s+AS\s+(ORC|PARQUET|TEXTFILE)", "USING DELTA", ddl, flags=re.I)
    # 2) Drop LOCATION
    ddl = re.sub(r"LOCATION\s+'[^']*'", "", ddl, flags=re.I)
    # 3) Drop transactional / bucket / serde props
    ddl = re.sub(r"CLUSTERED\s+BY\s*\([^)]*\)\s*INTO\s+\d+\s+BUCKETS", "", ddl, flags=re.I)
    ddl = re.sub(r'TBLPROPERTIES\s*\([^)]*"transactional"[^)]*\)', "", ddl, flags=re.I)
    # 4) Schema-qualify with lakehouse name
    ddl = re.sub(r"CREATE\s+TABLE\s+(\w+)\.(\w+)",
                 f"CREATE TABLE {lh}.\\2", ddl, flags=re.I)
    return ddl + "\nTBLPROPERTIES ('delta.autoOptimize.optimizeWrite'='true');"

for f in DDL_DIR.glob("*.sql"):
    out = translate(f.read_text(), LAKEHOUSE)
    (OUT_DIR / f.name).write_text(out)
print("Translated", len(list(OUT_DIR.glob('*.sql'))), "DDL files")

# Apply
for f in sorted(OUT_DIR.glob("*.sql")):
    try:
        spark.sql(f.read_text())
        print("✓", f.name)
    except Exception as e:
        print("✗", f.name, "→", str(e)[:120])
```

---

## 6. Spark Code Conversion

### 6.1 SparkSession Builder
```python
# HDInsight (typical)
from pyspark.sql import SparkSession
spark = (SparkSession.builder
         .appName("etl")
         .enableHiveSupport()                   # ← REMOVE in Fabric
         .config("spark.sql.warehouse.dir",
                 "wasbs://...")                 # ← REMOVE
         .getOrCreate())

# Fabric — `spark` is pre-instantiated in notebooks. In a Spark Job Definition:
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("etl").getOrCreate()
# enableHiveSupport() is unnecessary — Lakehouse catalog is the default.
```

### 6.2 Reading & Writing Tables
```python
# HDInsight pattern
df = spark.sql("SELECT * FROM sales.orders WHERE order_date = '2024-01-15'")
df.write.mode("overwrite").saveAsTable("sales.orders_daily")

# Fabric — replace database name with Lakehouse name (no other change)
df = spark.sql("SELECT * FROM sales_lakehouse.orders WHERE order_date = '2024-01-15'")
df.write.format("delta").mode("overwrite").saveAsTable("sales_lakehouse.orders_daily")
# `format("delta")` is the implicit default in Fabric, but be explicit for clarity.
```

### 6.3 Hive ACID MERGE → Delta MERGE
```sql
-- HDInsight Hive ACID
MERGE INTO transactions AS t
USING transactions_staging AS s
ON  (t.txn_id = s.txn_id)
WHEN MATCHED     THEN UPDATE SET amount = s.amount, txn_date = s.txn_date
WHEN NOT MATCHED THEN INSERT VALUES (s.txn_id, s.account_id, s.amount, s.txn_date);

-- Fabric Delta MERGE — identical syntax (Delta MERGE is a superset)
MERGE INTO sales_lakehouse.transactions AS t
USING sales_lakehouse.transactions_staging AS s
ON  (t.txn_id = s.txn_id)
WHEN MATCHED     THEN UPDATE SET amount = s.amount, txn_date = s.txn_date
WHEN NOT MATCHED THEN INSERT *;            -- INSERT * works in Delta, simpler
-- Bonus: Delta supports WHEN NOT MATCHED BY SOURCE THEN DELETE (Hive doesn't)
```

### 6.4 Common Per-API Replacements

| HDInsight API / Idiom | Fabric Replacement |
|-----------------------|-------------------|
| `spark.read.format("com.databricks.spark.csv")` | `spark.read.format("csv")` (built-in) |
| `spark.read.format("com.databricks.spark.avro")` | `spark.read.format("avro")` (Fabric pre-installed) |
| `dbutils.fs.ls(...)` (Databricks-style code on HDI) | `mssparkutils.fs.ls(...)` |
| `dbutils.notebook.run(...)` | `mssparkutils.notebook.run(...)` |
| `dbutils.secrets.get(scope, key)` | `mssparkutils.credentials.getSecret(kv_url, key)` |
| `sc.textFile("hdfs://...")` | `sc.textFile("abfss://...onelake...")` |
| `kinit` + UGI for Kerberos | Remove — Entra ID handled automatically |
| `org.apache.hadoop.hbase.*` | Re-platform to Cosmos DB / KQL DB |
| Phoenix JDBC connector | Re-platform to Lakehouse SQL endpoint or Warehouse |
| `org.apache.spark.sql.streaming.Trigger.Continuous` | `Trigger.AvailableNow()` (preferred) or `Trigger.ProcessingTime()` |

### 6.5 Custom JAR / Wheel Migration
```bash
# Build for the target runtime
# Scala JARs:
sbt clean +package        # Cross-compile if 2.12 → 2.13 needed
# OR Maven:
mvn clean package -Dscala.version=2.13.12 -Dspark.version=3.5.0   # for RT 1.3
mvn clean package -Dscala.version=2.13.12 -Dspark.version=4.0.0   # for RT 2.0

# Python wheels:
python -m build             # produces dist/myapp-1.0.0-py3-none-any.whl
```
Upload via **Environment** in Fabric (Workspace → New → Environment → Custom Libraries).
Publish (3-10 min build), then attach the Environment to your notebook or Spark Job
Definition.

---

## 7. Orchestration Migration

### 7.1 Livy Batches → Fabric Notebook / SJD REST
```python
# OLD — HDInsight Livy
import requests
resp = requests.post(
    f"https://{cluster}.azurehdinsight.net/livy/batches",
    auth=("admin", PASS),
    json={
        "file": "wasbs://.../my_job.py",
        "args": ["--date", "2024-01-15"],
        "numExecutors": 4,
        "executorMemory": "4g",
    },
    headers={"Content-Type": "application/json"},
)
batch_id = resp.json()["id"]

# NEW — Fabric Notebook on-demand run
import requests
token = "<entra-id-token-for-fabric-api>"
resp = requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WS_ID}/items/{NOTEBOOK_ID}/jobs/instances?jobType=RunNotebook",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    json={                                     # parameters override notebook defaults
        "executionData": {
            "parameters": {
                "date": {"value": "2024-01-15", "type": "string"}
            }
        }
    },
)
job_id = resp.headers["Location"].split("/")[-1]

# NEW — Fabric Spark Job Definition (better for headless production jobs)
resp = requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WS_ID}/items/{SJD_ID}/jobs/instances?jobType=sparkjob",
    headers={"Authorization": f"Bearer {token}"},
)
```

### 7.2 Oozie → Fabric Pipelines
**Coordinator + Workflow XML** maps to **Pipeline + Schedule**.

| Oozie Concept | Fabric Equivalent |
|---------------|-------------------|
| `<coordinator-app>` with `frequency` | Pipeline schedule trigger (UTC only — see gotcha) |
| `<workflow-app>` | Pipeline canvas |
| `<spark>` action | Notebook activity OR Spark Job Definition activity |
| `<shell>` action | Web Activity (call Azure Function) — Fabric has no shell action |
| `<hive>` action | Lakehouse SQL via Notebook (`spark.sql`) |
| `<email>` action | Office 365 Outlook activity |
| `<decision>` (case branch) | If Condition activity |
| `<fork>` / `<join>` | Parallel branches in canvas (auto-join at convergence) |
| `<sla>` | Pipeline alert rules + Capacity Metrics |
| Job properties (`${date}`) | Pipeline parameters |

**Migration script**: write a small Python tool that walks `workflow.xml`, emits a
pipeline JSON skeleton (one Notebook activity per `<spark>` action), then hand-edit
in the Fabric canvas.

---

## 8. Security Mapping

### 8.1 ESP / Kerberos → Entra ID
HDInsight ESP uses Kerberos tickets bound to AAD Domain Services. Fabric uses **Entra
ID** directly — no AADDS, no keytabs, no `kinit`.

| HDInsight (ESP) | Fabric |
|-----------------|--------|
| Domain user `alice@corp.onmicrosoft.com` + Kerberos | Entra ID user `alice@corp.com` |
| `kinit alice; spark-submit` | Sign in to portal / use Entra ID token in API calls |
| Service principal with keytab | Service principal with **Workspace Contributor** role |
| Managed Identity for storage | Managed Identity for storage (same — keep) |

**Action item**: Remove all `kinit`, `UserGroupInformation.loginUserFromKeytab`, and
`hadoop.security.authentication=kerberos` config from your code.

### 8.2 Ranger Policies → Fabric Roles + Lakehouse RLS/CLS
Ranger's fine-grained Hive policies have **no direct equivalent**. Plan a redesign:

| Ranger Policy Pattern | Fabric Mapping |
|-----------------------|---------------|
| Database-level SELECT for group | **Workspace** Viewer role (whole workspace) OR per-Lakehouse share |
| Table-level SELECT with column mask | **SQL endpoint Column-Level Security** (`GRANT SELECT ON TABLE ...(col1, col2)`) |
| Row filter `region = '${user.region}'` | **SQL endpoint Row-Level Security** (CREATE SECURITY POLICY) |
| Deny by tag | Manual policy in SQL endpoint — **no tag inheritance** in Fabric yet |
| Audit logs | Fabric Activity Log + Purview integration |

```sql
-- Fabric Lakehouse SQL endpoint RLS example
CREATE FUNCTION dbo.fn_orders_filter(@region NVARCHAR(50))
    RETURNS TABLE WITH SCHEMABINDING
AS RETURN
    SELECT 1 AS fn_result
    WHERE @region = USER_NAME() OR IS_ROLEMEMBER('SalesAdmin') = 1;

CREATE SECURITY POLICY orders_rls
    ADD FILTER PREDICATE dbo.fn_orders_filter(region) ON dbo.orders
    WITH (STATE = ON);
```

### 8.3 Storage Auth: Account Key / SAS → Workspace Identity
HDInsight clusters often use **storage account keys** in `core-site.xml`. Fabric uses
**Workspace Identity** (system-assigned MI) automatically. To grant Fabric access to
external storage (for shortcuts or pipeline copies):

```bash
# Get Fabric workspace identity object ID
WS_OID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['workspaceIdentity']['servicePrincipalId'])")

# Grant Storage Blob Data Contributor on the legacy ADLS account
az role assignment create \
  --assignee "$WS_OID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$SUB/resourceGroups/legacy-rg/providers/Microsoft.Storage/storageAccounts/legacystorage"
```

---

## 9. Cutover Strategy

### 9.1 Strangler-Fig Pattern (Recommended)

```
Phase 0: Discover & assess (1-2 weeks)
Phase 1: Stand up Fabric workspace, capacity, lakehouses (1 week)
Phase 2: Create shortcuts to existing ADLS (no data move) — read-only validation (1 week)
Phase 3: Migrate ONE pilot job — validate output equality vs HDI (1-2 weeks)
Phase 4: Migrate jobs in waves, run dual-write, compare daily (4-12 weeks)
Phase 5: Switch readers (Power BI, downstream) to Fabric tables
Phase 6: Stop HDI writers; copy remaining data; delete HDI cluster
Phase 7: Decommission ADLS containers (or convert to OneLake managed tables)
```

### 9.2 Dual-Write Validation Pattern
```python
# Run BOTH old (HDI) and new (Fabric) jobs for each daily batch
# Compare output deterministically using a checksum table

def validate_batch(date_str: str, hdi_path: str, fab_table: str):
    hdi = spark.read.parquet(hdi_path).filter(f"order_date = '{date_str}'")
    fab = spark.table(fab_table).filter(f"order_date = '{date_str}'")

    # 1) Row count
    h_n, f_n = hdi.count(), fab.count()
    assert h_n == f_n, f"Row mismatch on {date_str}: HDI={h_n}, Fabric={f_n}"

    # 2) Column-set / dtype match
    assert sorted(hdi.dtypes) == sorted(fab.dtypes), \
        f"Schema mismatch on {date_str}"

    # 3) Hash-based content comparison (order-insensitive)
    from pyspark.sql.functions import md5, concat_ws, sum as fsum, hash as fhash
    cols = [c for c, _ in sorted(hdi.dtypes)]
    h_h = hdi.select(fhash(concat_ws("|", *cols)).alias("h")) \
              .agg(fsum("h").alias("s")).collect()[0]["s"]
    f_h = fab.select(fhash(concat_ws("|", *cols)).alias("h")) \
              .agg(fsum("h").alias("s")).collect()[0]["s"]
    assert h_h == f_h, f"Content hash mismatch on {date_str}: {h_h} vs {f_h}"
    return {"date": date_str, "rows": h_n, "hash": h_h, "ok": True}

# Drive from Pipeline → run for 30 days, fail loud on mismatch
```

### 9.3 Cutover Day Checklist
- [ ] All upstream writers paused (or in dual-write for ≥7 days with zero diffs)
- [ ] Final delta-copy from ADLS → OneLake (use `azcopy sync` or pipeline)
- [ ] All Power BI semantic models pointed at Fabric tables (Direct Lake)
- [ ] All scheduled pipelines/notebooks have triggers enabled in Fabric
- [ ] All HDI Oozie coordinators paused
- [ ] Monitoring dashboards (Capacity Metrics, Pipeline run history) green for 24h
- [ ] HDI cluster scaled to 0 (or deleted) — but **keep ADLS data for 30 days** as rollback
- [ ] Alerting / on-call rotations updated to Fabric resource IDs

---

## 10. Cost Re-Modeling

| HDInsight Cost Driver | Fabric Equivalent | Optimization Tip |
|-----------------------|-------------------|------------------|
| Always-on head nodes (D4 × 2 × 24 × 30 = ~$580/mo) | None — capacity covers all overhead | Net win for bursty workloads |
| Worker nodes (D4 × 4 × 24 × 30 = ~$1160/mo) | Spark CU-hours (1 vCore ≈ 0.5 CU/h) | Use `availableNow` triggers; pause capacity off-hours |
| ADLS Gen2 storage | OneLake storage (same $/GB) | Schedule `VACUUM` to clean Delta history |
| ADLS transactions | OneLake transactions (same) | Compact small files via `OPTIMIZE` |
| AAD DS (~$110/mo) | None — Entra ID is free | Pure savings |
| Premium support | Fabric capacity includes support | Bundled |

**Sizing rule of thumb**:
- HDI Spark cluster of 4× D4 workers (~16 vCores, ~64 GB) ≈ **F32 capacity** for similar
  burst capacity, **F16** if jobs are short-lived (<10 min) and infrequent.
- Use the **Fabric Capacity Metrics** app for the first 30 days; right-size up or down.
- **Pause** F SKU capacity outside business hours for dev/test (CLI: `POST /capacities/{id}/suspend`).

---

## 11. Validation & Acceptance

Define explicit pass criteria **before** cutover. Sample acceptance template:

```yaml
migration_pass_criteria:
  data_correctness:
    - dual_write_diff_count: 0          # for ≥7 consecutive days
    - row_count_delta_pct: < 0.01
    - schema_match: exact
  performance:
    - p50_runtime_delta_pct: < 20      # Fabric ≤ 1.2× HDI baseline
    - p95_runtime_delta_pct: < 50
  cost:
    - monthly_cu_burn_pct: < 70        # leave headroom
  reliability:
    - pipeline_success_rate_7d: > 99.0
    - no_throttling_events_7d: true    # capacity not exceeded
  security:
    - all_workspaces_have_identity: true
    - rls_policies_validated: true     # SOC2 sign-off if applicable
```

---

## 12. Pitfalls & Lessons Learned

1. **`enableHiveSupport()` left in code → silent breakage**. Fabric ignores it but
   downstream `USE database` calls may resolve differently. Remove explicitly.

2. **Default Lakehouse trap**. Notebooks have a single "default" Lakehouse — `spark.table("orders")`
   resolves there. If you're reading across Lakehouses, **always 3-part name** them
   (`workspace.lakehouse.table`) or use full ABFS paths.

3. **V-Order on copied data**. If you `azcopy` raw Parquet from HDI into OneLake `Files/`,
   it's NOT V-Order. Reading is fine; **Direct Lake will fall back to DirectQuery**.
   Fix: re-write through Spark with `df.write.format("delta").saveAsTable(...)`.

4. **Liquid Clustering vs partitioned tables**. HDI tables partitioned by high-cardinality
   columns (e.g., `customer_id`) create thousands of tiny files. Migrating "as-is" preserves
   the small-file problem. Drop the partition, replace with `CLUSTER BY (customer_id)`.

5. **Hive ACID delta files**. If your HDI table is transactional, `azcopy` of the raw
   directory copies `delta_*` and `base_*` subfolders that Spark can't read directly.
   You **must** read via Hive (Beeline/HiveContext) on HDI, then write to Fabric.

6. **Custom Scala JARs from HDI 4.0**. Built against Spark 2.4 + Scala 2.11. Won't run
   on RT 1.3 (Spark 3.5 + Scala 2.12) without recompile **and** API audits — many Spark
   2.x → 3.x breaking changes (e.g., `org.apache.spark.sql.execution.datasources.csv` package moves).

7. **Livy session-based code → no equivalent**. If you used Livy `sessions` (interactive)
   from a custom UI, Fabric has no equivalent — switch to **notebook embedding** or
   **Spark Connect** (RT 2.0).

8. **Oozie EL functions**. `${coord:dataIn(...)}`, `${coord:formatTime(...)}` etc. have
   no Fabric equivalent. Pipeline expressions use `@formatDateTime(pipeline().parameters.date, 'yyyy-MM-dd')`
   — different syntax, semantically similar.

9. **Pipeline schedule triggers are UTC-only**. If your Oozie coordinators were in
   `America/Chicago`, you'll need to manually offset and handle DST. As of early 2026,
   Fabric pipelines have no timezone-aware cron.

10. **VACUUM is not automatic**. Old HDI Hive ACID had `MAJOR COMPACT`. Fabric needs
    explicit `OPTIMIZE` + `VACUUM` (default 7-day retention). Add a maintenance pipeline.

11. **HBase / Phoenix have no Fabric equivalent**. Don't try to "migrate" — re-platform
    to Cosmos DB (for OLTP-style key access) or KQL DB (for time-series).

12. **High-Concurrency Mode confusion**. Fabric's HC mode shares a SparkContext across
    notebooks but isolates SparkSessions. Code that depends on `sc.broadcast(...)` being
    visible across notebooks **breaks** — use Lakehouse tables for shared state instead.

13. **No SSH, no `/etc/spark/conf`**. All cluster-level config you used to set in Ambari
    (e.g., `spark.executor.extraJavaOptions=-XX:+UseG1GC`) must move to **Environment
    → Spark properties** (workspace level) or per-session `spark.conf.set(...)`.

14. **Network isolation**. HDI in a VNet with NSG locks down public ingress. Fabric
    Private Link / Managed VNet is newer (some features still preview as of early 2026).
    If you need true VNet injection, **Databricks** is the safer landing zone today.

15. **Test data PII**. Migration is a great opportunity to introduce **column masking**
    via SQL endpoint CLS — but it's a **breaking change** for downstream consumers if
    they previously had unrestricted access. Coordinate with security + consumers.

---

## 13. Quick-Reference Migration Checklist

```
[ ] Inventory all HDI clusters, jobs, tables, schedules, custom libs
[ ] Categorize jobs by complexity (Low / Medium / High / Blocker)
[ ] Decide per-workload target (Fabric / Databricks / re-platform)
[ ] Provision Fabric workspace + capacity (start with F32 trial)
[ ] Create destination Lakehouse(s) — one per data domain
[ ] Grant Workspace Identity access to legacy ADLS storage
[ ] Translate + apply Hive DDL → Delta tables
[ ] Create shortcuts for read-only validation
[ ] Rewrite Spark job paths (wasbs:// → abfss://...onelake)
[ ] Recompile Scala JARs (2.11/2.12 → 2.13 if RT 2.0)
[ ] Upload dependencies to Fabric Environment, publish
[ ] Convert Livy/Oozie orchestration → Pipelines + Notebooks/SJDs
[ ] Map Ranger policies → Workspace roles + SQL RLS/CLS
[ ] Run pilot job + dual-write validation for ≥7 days
[ ] Migrate remaining jobs in waves (small first)
[ ] Switch Power BI semantic models to Direct Lake
[ ] Pause HDI cluster; copy final delta; verify; delete
[ ] Schedule weekly OPTIMIZE + VACUUM maintenance pipeline
[ ] Set up Capacity Metrics monitoring + alerts
[ ] Document runbook + on-call procedures for the new platform
```

---

## 14. Useful Commands & API Snippets

```bash
# Get Entra ID token for Fabric API
az account get-access-token \
  --resource https://api.fabric.microsoft.com \
  --query accessToken -o tsv

# Get token for OneLake (storage data plane)
az account get-access-token \
  --resource https://storage.azure.com \
  --query accessToken -o tsv

# List all items in a Fabric workspace
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/items" | python3 -m json.tool

# Trigger a pipeline run with parameters
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/items/$PIPELINE_ID/jobs/instances?jobType=Pipeline" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"executionData":{"parameters":{"loadDate":"2024-01-15"}}}'

# Bulk OPTIMIZE all tables in a Lakehouse (run as a notebook)
spark.sql("SHOW TABLES IN sales_lakehouse").collect()
for row in spark.sql("SHOW TABLES IN sales_lakehouse").collect():
    spark.sql(f"OPTIMIZE sales_lakehouse.{row['tableName']}")
    spark.sql(f"VACUUM   sales_lakehouse.{row['tableName']} RETAIN 168 HOURS")
```

---

## Rules

1. **Never lift-and-shift blindly** — assess and re-architect storage layout (partitioning, clustering, V-Order).
2. **Always dual-write before cutover** — minimum 7 days, hash-based content validation.
3. **Land on Runtime 1.3 first** (GA) — schedule Runtime 2.0 upgrade as a separate later project.
4. **Use shortcuts for the cutover window** — defer data movement until job correctness is proven.
5. **Recompile JARs against the target Spark/Scala version** — never assume binary compatibility.
6. **Replace `enableHiveSupport()` and any HMS-coupled config** — Fabric Lakehouse is the catalog.
7. **Schedule OPTIMIZE + VACUUM** — Fabric does not auto-maintain Delta tables.
8. **Map Ranger → SQL RLS/CLS deliberately** — coarser model; communicate with consumers.
9. **Pause Fabric capacity outside business hours** for dev/test — major cost saving vs HDI.
10. **Keep HDI cluster + ADLS for 30 days post-cutover** — instant rollback path.
