---
name: azure-hdinsight-migration-hbase-to-fabric-cosmosdb
description: End-to-end migration playbook for moving Azure HDInsight HBase (and Phoenix-on-HBase) workloads to Azure Cosmos DB (NoSQL or Cassandra API) and/or Cosmos DB in Microsoft Fabric (mirrored database) — schema redesign (column-family → JSON document or wide-column), row-key → partition-key translation, RU/s sizing from HBase region/throughput, bulk export (HFile/Snapshot → Cosmos bulk import), client SDK rewrites, Phoenix SQL → Cosmos SQL/T-SQL via Mirroring, security mapping, dual-write cutover, validation.
license: MIT
version: 1.0.0
updated: 2026-04-29
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
triggers: hdinsight hbase, hbase migration, hbase to cosmos, phoenix migration, cosmos db nosql, cosmos cassandra, cosmos mirroring, fabric mirroring cosmos, hbase snapshot, region split, rowkey design, wide column, column family
requires: azure-hdinsight, azure-cosmosdb, azure-fabric
---
# Migrating HDInsight HBase → Azure Cosmos DB (and/or Fabric Mirroring)

A field guide for migrating production **HDInsight HBase** clusters (HBase 2.x with
Phoenix SQL, REST gateway, Thrift) to:

1. **Azure Cosmos DB for NoSQL** — primary target for new development; richest API,
   serverless option, Direct Lake-friendly via **Fabric mirroring**.
2. **Azure Cosmos DB for Apache Cassandra** — minimal-rewrite target when the existing
   data model is genuinely wide-column (CQL drop-in works for many HBase-via-Phoenix patterns).
3. **Cosmos DB Mirroring in Microsoft Fabric** — analytics overlay, not a write target.
   Mirrored Cosmos data appears as Delta in OneLake for Power BI / Spark / KQL.

> **Why this migration?** HDInsight 4.0 has reached end-of-standard-support; HDInsight
> 5.x is the last classic generation, and HBase has no continuation in HDInsight on AKS.
> Microsoft's recommended landing zone for **HDInsight HBase** is **Cosmos DB**.
> For OLAP overlays previously served by Phoenix-on-HBase + HDI Spark, the recommended
> pattern is **Cosmos DB (write) + Fabric Mirroring (read for analytics)**.

---

## 1. Decision Matrix — Which Cosmos DB Path?

| Existing HDI HBase Pattern | Recommended Target | Reason |
|---------------------------|-------------------|--------|
| App-driven OLTP key-value access (`get`/`put` by row key) | **Cosmos DB for NoSQL** | Best SDK support, lowest latency, serverless option |
| Wide-column model with CQL or Phoenix already | **Cosmos DB for Cassandra** | Drop-in CQL clients; existing Phoenix → CQL is straightforward |
| Time-series telemetry (one row per device per timestamp) | **Cosmos DB for NoSQL** with TTL OR **Azure Data Explorer / KQL DB** | NoSQL works; ADX/KQL is cheaper at sustained high ingest |
| Phoenix SQL secondary indexes used heavily | **Cosmos DB for NoSQL** with composite/range indexes | Cosmos auto-indexes everything; tune via index policy |
| Phoenix SQL JOINs across HBase tables | **Cosmos DB has no joins across containers** — denormalize OR use Fabric mirroring for analytics joins | Re-model is required either way |
| HBase Coprocessors (server-side aggregations) | **Stored Procedures / pre-triggers** in Cosmos DB | Different model; rewrite per-coprocessor |
| Heavy analytical scans (full-table) on HBase | **Mirror Cosmos DB → Fabric Lakehouse / KQL DB** for analytics | Don't run scans against transactional Cosmos |
| BI dashboards on HBase (via Phoenix JDBC) | **Cosmos DB + Fabric Mirroring → Direct Lake on OneLake** | Sub-second BI without scanning live OLTP store |
| OpenTSDB on HBase | **Azure Data Explorer** (purpose-built time-series) | Cosmos works; ADX is the better fit |
| Strict on-prem / air-gapped | Stay on HBase via **HBase on AKS** (community charts) | Cosmos is SaaS only |
| Very large datasets (>10 TB) with mostly cold data | **Cosmos DB Analytical Store** (auto-tiered) OR **ADLS Gen2 + Delta** | Cosmos hot tier is expensive at TB scale |

**Decision shortcut**: If your HBase is **OLTP-style with row-key lookups + small scans**,
go **Cosmos DB for NoSQL**. If you have **CQL-via-Phoenix** clients with wide-column
schemas, go **Cosmos DB for Cassandra**. **Always** add **Fabric Mirroring** when there
is downstream analytics or Power BI consumption.

---

## 2. Assessment Phase

### 2.1 Inventory the HDInsight HBase Cluster

```bash
ssh sshuser@<cluster>-ssh.azurehdinsight.net

# Cluster + service version metadata
curl -s -u admin:$PASS \
  "https://<cluster>.azurehdinsight.net/api/v1/clusters/<cluster>" \
  | python3 -m json.tool > cluster_meta.json

# HBase version
hbase version 2>&1 | tee hbase_version.txt

# Master / RegionServer status
echo "status" | hbase shell -n > status.txt
echo "status 'detailed'" | hbase shell -n > status_detailed.txt
echo "status 'replication'" | hbase shell -n > status_replication.txt 2>/dev/null

# All tables + descriptions
echo "list" | hbase shell -n | tail -n +2 > tables.txt

> table_inventory.tsv
echo -e "table\tcf\tversions\tcompression\tttl\tregions\tsize_mb" >> table_inventory.tsv

while read t; do
  desc=$(echo "describe '$t'" | hbase shell -n 2>/dev/null)
  # Extract column families
  cfs=$(echo "$desc" | grep -oE "NAME => '[^']*'" | sed "s/NAME => '//;s/'//")
  versions=$(echo "$desc" | grep -oE "VERSIONS => '[0-9]+'" | head -1 | sed "s/VERSIONS => '//;s/'//")
  compress=$(echo "$desc" | grep -oE "COMPRESSION => '[^']*'" | head -1 | sed "s/COMPRESSION => '//;s/'//")
  ttl=$(echo "$desc"   | grep -oE "TTL => '[^']*'" | head -1 | sed "s/TTL => '//;s/'//")

  # Region count + size (read from HBase Master UI JMX or hbase shell)
  regions=$(echo "list_regions '$t'" | hbase shell -n 2>/dev/null | tail -n +2 | wc -l)

  # Approximate size from HDFS
  size=$(hdfs dfs -du -s /hbase/data/default/$t 2>/dev/null | awk '{print int($1/1024/1024)}')

  for cf in $cfs; do
    echo -e "$t\t$cf\t$versions\t$compress\t$ttl\t$regions\t$size" >> table_inventory.tsv
  done
done < tables.txt

# Hot regions / region splits — symptom of bad row key design
curl -s "http://$(hostname):16010/jmx" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for b in d['beans']:
  if 'regionserver' in b.get('name','').lower():
    n = b.get('numRegions', 0); req = b.get('readRequestCount', 0) + b.get('writeRequestCount', 0)
    print(f'{b[\"tag.Hostname\"]}: regions={n} reqs={req}')
" > regionserver_load.txt

# Replication topology (if cross-cluster replication enabled)
echo "list_peers" | hbase shell -n > replication_peers.txt 2>/dev/null

# Phoenix tables (if Phoenix is installed)
/usr/hdp/current/phoenix-client/bin/sqlline.py localhost:2181:/hbase-unsecure \
  --silent=true --outputformat=tsv -e "!tables" > phoenix_tables.tsv 2>/dev/null

# Per-table read/write request counts (last reset)
echo "count_distinct_table_requests = true" >> /tmp/get_metrics.rb
hbase org.apache.hadoop.hbase.tool.MetricsReportTool > metrics_report.txt 2>/dev/null || true

# Actual sample data (for schema design — pull 10 rows per table)
for t in $(cat tables.txt); do
  echo "=== $t ==="
  echo "scan '$t', {LIMIT => 10}" | hbase shell -n 2>/dev/null | head -100
done > sample_data.txt

# Coprocessors loaded
for t in $(cat tables.txt); do
  echo "describe '$t'" | hbase shell -n 2>/dev/null \
    | grep -i COPROCESSOR | sed "s/^/$t: /"
done > coprocessors.txt
```

### 2.2 Inventory the Client Codebase

```bash
cd /path/to/repos

# 1) HBase Java client
grep -RInE 'org\.apache\.hadoop\.hbase|HTable|Connection.*ConnectionFactory|Admin.*createTable' \
  --include='*.java' --include='*.scala'

# 2) HBase REST gateway clients
grep -RInE 'hbaserest|/hbaserest/|Stargate' \
  --include='*.py' --include='*.java' --include='*.js' --include='*.ts' --include='*.cs'

# 3) HBase Thrift clients
grep -RInE 'hbase\.thrift|THBaseService|TIOError' \
  --include='*.py' --include='*.java' --include='*.cs'

# 4) happybase (Python)
grep -RInE 'import happybase|happybase\.Connection' --include='*.py'

# 5) Phoenix JDBC
grep -RInE 'jdbc:phoenix|phoenix.*PhoenixDriver|PhoenixConnection' \
  --include='*.java' --include='*.properties' --include='*.scala' --include='*.py'

# 6) Spark-on-HBase / SHC connector
grep -RInE 'org\.apache\.hadoop\.hbase\.spark|SparkOnHBase|HBaseRDD|spark\.hbase' \
  --include='*.scala' --include='*.py' --include='*.java'

# 7) Coprocessor Java implementations
grep -RInE 'BaseRegionObserver|BaseEndpointCoprocessor|RegionCoprocessor' \
  --include='*.java'

# 8) Bulk-load (HFile generation) jobs
grep -RInE 'HFileOutputFormat|LoadIncrementalHFiles|completeBulkLoad' \
  --include='*.java' --include='*.scala' --include='*.sh'

# 9) MapReduce ETL talking to HBase (TableInputFormat / TableOutputFormat)
grep -RInE 'TableInputFormat|TableOutputFormat|TableMapReduceUtil' \
  --include='*.java' --include='*.scala'
```

### 2.3 Categorize Each Table / Workload

Build a CSV: `table, cf_count, row_count_est, size_gb, ops_pattern, throughput_rps,
phoenix_indexed, coprocessor_used, target_api, complexity, notes`.

| Complexity | Definition | Migration Effort |
|------------|------------|------------------|
| **Low**    | Single CF, no coprocessor, simple `get`/`put` by row key, <100 GB | hours-days |
| **Medium** | Multiple CFs, Phoenix tables with secondary indexes, scan-heavy queries, 100 GB - 1 TB | days-weeks |
| **High**   | Coprocessors with custom server-side aggregation, `MAJOR_COMPACT` cron jobs, MOB columns, >1 TB | 2-6 weeks |
| **Blocker**| Cross-cluster HBase replication, custom RegionObservers with stateful logic, AsyncHBase clients with very tight latency SLAs (<1ms p99) | re-architect or stay on HBase-on-AKS |

---

## 3. Schema Redesign — Column-Family → Document / Wide-Column

### 3.1 Mental Model Shift

| HBase Concept | Cosmos DB for NoSQL | Cosmos DB for Cassandra |
|---------------|--------------------|--------------------------|
| **Table** | Container (collection) | Table |
| **Column Family** | Nested object property in JSON | Column family (1:1) |
| **Column Qualifier** | Field inside nested object | Column |
| **Row Key** | `id` + partition key | Primary key (partition + clustering) |
| **Cell Value** | JSON property value | Column value |
| **Timestamp / Version** | Item `_ts` (last-write only); use change feed for history; or embed version in `id` | TTL + write timestamps |
| **Region** | Physical partition (managed) | Token range (managed) |
| **Pre-split** | Define partition key with even cardinality | Same |
| **TTL on cell** | TTL on item (`/ttl` or container default) | TTL per column or row |
| **Coprocessor (aggregation)** | Stored procedure (single partition) OR materialize via Change Feed → Spark/KQL | Materialized views (limited) |
| **MOB (Medium Object)** | Store small (<2 MB) inline; large in Blob Storage with URL reference | Same |

### 3.2 Row-Key → Partition-Key Translation

HBase row key design is a single string with sort-locality semantics. Cosmos NoSQL
splits into **`id`** (unique within partition) + **partition key** (for distribution).

| HBase Row Key Pattern | Cosmos Partition Key | Cosmos `id` | Notes |
|-----------------------|---------------------|-------------|-------|
| `<deviceId>|<reverseTimestamp>` | `/deviceId` | `<reverseTimestamp>` (or natural string) | Time-series per device |
| `<userId>|<eventType>|<ts>` | `/userId` | `<eventType>|<ts>` | User activity feed |
| `<hash(userId)>_<userId>` (salted) | `/userId` | `<userId>` | Cosmos handles distribution; drop the salt prefix |
| `<reversedDomain>|<path>` (URL crawl) | `/reversedDomain` | `<path>` | Web crawl style |
| Short composite (`<region>|<id>`) for region-locality | `/region` | `<id>` | Region locality is implicit in partition key |
| Pure UUID/GUID | `/id` (id == partition key) | `<uuid>` | Even distribution; OK for OLTP |

**Critical rule**: A partition key must have **high cardinality** (≥1000 distinct
values) and **even read/write distribution**. The HBase salting trick (prefix-hash)
is **not needed** in Cosmos — Cosmos hashes the partition-key value internally.

### 3.3 Column Family → JSON Document Translation

```
HBase row:
  rowkey = "device-001|20240429T120000"
  cf "telemetry":
    qualifier "temperature" = 22.5
    qualifier "humidity"    = 65.2
  cf "metadata":
    qualifier "firmware"    = "v1.2.3"
    qualifier "site"        = "warehouse-a"
```

becomes

```json
// Cosmos DB for NoSQL document
{
  "id": "20240429T120000",
  "deviceId": "device-001",                // partition key
  "telemetry": {
    "temperature": 22.5,
    "humidity": 65.2
  },
  "metadata": {
    "firmware": "v1.2.3",
    "site": "warehouse-a"
  },
  "_ts": 1714392000                        // Cosmos auto-stamps; use as version
}
```

```cql
-- Cosmos DB for Cassandra (CQL DDL — wide-column)
CREATE TABLE telemetry.devices (
    device_id     text,
    event_time    timestamp,
    temperature   double,
    humidity      double,
    firmware      text,
    site          text,
    PRIMARY KEY ((device_id), event_time)    -- partition by device, cluster by time
) WITH CLUSTERING ORDER BY (event_time DESC);
```

### 3.4 Phoenix Schema → Cosmos Schema

Phoenix maps SQL onto HBase row keys. Translate:

| Phoenix DDL | Cosmos for NoSQL | Cosmos for Cassandra |
|-------------|-----------------|----------------------|
| `PRIMARY KEY (tenant_id, user_id)` | `id="<user_id>"`, partition key `/tenant_id` | `PRIMARY KEY ((tenant_id), user_id)` |
| `PRIMARY KEY (event_date, device_id)` | `id="<device_id>:<seq>"`, partition key `/event_date` (be careful — date may be low cardinality!) | `PRIMARY KEY ((event_date), device_id)` |
| `CREATE INDEX idx ON t (user_email)` | Add `user_email` to indexing policy `includedPaths` (Cosmos auto-indexes) | `CREATE INDEX ON t (user_email);` |
| `SALT_BUCKETS = 16` | **Drop the salt** — Cosmos auto-distributes | Drop |
| `IMMUTABLE_ROWS = true` | No equivalent — design app to never update | No equivalent |
| `MULTI_TENANT = true` | Use partition key = tenant_id | Same — `PRIMARY KEY ((tenant_id), ...)` |
| `VIEW` over base table | Manually denormalize OR use Fabric mirroring + Lakehouse view | Materialized view (limited support) |

---

## 4. RU/s Sizing from HBase Throughput

### 4.1 Estimating From RegionServer Metrics

```bash
# Pull peak read/write RPS from RegionServer JMX over 1 week
curl -s "http://<rs-host>:16030/jmx?qry=Hadoop:service=HBase,name=RegionServer,sub=Server" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
b = d['beans'][0]
print(f'readRequestCount={b[\"readRequestCount\"]}')
print(f'writeRequestCount={b[\"writeRequestCount\"]}')
"
```

### 4.2 Ballpark RU/s Conversion

Cosmos DB Request Unit (RU) costs (per 1 KB item):

| Operation | Approx RU |
|-----------|-----------|
| Point read (GET by `id` + partition key) | **1 RU** |
| Point write (PUT/upsert, lightly indexed) | **5–10 RU** |
| Indexed query (single partition) | **2.5 RU + per-item** |
| Cross-partition query | RU × number-of-partitions touched |

**Sizing recipe**:

```
RU/s required ≈
    (peak_reads_per_sec × avg_item_size_kb × 1)
  + (peak_writes_per_sec × avg_item_size_kb × 7)        # 5-10 indexed writes
  + (peak_query_per_sec × avg_query_ru)                 # measure during pilot
  × 1.5                                                 # 50% headroom
```

For an HBase cluster doing 5,000 reads/s + 2,000 writes/s on ~1 KB rows:
`(5000×1) + (2000×7) = 19,000 RU/s × 1.5 = ~28,500 RU/s` → start at **30,000 RU/s autoscale**.

**Tip**: Do a pilot with a representative workload, capture `x-ms-request-charge`
from response headers, then re-size — sizing on paper is always 30-50% wrong.

### 4.3 Provisioning Modes

| Workload | Mode | Why |
|----------|------|-----|
| Predictable steady traffic | **Provisioned, manual** | Cheapest at steady state |
| Spiky / business-hours | **Autoscale** | Pays for peak per hour, scales 10–100% |
| Dev/test or < 1M ops/month | **Serverless** | No minimum, pay per RU |

---

## 5. Bulk Data Migration

### 5.1 Pattern A — Spark Bulk Load (Recommended)

Use a **Fabric Spark notebook** (or your existing HDInsight Spark cluster as a
transitional bridge) to read from HBase and write to Cosmos DB in parallel.

```python
# Read HBase → Spark DataFrame
# (Use SHC, HBase Spark connector, or REST/Thrift with pyspark)

# Option 1: HBase REST → DataFrame (works without HBase JARs)
import requests, pandas as pd
def fetch_table(host, table, batch=1000):
    # Use scanner API with chunked rows
    resp = requests.post(f"https://{host}/hbaserest/{table}/scanner",
        headers={"Accept":"application/json","Content-Type":"text/xml"},
        data=f"<Scanner batch='{batch}'/>")
    scanner = resp.headers["Location"]
    while True:
        r = requests.get(scanner, headers={"Accept":"application/json"})
        if r.status_code == 204: break
        yield from r.json().get("Row", [])
    requests.delete(scanner)

# Convert HBase rows → flattened JSON
def hbase_row_to_cosmos(row):
    rowkey = base64.b64decode(row["key"]).decode()
    doc = {"id": rowkey.split("|", 1)[1], "deviceId": rowkey.split("|", 1)[0]}
    for c in row["Cell"]:
        col = base64.b64decode(c["column"]).decode()  # "cf:qualifier"
        cf, q = col.split(":", 1)
        val = base64.b64decode(c["$"]).decode()
        doc.setdefault(cf, {})[q] = val
    return doc

# Option 2 (preferred for scale): HBase Spark connector
# spark.read.format("org.apache.hadoop.hbase.spark") ...

# Bulk write to Cosmos via the Spark connector
COSMOS_CONFIG = {
    "spark.cosmos.accountEndpoint": "https://my-cosmos.documents.azure.com:443/",
    "spark.cosmos.accountKey":      "<key>",
    "spark.cosmos.database":        "telemetry",
    "spark.cosmos.container":       "devices",
    "spark.cosmos.write.strategy":  "ItemBulkUpdate",  # bulk mode
    "spark.cosmos.write.bulk.enabled": "true",
}
df.write.format("cosmos.oltp").options(**COSMOS_CONFIG).mode("append").save()
```

> **Throughput tip**: During backfill, **temporarily raise Cosmos RU/s** (e.g., from
> 4,000 to 50,000), let bulk writes complete, then scale back down. Bulk import is
> RU-bound, not CPU-bound.

### 5.2 Pattern B — HBase Snapshot → S3/Blob → Spark → Cosmos

For very large tables (>1 TB), avoid live RPC scans:

```bash
# On HDI HBase
echo "snapshot 'telemetry', 'telemetry-snap-20240429'" | hbase shell -n

# Export snapshot to ADLS/Blob
hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot telemetry-snap-20240429 \
  -copy-to wasbs://exports@hdistore.blob.core.windows.net/snapshots/ \
  -mappers 16

# Now read snapshot files in Spark on Fabric (snapshot format includes HFile + manifest)
# Use the hbase-spark library to parse HFiles, then write to Cosmos as above
```

### 5.3 Pattern C — Azure Data Factory / Synapse Pipeline Copy Activity

For one-shot small tables (<10 GB) where you don't want to spin up Spark, use **ADF
/ Synapse / Fabric Pipeline** with a Copy Activity. Source: ADLS JSON/Parquet (after
an HBase export). Sink: Cosmos DB connector (NoSQL API supported natively; Cassandra
API via the standard CQL connector).

> **Note on the legacy Cosmos DB Data Migration Tool**: The desktop tool at
> `Azure/azure-documentdb-datamigrationtool` (`dt.exe`) was **archived in Nov 2023**
> and is no longer maintained. Don't use it for new work — prefer the Spark connector
> (Pattern A), HBase snapshot + Spark (Pattern B), or Pipeline Copy Activity
> (this pattern).

### 5.4 Pattern D — Live Replication (Dual-Write Bridge)

For zero-downtime cutover, run a **bridge process** that consumes HBase WAL or change
events and replays them into Cosmos. Two viable approaches:

- **HBase replication peer** to a custom endpoint that translates and POSTs to Cosmos.
- **Application dual-write**: app writes to both HBase and Cosmos for the cutover
  window. Simpler when you control the app.

---

## 6. Phoenix SQL → Cosmos / Fabric Translation

### 6.1 Common Phoenix Patterns

| Phoenix SQL | Cosmos NoSQL Query | Cosmos for Cassandra (CQL) |
|-------------|--------------------|-----------------------------|
| `SELECT * FROM t WHERE pk = 'x'` (point read) | `SELECT * FROM c WHERE c.id = "x" AND c.tenantId = "t"` (or SDK `read_item`) | `SELECT * FROM t WHERE tenant_id='t' AND pk='x';` |
| `SELECT * FROM t WHERE col1 = ?` (secondary idx) | Same, with `col1` in `includedPaths` | `SELECT * FROM t WHERE col1 = ? ALLOW FILTERING;` (use SAI) |
| `UPSERT INTO t VALUES (?, ?, ?)` | SDK `upsert_item({...})` | `INSERT INTO t (...) VALUES (...);` |
| `DELETE FROM t WHERE pk = ?` | SDK `delete_item(id, partition_key)` | `DELETE FROM t WHERE pk = ?;` |
| `SELECT COUNT(*) FROM t WHERE pk = ?` (single partition) | Same with `VALUE COUNT(1)` | Same |
| `JOIN` between Phoenix tables | **Not supported** in Cosmos NoSQL — denormalize OR Fabric mirror + Lakehouse SQL endpoint | Same — denormalize |
| `GROUP BY` on whole table | Run via **Fabric mirror → Spark/KQL** | Run via Fabric/Spark |
| `CREATE INDEX` | Add path to indexing policy (auto, no rebuild) | `CREATE CUSTOM INDEX` (SAI) |
| `CREATE VIEW` | Manually denormalize OR mirror + Lakehouse view | N/A |
| Subqueries / CTEs | Limited — push to Fabric mirror | N/A |

### 6.2 Side-by-Side Examples

```sql
-- Phoenix
SELECT user_id, COUNT(*) AS event_count, MAX(ts) AS last_ts
FROM events
WHERE tenant_id = 't42' AND ts > CURRENT_DATE() - 7
GROUP BY user_id
ORDER BY event_count DESC LIMIT 100;

-- Cosmos NoSQL (single partition only — must include partition key)
SELECT c.user_id,
       COUNT(1) AS event_count,
       MAX(c.ts) AS last_ts
FROM c
WHERE c.tenant_id = 't42' AND c.ts > DateTimeAdd("dd", -7, GetCurrentDateTime())
GROUP BY c.user_id
ORDER BY COUNT(1) DESC                         -- Note: ORDER BY on aggregate needs composite idx
OFFSET 0 LIMIT 100
```

For **cross-tenant** aggregations, run against **Fabric mirrored data** in Lakehouse:
```sql
-- Fabric Lakehouse SQL endpoint over mirrored Cosmos data
SELECT user_id, COUNT(*) AS event_count, MAX(ts) AS last_ts
FROM events_mirror
WHERE ts > DATEADD(DAY, -7, GETUTCDATE())
GROUP BY user_id
ORDER BY event_count DESC
OFFSET 0 ROWS FETCH NEXT 100 ROWS ONLY;
```

---

## 7. Cosmos DB Mirroring in Microsoft Fabric

**Mirroring** continuously replicates a Cosmos DB account into OneLake as Delta tables.
Read-only on the Fabric side; near-real-time (~seconds latency); **no compute cost on
the source Cosmos** for the mirrored read path.

### 7.1 Why Mirror?

- Run **analytical scans / GROUP BY / JOINs** without burning Cosmos RUs.
- Power BI **Direct Lake** semantic models on mirrored data.
- Spark / KQL DB queries across **multiple Cosmos databases** in one workspace.
- Replaces the legacy Phoenix-on-HBase + Spark-on-HBase analytics stack.

### 7.2 Enable Mirroring

```python
# Fabric REST — create a mirrored database for a Cosmos account
import requests
ws = "<workspace-id>"
hdrs = {"Authorization": f"Bearer {fabric_token}", "Content-Type": "application/json"}

resp = requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{ws}/mirroredDatabases",
    headers=hdrs,
    json={
        "displayName": "telemetry_mirror",
        "definition": {
            "format": "MirroredDatabase",
            "parts": [{
                "path": "mirroredDatabase.json",
                "payload": "<base64 of mirroredDatabase.json>",
                "payloadType": "InlineBase64"
            }]
        }
    },
)
```

The `mirroredDatabase.json` payload references the Cosmos account, the database, and
the containers to mirror. After enabling, each container appears as a Delta table
under `Tables/` in OneLake — queryable via Lakehouse SQL endpoint, Spark, KQL, and
Power BI Direct Lake.

### 7.3 What Mirroring Does Not Do

- **Not bidirectional** — Fabric → Cosmos writes don't exist via mirroring.
- **No DDL changes** — schema changes in Cosmos appear; manual schema evolution NOT supported.
- **Lag varies** with Cosmos write rate — typical 1-30s; not for sub-second SLAs.
- **Cosmos `Analytical Store`** is a different feature (older); **Fabric Mirroring is the
  current recommendation** and supersedes Synapse Link for Cosmos for new projects.

---

## 8. Client SDK Rewrites

### 8.1 happybase (Python) → Cosmos DB SDK

```python
# OLD — happybase on HDI HBase
import happybase
conn = happybase.Connection('hbase-thrift-host', port=9090)
table = conn.table('telemetry')

# put
table.put(b'device-001|20240429T120000', {b'cf:temperature': b'22.5', b'cf:humidity': b'65.2'})

# get
row = table.row(b'device-001|20240429T120000')

# scan
for key, data in table.scan(row_prefix=b'device-001|'):
    print(key, data)

# NEW — Cosmos DB for NoSQL
from azure.cosmos import CosmosClient, PartitionKey
from azure.identity import DefaultAzureCredential

client = CosmosClient("https://my-cosmos.documents.azure.com:443/", DefaultAzureCredential())
container = client.get_database_client("telemetry").get_container_client("devices")

# upsert (replaces put)
container.upsert_item({
    "id": "20240429T120000",
    "deviceId": "device-001",
    "cf": {"temperature": 22.5, "humidity": 65.2}
})

# read (replaces get) — point read, 1 RU
item = container.read_item("20240429T120000", partition_key="device-001")

# scan device's events (single partition)
for it in container.query_items(
    query="SELECT * FROM c WHERE c.deviceId = @d ORDER BY c.id DESC",
    parameters=[{"name":"@d","value":"device-001"}],
    partition_key="device-001",
):
    print(it)
```

### 8.2 HBase Java Client → Cosmos for NoSQL Java SDK

```java
// OLD — HBase
Connection conn = ConnectionFactory.createConnection(config);
Table table = conn.getTable(TableName.valueOf("telemetry"));
Put put = new Put(Bytes.toBytes("device-001|20240429T120000"));
put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("temperature"), Bytes.toBytes("22.5"));
table.put(put);

// NEW — Cosmos DB for NoSQL
CosmosClient client = new CosmosClientBuilder()
    .endpoint("https://my-cosmos.documents.azure.com:443/")
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();

CosmosContainer container = client.getDatabase("telemetry").getContainer("devices");

JsonNode doc = mapper.readTree("""
    {"id":"20240429T120000","deviceId":"device-001",
     "cf":{"temperature":22.5,"humidity":65.2}}
""");
container.upsertItem(doc, new PartitionKey("device-001"), new CosmosItemRequestOptions());
```

### 8.3 Phoenix JDBC → Cosmos for Cassandra (Datastax driver)

```java
// OLD — Phoenix JDBC
String url = "jdbc:phoenix:zk-host:2181:/hbase-unsecure";
Connection c = DriverManager.getConnection(url);
PreparedStatement ps = c.prepareStatement(
    "UPSERT INTO events (tenant_id, user_id, event_type, ts) VALUES (?,?,?,?)");
ps.setString(1, "t42"); ps.setString(2, "u1"); ps.setString(3, "click"); ps.setLong(4, ts);
ps.execute(); c.commit();

// NEW — Cosmos for Cassandra (Datastax driver)
CqlSession session = CqlSession.builder()
    .addContactPoint(new InetSocketAddress("my-cosmos.cassandra.cosmos.azure.com", 10350))
    .withSslContext(SSLContext.getDefault())
    .withAuthProvider(new ProgrammaticPlainTextAuthProvider("my-cosmos", primaryKey))
    .withLocalDatacenter("East US")
    .build();
session.execute(
    "INSERT INTO telemetry.events (tenant_id, user_id, event_type, ts) VALUES (?,?,?,?)",
    "t42", "u1", "click", ts);
```

### 8.4 HBase REST Gateway → Cosmos REST (or SDK)

The HBase REST gateway is **not migrated** — Cosmos DB has its own REST API and SDKs in
all major languages. If you have polyglot REST consumers, point them at the Cosmos REST
endpoint or wrap the SDK in a thin internal HTTP service.

---

## 9. Coprocessors → Stored Procedures / Change Feed

HBase coprocessors (server-side aggregation, validation) have two replacement patterns:

### 9.1 Cosmos Stored Procedures (single-partition only)

```javascript
// Cosmos JS stored procedure — atomic upsert + counter increment within one partition
function upsertWithCounter(doc) {
    var ctx = getContext(), col = ctx.getCollection();
    var counterDocId = doc.deviceId + "_counter";

    // 1) upsert event doc
    col.upsertDocument(col.getSelfLink(), doc, function(err) {
        if (err) throw err;

        // 2) read + increment counter doc
        col.readDocument(col.getAltLink() + "/docs/" + counterDocId, function(err, counter) {
            counter = counter || { id: counterDocId, deviceId: doc.deviceId, count: 0 };
            counter.count++;
            col.upsertDocument(col.getSelfLink(), counter, function(err) {
                if (err) throw err;
                ctx.getResponse().setBody({ inserted: doc.id, count: counter.count });
            });
        });
    });
}
```

### 9.2 Change Feed → Spark / KQL Materialization

For aggregations across partitions or complex business logic, use the **Cosmos DB
change feed** consumed by Azure Functions, Spark Structured Streaming, or **Fabric
mirroring + Spark notebook** to maintain materialized aggregates.

```python
# Fabric Spark notebook — read mirrored Cosmos data, recompute hourly aggregate
events = spark.read.format("delta").table("telemetry_mirror.events")
agg = (events
    .groupBy(F.window("ts", "1 hour"), "deviceId")
    .agg(F.count("*").alias("event_count"), F.avg("temperature").alias("avg_temp"))
    .select("window.start", "deviceId", "event_count", "avg_temp"))
agg.write.format("delta").mode("overwrite").saveAsTable("telemetry_lakehouse.events_agg_hourly")
```

---

## 10. Security Mapping

### 10.1 Authentication

| HDI HBase | Cosmos DB |
|-----------|-----------|
| LDAP (Ambari) | **Entra ID** (recommended) — `DefaultAzureCredential` |
| Kerberos / ESP (`GSSAPI`) | **Entra ID** + Cosmos RBAC roles |
| HBase REST gateway with anon access | Always require auth — **Entra ID** or **account key** (avoid keys for new) |
| Static keytab for service accounts | Workload identity / managed identity |

```bash
# Grant a managed identity Cosmos DB Data Contributor on a database
az cosmosdb sql role assignment create \
  --account-name my-cosmos -g cosmos-rg \
  --role-definition-name "Cosmos DB Built-in Data Contributor" \
  --principal-id $MI_OID \
  --scope "/dbs/telemetry"

# Disable account-key auth entirely (best practice for new deployments)
az cosmosdb update -n my-cosmos -g cosmos-rg --disable-key-based-metadata-write-access true
# (Combine with `--default-identity` and rotate apps to AAD before disabling keys for data plane.)
```

### 10.2 Authorization — Ranger HBase → Cosmos RBAC

| Ranger HBase Policy | Cosmos DB Equivalent |
|---------------------|----------------------|
| Table-level READ | `Cosmos DB Built-in Data Reader` on container scope |
| Table-level WRITE | `Cosmos DB Built-in Data Contributor` on container scope |
| Column-family-level allow/deny | **Not supported** in Cosmos — denormalize into separate containers OR enforce in app layer |
| Cell-level (visibility labels) | Not supported — enforce in app or use separate containers per visibility tier |
| Audit logs | Cosmos diagnostic logs → Log Analytics + Activity Log |

### 10.3 Network

| HDI HBase | Cosmos DB |
|-----------|-----------|
| VNet-injected cluster | **Private Endpoint** (recommended) |
| NSG on RegionServer subnet | NSG on PE subnet |
| Public endpoint | **Public endpoint with IP firewall rules** (allowlist) |
| ExpressRoute → HBase | ExpressRoute → Cosmos PE (same model) |

---

## 11. Cutover Strategy

### 11.1 Strangler-Fig + Dual-Write

```
Phase 0  : Inventory + categorize tables by complexity (1-2 weeks)
Phase 1  : Provision Cosmos DB account + containers + indexing policies (1 week)
Phase 2  : Bulk load history (snapshot → Spark → Cosmos), validate counts (1-3 weeks)
Phase 3  : Enable Fabric Mirroring; rebuild analytics dashboards on mirrored data (1-2 weeks)
Phase 4  : Dual-write from app layer to BOTH HBase and Cosmos (2-4 weeks observation)
Phase 5  : Cut READS to Cosmos one consumer at a time; monitor RU/throttling (2-4 weeks)
Phase 6  : Stop dual-write; HBase becomes read-only fallback for 30 days
Phase 7  : Decommission HDI HBase cluster
```

### 11.2 Dual-Write Pattern (Application Layer)

```python
# Wrapper that writes to HBase (legacy) and Cosmos (new) in parallel
import asyncio
from azure.cosmos.aio import CosmosClient

class DualWriter:
    def __init__(self, hbase_client, cosmos_container):
        self.hb, self.co = hbase_client, cosmos_container

    async def put_event(self, device_id, ts, payload):
        # Run both in parallel; treat HBase as canonical until cutover
        async def _hb():
            self.hb.table('events').put(
                f'{device_id}|{ts}'.encode(),
                {b'cf:data': json.dumps(payload).encode()})
        async def _co():
            await self.co.upsert_item({
                "id": str(ts), "deviceId": device_id, "data": payload})
        results = await asyncio.gather(_hb(), _co(), return_exceptions=True)

        # Log Cosmos failures but DON'T fail the request — HBase is canonical
        if isinstance(results[1], Exception):
            log.error(f"Cosmos dual-write failed: {results[1]}")
        # During Phase 6, swap: Cosmos is canonical, HBase is best-effort
```

### 11.3 Dual-Read Validation

```python
# Sample-based: pick 1000 random row keys, fetch from both, diff
import random
sample_keys = [...]   # extract from HBase scan or random sample
diffs = []
for k in sample_keys:
    hb_doc = hbase_to_dict(hbase_table.row(k.encode()))
    co_doc = cosmos.read_item(item=k.split("|",1)[1], partition_key=k.split("|",1)[0])
    if normalize(hb_doc) != normalize(co_doc):
        diffs.append((k, hb_doc, co_doc))
print(f"{len(diffs)}/{len(sample_keys)} differ")
```

---

## 12. Validation & Acceptance

```yaml
migration_pass_criteria:
  data_correctness:
    - per_table_row_count_delta_pct: < 0.01
    - sample_content_match_pct: > 99.9
    - dual_write_diff_count: 0     # for ≥7 consecutive days before cutover
  performance:
    - p99_point_read_ms: < 20      # Cosmos in same region
    - p99_query_ms: < 100
    - throttle_429_pct: < 0.1
  reliability:
    - cosmos_availability_7d_pct: > 99.99
    - mirroring_lag_p95_seconds: < 30
  security:
    - all_clients_use_entra_id: true
    - account_keys_in_keyvault_only: true
    - private_endpoint_enabled: true
  cost:
    - monthly_ru_consumption_pct_of_provisioned: 50-70%   # leave headroom
```

---

## 13. Cost Re-Modeling

| HDI HBase Cost Driver | Cosmos DB Equivalent | Optimization Tip |
|-----------------------|----------------------|------------------|
| Master VMs (D4 × 2 × 24 × 30 ≈ $580/mo) | None — managed | Pure savings |
| RegionServer VMs (D4 × 4 × 24 × 30 ≈ $1160/mo) | Provisioned RU/s OR Autoscale OR Serverless | Right-size after pilot; use Autoscale for spiky |
| ZooKeeper VMs (~$220/mo) | None | Pure savings |
| HDFS storage (replication × 3) | Cosmos transactional storage ($0.25/GB/mo) + Mirror to OneLake (cheap) | Use TTL to auto-expire |
| AAD DS (~$110/mo for ESP) | None — Entra ID | Pure savings |
| Spark cluster for analytics on HBase | Fabric Spark CU (only during runs) + mirrored Cosmos data | No HBase scan cost on Cosmos |
| Phoenix indexes (HBase storage overhead) | Cosmos indexing (default all paths, tune to reduce write RU) | Exclude high-volume / unused paths |

**Sizing example**: An HDI HBase cluster of 2 master + 4 RegionServer D4 VMs (~$2,000/mo
infra) handling 5K reads/s + 2K writes/s on 500 GB → roughly **30,000 RU/s autoscale**
+ **500 GB storage** ≈ **$1,750/mo** (East US list prices, autoscale 50% utilized).
Add **~$200/mo** for Fabric mirroring overhead. Net break-even or savings, with much
lower ops burden.

---

## 14. Pitfalls & Lessons Learned

1. **HBase row key ≠ Cosmos partition key**. Don't blindly map. The partition key
   needs **high cardinality + even distribution**; the HBase salt prefix is often
   redundant in Cosmos. Re-design from access patterns, not from the legacy key.

2. **Column families are not first-class in Cosmos NoSQL**. Nest them as JSON sub-objects
   for clarity, but plan for flat columns when querying — `WHERE c.cf.temperature > 30`
   works but each path level adds RU.

3. **Cosmos writes are expensive when over-indexed**. By default Cosmos indexes every
   property. For write-heavy workloads, **exclude unused paths** from the index policy
   to cut write RU by 30-60%.

4. **Cross-partition queries fan out and burn RUs**. Always include the partition key
   in `WHERE` for hot-path queries. Use **Fabric mirroring** for legitimate
   cross-partition analytical workloads.

5. **No JOINs across containers in Cosmos NoSQL**. If your Phoenix workload depended
   on JOIN, either denormalize at write time or run analytics over **Fabric mirrored
   data** in Lakehouse SQL endpoint.

6. **Partition key is immutable**. Choosing wrong means a full re-export to a new
   container. Pilot the access pattern with realistic data first.

7. **Coprocessors → not 1:1**. Stored procedures only run within one partition. For
   cross-partition logic, use Change Feed + Functions/Spark — fundamentally different
   model than HBase server-side hooks.

8. **HBase MVCC versions don't exist in Cosmos**. Cosmos keeps only the latest version.
   If you used `VERSIONS => 5`, replicate via Change Feed → append-only history table,
   or include version in `id`.

9. **MOB (Medium Object) columns**. HBase MOB stores larger cells off the regular WAL.
   Cosmos has a **2 MB item size limit** — store larger blobs in **Azure Blob Storage**
   and keep only the URL in the Cosmos document.

10. **Phoenix secondary indexes are different**. Cosmos auto-indexes all paths by
    default. Drop Phoenix secondary index DDL; tune Cosmos indexing policy instead.
    For range/composite queries, add **composite indexes** explicitly.

11. **Bulk migration spikes RUs**. Temporarily raise RU/s to 5-10× steady-state during
    backfill, then scale down. Bulk import in serverless mode will hit the 5K RU/s
    burst ceiling and slow significantly.

12. **Fabric mirroring lag is not instant**. ~1-30s typical; not suitable as a primary
    read path for sub-second transactional reads. Use it for analytics/BI only.

13. **Cassandra API ≠ pure Cassandra**. Cosmos for Cassandra implements CQL but not
    every feature (no `BATCH LOG`, limited materialized views, Lightweight Transactions
    have caveats). Test compatibility with your existing Cassandra-style client code.

14. **Cassandra API uses port 10350, not 9042**. Update connection strings; force TLS;
    auth is the account name + primary key (or AAD token).

15. **Account-key leakage**. Keys grant full access. New deployments should use
    **Entra ID + RBAC** exclusively and disable keys after migration.

16. **HBase "column versions over time" use case**. Cosmos doesn't store version
    history. Replace with: (a) include version in `id`, (b) write to append-only
    history container via change feed, (c) use Cosmos point-in-time restore (last
    30 days only).

17. **Shared throughput vs dedicated**. Sharing 400 RU/s across many small containers
    saves money in dev; in production, give heavy containers dedicated RU/s to avoid
    one container starving others.

18. **`scan` semantics differ**. HBase scans iterate by row key range and stream.
    Cosmos cross-partition queries fan out and may hit timeouts on big tables. Drive
    bulk reads via **Spark connector** in batched parallel mode, not single-client scans.

19. **Cosmos stored procedures have time and resource quotas**. ~5s execution limit
    and bounded memory. Long aggregations belong in Change Feed → Spark/KQL, not
    stored procs.

20. **HBase MAJOR_COMPACT cron jobs disappear**. Cosmos manages physical compaction
    internally. Remove your scheduled compaction code/scripts.

---

## 15. Quick-Reference Migration Checklist

```
[ ] Inventory HDI HBase tables, CFs, sizes, hot regions, coprocessors
[ ] Inventory clients: Java, Python (happybase), REST, Thrift, Phoenix JDBC, Spark-on-HBase
[ ] Categorize each table: target API (NoSQL vs Cassandra) + complexity
[ ] Pilot RU sizing: load 10% of data, measure x-ms-request-charge under realistic load
[ ] Provision Cosmos account (start with Autoscale 4K-50K RU/s + Serverless dev account)
[ ] Provision per-table containers with chosen partition keys + indexing policies
[ ] Set up Entra ID auth + RBAC role assignments + Private Endpoint
[ ] Bulk-load history: snapshot → Spark → Cosmos (batch in parallel, scale RU temporarily)
[ ] Enable Fabric Mirroring → mirror containers to OneLake
[ ] Rebuild BI/analytics on mirrored data (Direct Lake / KQL / Lakehouse SQL endpoint)
[ ] Translate Phoenix DDL/DML → Cosmos SDK/CQL (case-by-case; see §6)
[ ] Rewrite client SDKs (happybase → azure-cosmos, HBase Java → Cosmos Java, etc.)
[ ] Convert coprocessors → stored procedures (single-partition) OR Change Feed materialization
[ ] Implement dual-write from app layer (HBase + Cosmos in parallel)
[ ] Run dual-read validation for ≥7 days, target diff < 0.01%
[ ] Cut read traffic to Cosmos consumer-by-consumer; monitor 429s
[ ] Stop dual-write; HBase read-only fallback for 30 days
[ ] Decommission HDI HBase cluster (keep snapshot for 30+ days)
[ ] Right-size Cosmos RU/s based on actual consumption
[ ] Set up Cosmos diagnostic alerts (429 throttling, hot partitions, NormalizedRUConsumption > 80%)
[ ] Document new runbook + on-call procedures
```

---

## 16. Useful Commands & Snippets

```bash
# Cosmos: create account + database + container with autoscale
az cosmosdb create -n my-cosmos -g cosmos-rg --kind GlobalDocumentDB \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --default-consistency-level Session
az cosmosdb sql database create -a my-cosmos -g cosmos-rg -n telemetry
az cosmosdb sql container create -a my-cosmos -g cosmos-rg -d telemetry \
  -n devices --partition-key-path "/deviceId" --max-throughput 10000

# Top expensive recent queries (Log Analytics KQL)
AzureDiagnostics
| where Category == "QueryRuntimeStatistics" and TimeGenerated > ago(1h)
| project TimeGenerated, collectionName_s, queryText_s, requestCharge_s
| order by todouble(requestCharge_s) desc | take 20

# Cosmos: check throttle (429) rate per container
AzureDiagnostics
| where Category == "DataPlaneRequests" and statusCode_s == "429"
| summarize count() by bin(TimeGenerated, 5m), collectionName_s

# Hot partitions
AzureDiagnostics
| where Category == "PartitionKeyStatistics"
| project partitionKey_s, sizeKb_d | order by sizeKb_d desc

# HBase: list snapshots + sizes (run on HDI)
echo "list_snapshots" | hbase shell -n
hdfs dfs -du -s -h /hbase/.hbase-snapshot/

# Bulk-import RU charge log (Python)
resp = container.upsert_item(item)
print("RU:", container.client_connection.last_response_headers["x-ms-request-charge"])
```

---

## Rules

1. **Never lift the row-key 1:1** — redesign partition key from access patterns, drop salt prefixes, ensure high cardinality + even distribution.
2. **Pilot RU sizing before committing** — paper estimates are 30-50% wrong; measure `x-ms-request-charge` on real workload.
3. **Always pair Cosmos OLTP with Fabric Mirroring** when there's downstream analytics — never run scans against the OLTP store.
4. **Use Entra ID + RBAC, not account keys** for all new deployments — disable keys after migration.
5. **Tune indexing policy** — exclude unused paths to cut write RU 30-60% on write-heavy workloads.
6. **Always include partition key in queries** — cross-partition queries fan out and burn RUs.
7. **Replace coprocessors deliberately** — stored procs for single-partition atomic logic, Change Feed for everything else.
8. **Handle HBase MVCC explicitly** — Cosmos keeps only latest; if you need history, append-only history container via change feed.
9. **Bulk-load with elevated RU/s** — temporarily raise RU/s 5-10× during backfill, scale back after.
10. **Keep HDI HBase cluster + snapshot for 30 days post-cutover** — instant rollback path.
