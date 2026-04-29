---
name: azure-hdinsight-migration-interactive-query-to-fabric-spark-or-dw
description: End-to-end migration playbook for moving Azure HDInsight Interactive Query (Hive LLAP, HiveServer2, Tez) workloads to Microsoft Fabric — choosing between Fabric Spark (Lakehouse + SQL endpoint) and Fabric Data Warehouse, HiveQL → Spark SQL / T-SQL conversion, ACID/transactional table migration, ORC → Delta conversion, Beeline/JDBC client cutover, Ranger → SQL RLS/CLS, LLAP cache → Direct Lake / Result-Set Cache, validation.
license: MIT
version: 1.0.0
updated: 2026-04-29
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
triggers: hdinsight interactive query, hive llap, hive migration, hiveserver2, beeline, hiveql, tez, llap, hive acid, orc to delta, fabric warehouse, fabric dw, sql endpoint, ranger hive, hive metastore, hdi iq
requires: azure-hdinsight, azure-fabric
---
# Migrating HDInsight Interactive Query (Hive LLAP) → Microsoft Fabric

A field guide for migrating production **HDInsight Interactive Query** clusters
(Hive LLAP, HiveServer2, Tez execution, ORC ACID tables) to **Microsoft Fabric**.

Two viable landing zones in Fabric, often combined:
- **Fabric Lakehouse + SQL Analytics Endpoint** — Delta tables in OneLake; read-only T-SQL via auto-generated endpoint; preferred when the workload is part of a Spark/data-engineering pipeline.
- **Fabric Data Warehouse** — Fully writable T-SQL warehouse, also Delta-backed in OneLake, with multi-table transactions, stored procedures, and DDL via T-SQL.

> **Why this migration?** HDInsight 4.0 reached end-of-standard-support; 5.x is the
> last classic generation. LLAP's "always-on hot cache" cost model is misaligned with
> serverless Fabric. **Direct Lake** + **Result-Set Cache** + **V-Order Parquet** in
> Fabric collectively replace the LLAP role for sub-second analytical query patterns.

---

## 1. Decision Matrix — Lakehouse SQL Endpoint vs Fabric Warehouse

| Workload Pattern | Recommended Target | Reason |
|------------------|-------------------|--------|
| Reporting queries (BI tools) on existing Delta | **Lakehouse SQL endpoint** | Read-only, free with Lakehouse, Direct Lake-ready |
| Power BI dashboards needing fastest refresh | **Lakehouse SQL endpoint + Direct Lake** | V-Order auto-applied; sub-second visuals |
| ETL jobs that issue `INSERT/UPDATE/DELETE/MERGE` via HiveQL | **Fabric Warehouse** OR **Fabric Spark notebooks** | SQL endpoint is read-only |
| Multi-statement transactions (`BEGIN/COMMIT`) | **Fabric Warehouse** | Lakehouse has no T-SQL transactions |
| Stored procedures (Hive UDFs / Hive Procedural SQL) | **Fabric Warehouse** (T-SQL stored procs) | Lakehouse SQL endpoint has no procs |
| Cross-database joins within one workspace | **Either** — both expose 3-part naming | Warehouse is faster for joining warehouse-resident tables |
| Heavy ACID `MERGE` workload | **Fabric Spark** (Delta MERGE) OR **Warehouse** (T-SQL MERGE) | Both work; Spark scales better for large MERGEs |
| External tables over raw files (CSV/Parquet) | **Lakehouse Files + SQL endpoint** | `OPENROWSET` in Warehouse also works |
| LLAP daemon cache for "hot" tables | **Direct Lake (Power BI)** + **Result-Set Cache (Warehouse)** | No equivalent always-on cache |
| Hive UDFs (Java JARs) | **Spark UDFs (Python/Scala)** in notebooks | Warehouse has no UDFs (yet) |
| Hive views with complex CTEs | **Lakehouse SQL endpoint VIEW** OR **Warehouse VIEW** | Both support T-SQL views |
| Workloads needing per-column masking | **Either** — both support T-SQL CLS/RLS | Choose by write/read needs |

**Decision shortcut**:
- **Mostly read?** → **Lakehouse SQL endpoint** (free, Direct Lake, fast).
- **Need writes via SQL?** → **Fabric Warehouse**.
- **Mixed: Spark writes + SQL reads?** → **Lakehouse for writes, SQL endpoint for reads** (single Delta storage; both engines see the same tables).

---

## 2. Assessment Phase

### 2.1 Inventory the HDInsight Interactive Query Cluster

Run on a head node (SSH).

```bash
ssh sshuser@<cluster>-ssh.azurehdinsight.net

# Cluster + service version metadata
curl -s -u admin:$PASS \
  "https://<cluster>.azurehdinsight.net/api/v1/clusters/<cluster>" \
  | python3 -m json.tool > cluster_meta.json

# Hive / Tez / LLAP versions
hive --version 2>&1 | tee hive_version.txt
hdp-select status hive-server2-hive2 hive2_llap | tee hive_versions.txt

# LLAP daemon status + cache hit rate
sudo -u hive curl -s "http://$(hostname):15002/status" | python3 -m json.tool > llap_status.json
sudo -u hive curl -s "http://$(hostname):15002/jmx" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    [print(b['name'],':',b.get('CacheHitBytes','?'),'/',b.get('CacheRequestedBytes','?')) \
     for b in d['beans'] if 'LlapDaemon' in b.get('name','')]" > llap_cache.txt

# Beeline shortcut for inventory queries
BEELINE="beeline -u jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice -n admin -p $PASS --silent=true --outputformat=tsv2"

# All databases + tables
$BEELINE -e "SHOW DATABASES" | tail -n +2 > databases.txt

> table_inventory.tsv
echo -e "db\ttable\towner\ttype\tinput_format\tnum_rows\tsize_bytes\ttransactional\tlocation" >> table_inventory.tsv
while read db; do
  $BEELINE -e "USE $db; SHOW TABLES" | tail -n +2 | while read tb; do
    {
      # DESCRIBE FORMATTED returns key:value rows we can grep
      desc=$($BEELINE -e "DESCRIBE FORMATTED $db.$tb" 2>/dev/null)
      owner=$(echo "$desc"      | awk -F'\t' '/^Owner:/{print $2; exit}')
      ttype=$(echo "$desc"      | awk -F'\t' '/^Table Type:/{print $2; exit}')
      ifmt=$(echo "$desc"       | awk -F'\t' '/InputFormat:/{print $2; exit}')
      nrow=$(echo "$desc"       | awk -F'\t' '/numRows/{print $3; exit}')
      sb=$(echo "$desc"         | awk -F'\t' '/totalSize|rawDataSize/{print $3; exit}')
      txn=$(echo "$desc"        | awk -F'\t' '/transactional/{print $3; exit}')
      loc=$(echo "$desc"        | awk -F'\t' '/^Location:/{print $2; exit}')
      echo -e "$db\t$tb\t$owner\t$ttype\t$ifmt\t$nrow\t$sb\t$txn\t$loc"
    } >> table_inventory.tsv
  done
done < databases.txt

# Views (and their definitions — useful for translation)
> views_inventory.txt
while read db; do
  $BEELINE -e "USE $db; SHOW VIEWS" 2>/dev/null | tail -n +2 | while read v; do
    echo "=== $db.$v ===" >> views_inventory.txt
    $BEELINE -e "SHOW CREATE TABLE $db.$v" >> views_inventory.txt
  done
done < databases.txt

# Hive UDFs registered in the metastore
$BEELINE -e "SHOW FUNCTIONS" > all_functions.txt
# Filter to user-defined (non-built-in) — anything containing a database prefix is custom
grep -E '\.' all_functions.txt > custom_udfs.txt

# HiveQL workload — pull recent queries from Hive query log (if enabled)
# Default location: /tmp/hive/operation_logs/ or HDFS /tmp/hive/_resultscache_/
# Or query Tez UI / Hive Metastore audit DB if configured
sudo find /tmp/hive -name "*.log" -mtime -7 2>/dev/null | head -100 \
  | xargs grep -hE 'COMPILE|EXEC' 2>/dev/null > recent_queries.log

# LLAP I/O cache contents — what's "hot" today (drives Power BI / Direct Lake sizing)
sudo -u hive hive --service llapdump --output llap_cache_dump.json 2>/dev/null
```

### 2.2 Inventory the Client / BI Codebase

```bash
cd /path/to/repos

# 1) JDBC connection strings to Hive
grep -RInE 'jdbc:hive2://|HiveServer2|hiveserver2' \
  --include='*.py' --include='*.java' --include='*.scala' \
  --include='*.cs' --include='*.r' --include='*.properties' \
  --include='*.yaml' --include='*.yml' --include='*.json' --include='*.xml'

# 2) Beeline / Hive CLI scripts
find . -name 'run_*.sh' -o -name '*.hql' -o -name '*.sql' \
  | xargs grep -lE 'beeline|hive -e|hive -f' 2>/dev/null

# 3) BI tool / ODBC DSNs (often live in odbc.ini, registry export, BI tool repo)
find . -name 'odbc.ini' -o -name 'tnsnames.ora' -o -name '*.dsn'
grep -RIn "Hortonworks Hive|Microsoft Hive ODBC|Cloudera Hive" \
  --include='*.ini' --include='*.dsn' --include='*.txt'

# 4) Hive UDF JAR usage in HQL
grep -RIn "ADD JAR\|CREATE TEMPORARY FUNCTION\|CREATE FUNCTION" \
  --include='*.hql' --include='*.sql'

# 5) HQL constructs that need attention during migration
grep -RInE 'LATERAL VIEW|EXPLODE|TABLESAMPLE|DISTRIBUTE BY|CLUSTER BY|SORT BY|TRANSFORM\(|MAP\(|REDUCE\(|MAPJOIN\(' \
  --include='*.hql' --include='*.sql'

# 6) DML / ACID usage (signals you need Warehouse, not Lakehouse SQL endpoint)
grep -RInE '^\s*(INSERT|UPDATE|DELETE|MERGE)\s' \
  --include='*.hql' --include='*.sql' \
  | grep -vE 'INSERT INTO.*SELECT' | head -100   # quick "are there mutations?" check
```

### 2.3 Categorize Each Object/Workload

Build a CSV: `db, object, type, hql_features_used, dml_kind, target_engine, complexity, notes`.

| Complexity | Definition | Migration Effort |
|------------|------------|------------------|
| **Low**    | EXTERNAL ORC/Parquet table, basic SELECT, no UDF, no ACID | hours |
| **Medium** | Partitioned tables, complex CTEs, views, `LATERAL VIEW EXPLODE`, ACID INSERT | days |
| **High**   | Hive ACID `MERGE`/`UPDATE`/`DELETE`, custom Java UDFs, `TRANSFORM USING <script>`, bucketed joins | 1-3 weeks |
| **Blocker**| `MAP REDUCE` blocks (streaming), legacy SerDes, cross-cluster Hive replication | re-architect |

---

## 3. Component Mapping

### 3.1 Engine & Catalog

| HDInsight Interactive Query | Fabric Lakehouse SQL Endpoint | Fabric Warehouse |
|----------------------------|------------------------------|------------------|
| HiveServer2 (port 10001 HTTP) | T-SQL endpoint URL (workspace.fabric.microsoft.com,1433) | T-SQL endpoint URL (different host, same port 1433) |
| Hive Metastore (Azure SQL DB) | Lakehouse catalog (auto, per Lakehouse) | Warehouse catalog (per Warehouse) |
| LLAP daemons (always-on cache) | None — replaced by V-Order + Direct Lake | **Result-set cache** (per warehouse, automatic) |
| Tez execution engine | Spark (Lakehouse) | Fabric Warehouse distributed query processing engine (T-SQL surface) |
| YARN scheduling | Capacity Units (CU-based) | Capacity Units (CU-based) |
| ORC ACID tables | Delta tables (Spark writes) | Delta tables (T-SQL writes via Fabric Warehouse engine) |
| Hive views | Lakehouse SQL endpoint VIEW (read-only) | Warehouse VIEW (full T-SQL) |
| Hive UDFs (Java) | Spark UDFs (Python/Scala) in notebooks | Not supported (rewrite as views or T-SQL functions) |
| Hive ACID `MERGE` | Delta `MERGE` via Spark | T-SQL `MERGE` |
| Beeline / JDBC clients | sqlcmd, pyodbc, JDBC (TDS, port 1433) | Same — SQL Server-style TDS |

### 3.2 Storage

| HDI Hive Storage | Fabric Equivalent |
|------------------|-------------------|
| ORC files in `wasbs://.../warehouse/db.db/table/` | Delta in `abfss://workspace@onelake.dfs.fabric.microsoft.com/<lh>.Lakehouse/Tables/<db>/<table>/` |
| ORC ACID with `delta_*` / `base_*` subdirs | Delta `_delta_log/` JSON commit log + parquet |
| Bucketed Hive tables | Liquid Clustering (Delta) — NOT Hive bucketing |
| Partitioned (Hive-style `key=value`) | Delta partitioning OR Liquid Clustering (preferred) |
| External tables over wasbs/adls | Fabric Lakehouse **shortcut** to ADLS — read-only via SQL endpoint |

### 3.3 Auth & Security

| HDI Hive | Fabric SQL endpoint / Warehouse |
|----------|--------------------------------|
| LDAP (Ambari managed) | Entra ID |
| Kerberos / ESP (`GSSAPI`) | Entra ID (Azure AD authentication for SQL) |
| Ranger Hive policies (db/table/column ACL) | T-SQL `GRANT/DENY` + RLS predicates + Dynamic Data Masking |
| Ranger row filter `region = '${user.region}'` | T-SQL RLS via `CREATE SECURITY POLICY` |
| Ranger column mask | T-SQL Dynamic Data Masking OR explicit view-level redaction |
| Ranger audit logs | Fabric Activity Log + Purview integration |

---

## 4. Choosing Lakehouse SQL Endpoint vs Fabric Warehouse — Per-Object

Apply this filter to each Hive object:

```
For each Hive table/view:
  If only READ access is needed downstream → Lakehouse SQL endpoint
  Else if has stored procs / multi-statement TX / heavy DML → Fabric Warehouse
  Else if Spark already handles writes → Lakehouse (writes via Spark, reads via SQL endpoint)
  Else (mixed-write SQL workload) → Fabric Warehouse

For each Hive ACID write workload (MERGE/UPDATE/DELETE):
  If batch ETL with PySpark talent on team → Spark notebook with Delta MERGE → Lakehouse
  Else (T-SQL skill set, BI dev team) → Fabric Warehouse with T-SQL MERGE
```

**Hybrid pattern that works well**: Lakehouse for raw + silver layers (Spark writes,
big data volumes), Warehouse for gold marts (T-SQL writes, BI semantic-layer-friendly).

---

## 5. Storage & Schema Migration

### 5.1 Pattern A — Shortcut (Fastest Cutover)

Create a Fabric **shortcut** from the destination Lakehouse to the existing ADLS path
holding ORC files. **No data movement.** Tables become readable via SQL endpoint
immediately. Cut readers over, then convert to Delta in the background.

```python
import requests
payload = {
    "name": "legacy_orders",
    "path": "Tables",
    "target": {
        "type": "AdlsGen2",
        "adlsGen2": {
            "location": "https://hdistore.dfs.core.windows.net",
            "subpath": "/warehouse/sales/orders/",
            "connectionId": "<conn-id>"
        }
    }
}
requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{WS_ID}/lakehouses/{LH_ID}/shortcuts",
    headers={"Authorization": f"Bearer {token}", "Content-Type":"application/json"},
    json=payload,
)
```

**Critical limitations of shortcut path for Hive**:
- **ACID ORC tables (delta files / base files) cannot be read directly** by Spark or
  the SQL endpoint. You must read via Hive on HDI and re-write to Delta.
- **External ORC tables** read fine, but **Direct Lake will fall back to DirectQuery**
  (no V-Order, not Delta).
- **Bucketed Hive tables** lose their bucketing semantics — joins won't be co-located.

### 5.2 Pattern B — Convert ORC → Delta (Recommended for Long Term)

```python
# Run in a Fabric notebook attached to the destination Lakehouse
src = "abfss://container@hdistore.dfs.core.windows.net/warehouse/sales.db/orders/"

# For non-ACID ORC: read directly
df = spark.read.format("orc").load(src)

# For ACID ORC: must read via Hive (run on HDI side)
#   beeline -e "INSERT OVERWRITE DIRECTORY 'wasbs://stage/orders' \
#       STORED AS PARQUET \
#       SELECT * FROM sales.orders;"
# then on Fabric:
# df = spark.read.format("parquet").load(
#     "abfss://stage@hdistore.dfs.core.windows.net/orders/")

# Write as managed Delta in Fabric (V-Order on by default)
(df.write.format("delta")
   .mode("overwrite")
   .option("overwriteSchema", "true")
   .saveAsTable("sales_lakehouse.orders"))

# Apply table properties used by Hive ACID equivalents
spark.sql("""
    ALTER TABLE sales_lakehouse.orders SET TBLPROPERTIES (
      'delta.autoOptimize.optimizeWrite' = 'true',
      'delta.autoOptimize.autoCompact'   = 'true'
    )
""")
```

### 5.3 Pattern C — Pipeline Copy (No-Code Bulk Load)

Use a Fabric Pipeline → Copy Activity → ADLS Gen2 source → Lakehouse Table sink. Best
for hundreds of tables; supports parallelism, monitoring, and retry. Drives schema
inference automatically.

### 5.4 Hive DDL → Delta DDL Translation

```sql
-- HDInsight Hive (source)
CREATE TABLE sales.orders (
    order_id     BIGINT,
    customer_id  BIGINT,
    amount       DECIMAL(18,2),
    order_date   DATE
)
PARTITIONED BY (load_date STRING)
CLUSTERED BY (customer_id) INTO 16 BUCKETS
STORED AS ORC
LOCATION 'wasbs://datalake@hdistore.blob.core.windows.net/warehouse/sales.db/orders'
TBLPROPERTIES ("transactional" = "true");

-- Fabric Lakehouse / Spark target
CREATE TABLE sales_lakehouse.orders (
    order_id    BIGINT,
    customer_id BIGINT,
    amount      DECIMAL(18,2),
    order_date  DATE,
    load_date   STRING
)
USING DELTA
CLUSTER BY (customer_id, order_date)
TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact'   = 'true'
);

-- Fabric Warehouse target
CREATE TABLE dbo.orders (
    order_id    BIGINT NOT NULL,
    customer_id BIGINT,
    amount      DECIMAL(18,2),
    order_date  DATE,
    load_date   VARCHAR(8)
);
-- Warehouse picks distribution + storage automatically; no DISTRIBUTION/INDEX clauses.
```

### 5.5 Bulk DDL Translation Helper

```python
# Run in a Fabric notebook to bulk-translate exported Hive DDL
import re, pathlib

DDL_DIR = pathlib.Path("/lakehouse/default/Files/ddl_export")
OUT_DIR = pathlib.Path("/lakehouse/default/Files/ddl_fabric")
OUT_DIR.mkdir(exist_ok=True)
LH = "sales_lakehouse"

def translate(ddl: str, lh: str) -> str:
    ddl = re.sub(r"STORED\s+AS\s+(ORC|PARQUET|TEXTFILE|AVRO)", "USING DELTA", ddl, flags=re.I)
    ddl = re.sub(r"LOCATION\s+'[^']*'", "", ddl, flags=re.I)
    ddl = re.sub(r"CLUSTERED\s+BY\s*\([^)]*\)\s*INTO\s+\d+\s+BUCKETS", "", ddl, flags=re.I)
    ddl = re.sub(r'TBLPROPERTIES\s*\([^)]*"transactional"[^)]*\)', "", ddl, flags=re.I)
    ddl = re.sub(r"ROW\s+FORMAT\s+SERDE\s+'[^']*'", "", ddl, flags=re.I)
    ddl = re.sub(r"WITH\s+SERDEPROPERTIES\s*\([^)]*\)", "", ddl, flags=re.I)
    ddl = re.sub(r"CREATE\s+(?:EXTERNAL\s+)?TABLE\s+(IF\s+NOT\s+EXISTS\s+)?(\w+)\.(\w+)",
                 lambda m: f"CREATE TABLE {m.group(1) or ''}{lh}.{m.group(3)}", ddl, flags=re.I)
    return ddl.strip().rstrip(";") + (
        "\nTBLPROPERTIES ('delta.autoOptimize.optimizeWrite'='true');"
    )

for f in DDL_DIR.glob("*.sql"):
    (OUT_DIR / f.name).write_text(translate(f.read_text(), LH))

# Apply
for f in sorted(OUT_DIR.glob("*.sql")):
    try:
        spark.sql(f.read_text()); print("✓", f.name)
    except Exception as e:
        print("✗", f.name, "→", str(e)[:120])
```

---

## 6. HiveQL → Spark SQL / T-SQL Conversion

### 6.1 Common HiveQL Constructs and Their Targets

| HiveQL                                              | Spark SQL (Lakehouse)                | T-SQL (Warehouse)                                |
|-----------------------------------------------------|--------------------------------------|--------------------------------------------------|
| `SELECT col, COUNT(*) FROM t GROUP BY col`          | identical                            | identical                                        |
| `LATERAL VIEW EXPLODE(arr) e AS x`                  | `LATERAL VIEW EXPLODE(arr) e AS x` (same) | use OPENJSON / cross apply STRING_SPLIT / pre-explode in Spark |
| `LATERAL VIEW POSEXPLODE(arr) e AS pos, x`          | identical                            | `CROSS APPLY (SELECT ...)` after JSON shred      |
| `INSERT OVERWRITE TABLE t PARTITION (d='x') ...`    | `INSERT OVERWRITE TABLE t PARTITION (d='x') ...` (or use `replaceWhere`) | `DELETE FROM t WHERE d='x';` then `INSERT INTO t ...`  |
| `MERGE INTO t USING s ON ... WHEN MATCHED ...`      | identical (Delta MERGE — superset)   | `MERGE INTO t USING s ON ...` (T-SQL MERGE)      |
| `CLUSTER BY k` (post-shuffle ordering)              | `CLUSTER BY k`                       | not applicable                                   |
| `DISTRIBUTE BY k SORT BY k`                         | `DISTRIBUTE BY k SORT BY k`          | not applicable                                   |
| `TRANSFORM (cols) USING 'python script.py' AS ...`  | rewrite as PySpark `mapInPandas` or UDF | not supported                                  |
| `MAPJOIN(small)` hint                               | `/*+ BROADCAST(small) */`            | `OPTION (HASH JOIN)` or just let optimizer       |
| Hive UDF `my_udf(col)` (Java JAR)                   | Spark UDF (Py/Scala) registered      | Rewrite as T-SQL function or pre-compute in Spark |
| `STRING(col)`                                       | `CAST(col AS STRING)`                | `CAST(col AS VARCHAR(MAX))`                      |
| `unix_timestamp(s, fmt)`                            | `unix_timestamp(s, fmt)`             | `DATEDIFF(SECOND, '1970-01-01', TRY_PARSE(s AS DATETIME2))` |
| `from_unixtime(ts)`                                 | `from_unixtime(ts)`                  | `DATEADD(SECOND, ts, '1970-01-01')`              |
| `regexp_replace(s, p, r)`                           | identical                            | not built-in — use CLR or pre-process in Spark   |
| `get_json_object(s, '$.a.b')`                       | identical                            | `JSON_VALUE(s, '$.a.b')`                         |
| `collect_list(col)` / `collect_set(col)`            | identical                            | `STRING_AGG(col, ',')` for list; no native set   |
| `percentile_approx(col, 0.95)`                      | `percentile_approx(col, 0.95)`       | `APPROX_PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY col)` |
| `IF (cond, a, b)`                                   | `IF (cond, a, b)`                    | `IIF(cond, a, b)`                                |
| `NVL(a, b)`                                         | `NVL(a, b)` / `COALESCE`             | `ISNULL(a, b)` / `COALESCE`                      |
| `SORT BY` (per-reducer sort)                        | `SORT BY`                            | use `ORDER BY` (global)                          |
| `TABLESAMPLE (10 PERCENT)`                          | `TABLESAMPLE (10 PERCENT)`           | `TABLESAMPLE (10 PERCENT)` (slightly different semantics) |

### 6.2 Side-by-Side Examples

#### MERGE / Upsert
```sql
-- HDInsight Hive ACID
MERGE INTO transactions AS t
USING transactions_staging AS s
ON  (t.txn_id = s.txn_id)
WHEN MATCHED     THEN UPDATE SET amount = s.amount, txn_date = s.txn_date
WHEN NOT MATCHED THEN INSERT VALUES (s.txn_id, s.account_id, s.amount, s.txn_date);

-- Fabric Spark (Delta MERGE — same syntax + INSERT *)
MERGE INTO sales_lakehouse.transactions AS t
USING sales_lakehouse.transactions_staging AS s
ON  t.txn_id = s.txn_id
WHEN MATCHED     THEN UPDATE SET amount = s.amount, txn_date = s.txn_date
WHEN NOT MATCHED THEN INSERT *;     -- Delta-specific shortcut

-- Fabric Warehouse (T-SQL MERGE — note required terminating ;)
MERGE INTO dbo.transactions AS t
USING dbo.transactions_staging AS s
ON  t.txn_id = s.txn_id
WHEN MATCHED     THEN UPDATE SET amount = s.amount, txn_date = s.txn_date
WHEN NOT MATCHED THEN INSERT (txn_id, account_id, amount, txn_date)
                       VALUES (s.txn_id, s.account_id, s.amount, s.txn_date);
```

#### LATERAL VIEW EXPLODE
```sql
-- HDInsight Hive
SELECT u.user_id, t.tag
FROM users u
LATERAL VIEW EXPLODE(u.tags) e AS tag;

-- Fabric Spark — identical (Spark inherited Hive's LATERAL VIEW)
SELECT u.user_id, e.tag
FROM users u
LATERAL VIEW EXPLODE(u.tags) e AS tag;

-- Fabric Warehouse — no array type; assume tags stored as JSON string
SELECT u.user_id, j.value AS tag
FROM dbo.users u
CROSS APPLY OPENJSON(u.tags_json) j;
```

#### Dynamic Partition Insert
```sql
-- HDInsight Hive
SET hive.exec.dynamic.partition.mode = nonstrict;
INSERT OVERWRITE TABLE sales PARTITION (sale_date)
SELECT id, amount, sale_date FROM sales_staging;

-- Fabric Spark (use replaceWhere or partition overwrite)
INSERT OVERWRITE TABLE sales_lakehouse.sales PARTITION (sale_date)
SELECT id, amount, sale_date FROM sales_lakehouse.sales_staging;
-- OR (Delta-native):
-- df.write.format("delta").mode("overwrite") \
--   .option("replaceWhere", "sale_date >= '2024-01-01'") \
--   .saveAsTable("sales_lakehouse.sales")

-- Fabric Warehouse (no partitions — DELETE + INSERT)
DELETE FROM dbo.sales WHERE sale_date IN (SELECT DISTINCT sale_date FROM dbo.sales_staging);
INSERT INTO dbo.sales (id, amount, sale_date)
SELECT id, amount, sale_date FROM dbo.sales_staging;
```

#### Hive UDF → Spark UDF
```sql
-- HDInsight Hive (custom Java UDF)
ADD JAR wasbs://jars@hdistore.blob.core.windows.net/myudfs.jar;
CREATE TEMPORARY FUNCTION normalize_phone AS 'com.example.NormalizePhoneUDF';

SELECT normalize_phone(phone) FROM customers;
```

```python
# Fabric Spark — register Python UDF
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType
import re

@udf(StringType())
def normalize_phone(p):
    if p is None: return None
    digits = re.sub(r'\D', '', p)
    if len(digits) == 11 and digits.startswith('1'): digits = digits[1:]
    return f"({digits[0:3]}) {digits[3:6]}-{digits[6:10]}" if len(digits) == 10 else p

spark.udf.register("normalize_phone", normalize_phone)
spark.sql("SELECT normalize_phone(phone) FROM sales_lakehouse.customers").show()
```

### 6.3 HQL Translation Tooling
- **sqlglot** (Python lib) — programmatic AST translation `hive` → `spark` or `hive`
  → `tsql`. Good for bulk passes; expect to hand-fix ~20% (UDFs, hints, custom
  serdes, ORDER-BY semantics).
- **Microsoft Fabric Migration Assistant for Data Warehouse** (note: it targets
  Synapse Dedicated / SQL Server sources, NOT Hive). For Hive source, sqlglot is the
  realistic option as of this writing — verify whether Microsoft has added a
  Hive-source path in the latest Fabric migration docs before relying on it.

```python
# Example: bulk translate .hql files with sqlglot
import sqlglot, pathlib
SRC = pathlib.Path("./hql"); OUT = pathlib.Path("./spark_sql"); OUT.mkdir(exist_ok=True)
for f in SRC.glob("*.hql"):
    try:
        translated = sqlglot.transpile(f.read_text(), read="hive", write="spark", pretty=True)[0]
        (OUT / (f.stem + ".sql")).write_text(translated)
    except Exception as e:
        print("FAIL", f.name, e)
```

---

## 7. Replacing LLAP Cache — Performance Strategy

LLAP's "always-hot" daemon cache has no direct equivalent. Replace with a layered
strategy:

| LLAP Role | Fabric Replacement |
|-----------|--------------------|
| Sub-second SELECT for BI tools | **Power BI Direct Lake** (V-Order Parquet, hot in semantic model cache) |
| Repeated identical SQL queries | **Fabric Warehouse Result-Set Cache** (automatic, per-warehouse) |
| Predicate pushdown on ORC indexes | **Delta data skipping** (column statistics in `_delta_log`) + **Liquid Clustering** |
| Bucketed join elimination | **Liquid Clustering** on join keys + AQE broadcast hints |
| In-memory hot column scan | **V-Order Parquet** (Microsoft codec optimized for vectorized scan) |
| Dynamic partition pruning | Native in Spark 3.x and Fabric Warehouse engine (automatic) |

**Performance migration checklist per table**:
1. Convert ORC → Delta with V-Order (default in Fabric Spark).
2. Replace bucketing with `CLUSTER BY (cols)` (Liquid Clustering).
3. Run `OPTIMIZE` once after backfill.
4. For star-schema dimensions: keep as-is in Lakehouse, mark for **broadcast** in joins.
5. For BI: build a Power BI semantic model with **Direct Lake** mode.
6. For repeated SQL: ensure queries are **deterministic** (no `current_timestamp()` in WHERE) so result-set cache hits.

---

## 8. Cutover Strategy

### 8.1 Strangler-Fig Pattern (Recommended)

```
Phase 0  : Inventory + categorize (1-2 weeks)
Phase 1  : Provision Fabric workspace, capacity, Lakehouse(s) and/or Warehouse(s) (1 week)
Phase 2  : Shortcut to existing ADLS for read-only validation (1 week)
Phase 3  : Convert pilot DDL + bulk-load tables; run dual-read validation (1-2 weeks)
Phase 4  : Translate HQL queries / scripts; cut BI tools to new SQL endpoint (2-4 weeks)
Phase 5  : Cut ETL writers (HQL → Spark or T-SQL) (4-8 weeks)
Phase 6  : Decommission HDI Hive cluster
Phase 7  : Final ADLS cleanup (or shortcut → managed Delta migration)
```

### 8.2 Dual-Read Validation Pattern

Run identical queries on both engines and diff results.

```python
# Fabric notebook — diff Hive (via pyhive) vs Lakehouse (via spark.sql)
from pyhive import hive

def hive_query(sql, host="<cluster>.azurehdinsight.net", user="admin", pwd=None):
    conn = hive.Connection(host=host, port=443, username=user, password=pwd,
                           auth="LDAP",
                           configuration={"hive.server2.transport.mode": "http",
                                          "hive.server2.thrift.http.path": "/hive2"})
    import pandas as pd
    return pd.read_sql(sql, conn)

def fabric_query(sql):
    return spark.sql(sql).toPandas()

queries = [
    "SELECT region, COUNT(*) c, SUM(amount) s FROM sales.orders WHERE order_date='2024-06-01' GROUP BY region ORDER BY region",
    # ... add representative business queries
]
for q in queries:
    h = hive_query(q.replace("sales.orders", "sales.orders"))           # Hive view
    f = fabric_query(q.replace("sales.orders", "sales_lakehouse.orders")) # Fabric view
    eq = h.equals(f)
    print("✓" if eq else "✗", q[:80])
    if not eq:
        print("  hive_rows:", len(h), "fabric_rows:", len(f))
        print(h.merge(f, how="outer", indicator=True)
                .query("_merge != 'both'").head(10))
```

### 8.3 BI Tool Cutover (ODBC / JDBC)

| Tool | Old (HDI Hive) | New (Fabric SQL endpoint / Warehouse) |
|------|---------------|---------------------------------------|
| Power BI Desktop | Microsoft Hive ODBC + workspace credential | **Built-in Fabric connector** (Lakehouse or Warehouse) — Entra ID SSO |
| Tableau | Hortonworks Hive ODBC | **Microsoft SQL Server connector** (use Fabric SQL endpoint URL) |
| Excel | Hive ODBC DSN | **Get Data → SQL Server** (paste SQL endpoint) |
| Generic JDBC client | `jdbc:hive2://<host>:443/...;transportMode=http` | `jdbc:sqlserver://<workspace>.datawarehouse.fabric.microsoft.com:1433;database=<name>;authentication=ActiveDirectoryInteractive` |
| Beeline scripts | `beeline -u jdbc:hive2://...` | **sqlcmd** (`sqlcmd -S <endpoint> -d <db> -G`) — `-G` enables AAD auth |
| pyhive | `from pyhive import hive` | `import pyodbc` with ODBC Driver 18 for SQL Server |

```bash
# Get the SQL endpoint URL of a Lakehouse via REST API
ENDPOINT=$(az rest --method get --resource "https://api.fabric.microsoft.com" \
  --url "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/lakehouses/$LH_ID" \
  --query "properties.sqlEndpointProperties.connectionString" -o tsv)
echo "SQL endpoint: $ENDPOINT"

# Connect with sqlcmd (Entra ID auth)
sqlcmd -S "$ENDPOINT" -d "MyLakehouse" -G \
       -Q "SELECT TOP 10 * FROM dbo.orders;"
```

```python
# pyodbc — replaces pyhive
import pyodbc, struct
from azure.identity import DefaultAzureCredential

cred = DefaultAzureCredential()
token = cred.get_token("https://database.windows.net/.default").token.encode("utf-16-le")
SQL_COPT_SS_ACCESS_TOKEN = 1256
attrs = {SQL_COPT_SS_ACCESS_TOKEN: struct.pack(f"=i{len(token)}s", len(token), token)}

cnxn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=<workspace>.datawarehouse.fabric.microsoft.com,1433;"
    "Database=MyWarehouse;Encrypt=yes;TrustServerCertificate=no;",
    attrs_before=attrs,
)
df = pd.read_sql("SELECT TOP 10 * FROM dbo.orders", cnxn)
```

---

## 9. Security Mapping — Ranger → T-SQL RLS / DDM

### 9.1 Ranger Hive Policy Patterns

| Ranger Policy | Fabric Replacement |
|---------------|-------------------|
| Database-level SELECT for group | Workspace **Viewer** role; or per-Lakehouse share |
| Table-level SELECT | T-SQL `GRANT SELECT ON dbo.t TO [group@tenant]` |
| Column-level SELECT (allow list) | Create a **VIEW** that projects allowed columns, grant on view |
| Column mask (e.g. mask SSN) | **Dynamic Data Masking** — `ALTER COLUMN ssn ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)')` |
| Row filter (`region = '${user.region}'`) | T-SQL **Row-Level Security** policy |
| Tag-based mask / filter | Manual — Fabric has no Purview tag inheritance into SQL endpoint yet |
| Audit logs | Fabric Activity Log + Purview integration |

### 9.2 RLS Example
```sql
-- Equivalent of Ranger row filter: user only sees their region
CREATE FUNCTION dbo.fn_orders_filter(@region NVARCHAR(50))
    RETURNS TABLE WITH SCHEMABINDING
AS RETURN
    SELECT 1 AS fn_result
    WHERE @region = (SELECT region FROM dbo.user_region WHERE upn = USER_NAME())
       OR IS_ROLEMEMBER('SalesAdmin') = 1;

CREATE SECURITY POLICY orders_rls
    ADD FILTER PREDICATE dbo.fn_orders_filter(region) ON dbo.orders
    WITH (STATE = ON);
```

### 9.3 DDM Example
```sql
-- Mask SSN: show only last 4 digits unless user has UNMASK permission
ALTER TABLE dbo.customers
  ALTER COLUMN ssn ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)');

-- Grant unmask to a specific role
GRANT UNMASK TO [HRAdmins@contoso.com];
```

---

## 10. Validation & Acceptance

```yaml
migration_pass_criteria:
  data_correctness:
    - per_table_row_count_match: exact (or ≤0.01% delta)
    - per_table_aggregate_match: exact (sum/count/avg of all numeric columns)
    - sample_query_set_match: 100%   # canonical 50-100 business queries
  performance:
    - p50_query_runtime_delta_pct: ≤ 30   # Fabric ≤ 1.3× LLAP baseline
    - p95_query_runtime_delta_pct: ≤ 80
    - direct_lake_no_fallback_for_top10_dashboards: true
  reliability:
    - sql_endpoint_uptime_7d_pct: > 99.5
    - no_throttling_events_7d: true
  security:
    - ranger_policy_count_mapped_to_rls_or_grant: 100%
    - all_pii_columns_masked: true
```

---

## 11. Cost Re-Modeling

| HDI IQ Cost Driver | Fabric Equivalent | Optimization Tip |
|--------------------|-------------------|------------------|
| LLAP daemon nodes always-on (D14 × 4 × 24 × 30 ≈ $4500/mo) | Fabric capacity CU consumption (only when query runs) | Pure win for spiky / business-hours workloads |
| Head/Hive Metastore VMs (~$300/mo) | None — included in Fabric | Pure savings |
| ADLS storage | OneLake (same $/GB) | `VACUUM` weekly to control history |
| ZooKeeper / Ambari | None | Pure savings |
| AAD DS (~$110/mo) | None — Entra ID | Pure savings |
| LLAP cache RAM | Direct Lake semantic-model cache (included in capacity) | Tune semantic model size to fit guardrails |

**Sizing rule of thumb**:
- HDI IQ cluster of 4× D14 LLAP nodes (~64 vCores, 448 GB RAM) ≈ **F32–F64** capacity
  for similar interactive concurrency. Bigger if BI report concurrency is high.
- For dev/test: **Pause** the F SKU outside business hours via REST.

---

## 12. Pitfalls & Lessons Learned

1. **ACID ORC `delta_*` directories aren't readable as ORC**. Spark/Fabric will read
   garbage if you point at the raw path. Always go through Hive (`INSERT OVERWRITE
   DIRECTORY ... STORED AS PARQUET`) on HDI before copying.

2. **Bucketed Hive joins lose co-location**. After migrating to Delta with Liquid
   Clustering, a previously bucket-pruned join becomes a regular shuffle. Test
   performance of joins that previously relied on bucketing — may need clustering on
   join keys explicitly.

3. **Hive ORC indexes vs Delta data skipping — different shape**. ORC has predicate
   pushdown via stripe-level min/max + bloom filters. Delta has file-level
   min/max/null counts. Bloom-filter-heavy point lookups may slow down — consider
   **Z-ORDER** or **Liquid Clustering** on the predicate columns.

4. **Lakehouse SQL endpoint is read-only**. Any HQL `INSERT/UPDATE/DELETE/MERGE` must
   go to either Spark (writes the underlying Delta) or to **Fabric Warehouse**. Don't
   try to point an ETL tool's writeback at the SQL endpoint.

5. **Hive views with Hive-specific UDFs break**. Recreate the view referencing the
   Spark/T-SQL equivalent of the UDF. Dump all view DDLs upfront via `SHOW CREATE
   TABLE` and audit each.

6. **`enableHiveSupport()` left in Spark code** (if migrating Spark consumers too) —
   ignored by Fabric and may resolve `default` differently. Remove explicitly.

7. **Direct Lake fallback to DirectQuery is silent**. If a table isn't V-Order Delta
   (because it came from a shortcut to non-Fabric Delta or copied raw), Power BI works
   but slowly. Enable Power BI Desktop **Performance Analyzer** or check the
   `DirectLakeFallbackReason` semantic-model property.

8. **Fabric Warehouse engine is NOT Spark and NOT Synapse Dedicated**. It runs
   distributed T-SQL over Delta in OneLake but doesn't support Spark UDFs, doesn't
   read non-Delta Parquet via SHOW TABLES (use `OPENROWSET`), and exposes a different
   T-SQL surface than Azure SQL DB or Synapse Dedicated SQL pool. Verify each feature
   against the current `learn.microsoft.com` Fabric Warehouse T-SQL surface area docs
   before assuming it works.

9. **Hive Metastore custom properties (`TBLPROPERTIES`) are not preserved**. ORC-
   specific props (`orc.compress`, `orc.bloom.filter.columns`) have no Delta
   equivalent. Replace bloom-filter intent with column ordering / clustering.

10. **HQL `INSERT OVERWRITE DIRECTORY` for ad-hoc exports**. Pattern doesn't exist in
    Spark SQL or T-SQL. Replace with `df.write.csv(...)` in a Spark notebook, or
    `COPY INTO` from Warehouse to ADLS.

11. **Hive ACID `MAJOR COMPACT` cron jobs**. No Fabric equivalent — replace with
    scheduled `OPTIMIZE` + `VACUUM` pipelines.

12. **`SET hive.exec.dynamic.partition.mode=nonstrict`** (and similar `SET` calls in
    HQL). All silently ignored in Spark/T-SQL — but the semantics differ. In Spark,
    dynamic partition writes need `INSERT OVERWRITE` + matching column count;
    Warehouse has no partitioning at all.

13. **Beeline transport mode/path differences**. HDI used `transportMode=http;httpPath=cliservice`.
    Fabric uses TDS over port 1433 — completely different protocol. Drop Beeline; use
    sqlcmd / pyodbc / SQL Server JDBC.

14. **Hive concurrency limits (LLAP queue full)** vs **Fabric capacity throttling** —
    different failure modes. LLAP queues; Fabric throttles or rejects requests when
    capacity is exhausted. Set up Capacity Metrics alerts BEFORE cutover.

15. **`SHOW PARTITIONS` syntax**. Hive: `SHOW PARTITIONS db.table`. Spark: same.
    Warehouse: tables aren't partitioned in the Hive sense; query `sys.partitions`
    via T-SQL — but it returns physical-storage partitions, not logical Hive ones.

---

## 13. Quick-Reference Migration Checklist

```
[ ] Inventory all HDI IQ databases, tables, views, UDFs, ACID flags, sizes, ownership
[ ] Inventory client code: BI tools, ODBC DSNs, Beeline scripts, JDBC apps
[ ] Categorize each object: target = Lakehouse-SQL / Warehouse / Spark notebook
[ ] Provision Fabric workspace + capacity (start with F32 or trial)
[ ] Provision destination Lakehouse(s) and/or Warehouse(s)
[ ] Grant Workspace Identity access to legacy ADLS
[ ] Create shortcuts for read-only validation (skip ACID tables)
[ ] For ACID tables: export via Hive INSERT OVERWRITE DIRECTORY → Parquet on ADLS
[ ] Translate Hive DDL → Delta DDL (Lakehouse) or T-SQL DDL (Warehouse)
[ ] Bulk-load tables (Spark CTAS or Pipeline Copy Activity)
[ ] Translate HQL → Spark SQL / T-SQL (sqlglot bulk + manual fix-ups)
[ ] Rewrite Java UDFs as Spark UDFs OR pre-materialize via Spark
[ ] Map Ranger policies → T-SQL GRANT/RLS/DDM
[ ] Cut BI tools (Power BI / Tableau / Excel) to new endpoint
[ ] Run dual-read validation for ≥7 days
[ ] Cut ETL writers (HQL scripts → Spark notebooks / T-SQL stored procs)
[ ] Schedule weekly OPTIMIZE + VACUUM maintenance pipeline
[ ] Build Capacity Metrics dashboards + throttling alerts
[ ] Tear down HDI IQ cluster (keep ADLS for 30 days)
```

---

## 14. Useful Commands & Snippets

```bash
# Get Lakehouse SQL endpoint URL
az rest --method get --resource "https://api.fabric.microsoft.com" \
  --url "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/lakehouses/$LH_ID" \
  --query "properties.sqlEndpointProperties.connectionString" -o tsv

# Get Warehouse endpoint URL
az rest --method get --resource "https://api.fabric.microsoft.com" \
  --url "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/warehouses/$DW_ID" \
  --query "properties.connectionString" -o tsv

# Connect with sqlcmd (Entra ID interactive)
sqlcmd -S "<endpoint>" -d "<db>" -G -Q "SELECT @@VERSION"

# Bulk OPTIMIZE all tables in a Lakehouse (run as a notebook)
for row in spark.sql("SHOW TABLES IN sales_lakehouse").collect():
    spark.sql(f"OPTIMIZE sales_lakehouse.{row['tableName']}")
    spark.sql(f"VACUUM   sales_lakehouse.{row['tableName']} RETAIN 168 HOURS")

# Warehouse: list tables + row counts
sqlcmd -S "$DW" -d "MyDW" -G -Q "
SELECT s.name + '.' + t.name AS tbl, SUM(p.rows) AS rows
FROM sys.tables t JOIN sys.schemas s ON t.schema_id=s.schema_id
JOIN sys.partitions p ON p.object_id=t.object_id AND p.index_id IN (0,1)
GROUP BY s.name, t.name ORDER BY rows DESC;"
```

---

## Rules

1. **Never lift-and-shift ACID ORC tables** — always export via Hive to Parquet, then ingest as Delta.
2. **Pick Lakehouse SQL endpoint for read-only workloads** — free, Direct Lake-ready, fastest cutover.
3. **Pick Fabric Warehouse for T-SQL writes / stored procs / multi-statement transactions** — not Lakehouse.
4. **Drop Hive bucketing; replace with Liquid Clustering on Delta** — preserve join performance intent without bucket file fragmentation.
5. **Always V-Order Delta tables** that feed Power BI Direct Lake — verify no fallback to DirectQuery.
6. **Schedule OPTIMIZE + VACUUM weekly** — Fabric does not auto-maintain Delta tables.
7. **Replace LLAP cache with the layered model**: Direct Lake (BI) + Result-Set Cache (Warehouse) + V-Order + Liquid Clustering + dataset-level statistics.
8. **Map Ranger policies deliberately** — communicate row-level / column-mask changes to consumers before cutover.
9. **Use sqlglot to bulk-translate HQL** — but plan for ~20% manual fix-up (UDFs, hints, custom serdes).
10. **Keep the HDI IQ cluster + source ADLS for 30 days post-cutover** — instant rollback path.
