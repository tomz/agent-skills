---
name: azure-synapse
description: Azure Synapse Analytics — dedicated SQL pools, serverless SQL, Spark pools, pipelines, Synapse Link, security, and monitoring
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

# Azure Synapse Analytics

Azure Synapse is an enterprise analytics service combining dedicated SQL pools
(formerly SQL Data Warehouse), a serverless SQL pool, Apache Spark pools, and
Data Factory-compatible pipelines — all within a single workspace with shared security,
monitoring, and a unified Studio UI.

---

## Workspace Setup

### az CLI Provisioning
```bash
# Create resource group and storage (ADLS Gen2)
az group create --name synapse-rg --location eastus

az storage account create \
  --name mysynapsestorage \
  --resource-group synapse-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true   # Required for ADLS Gen2

# Create Synapse workspace
az synapse workspace create \
  --name my-synapse-ws \
  --resource-group synapse-rg \
  --storage-account mysynapsestorage \
  --file-system synapsefs \         # Container in ADLS Gen2
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password "YourP@ssw0rd!" \
  --location eastus

# Open firewall for your IP (dev/test only)
az synapse workspace firewall-rule create \
  --name AllowMyIP \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --start-ip-address $(curl -s ifconfig.me) \
  --end-ip-address $(curl -s ifconfig.me)

# Enable managed VNet at creation (cannot change after)
az synapse workspace create \
  ... \
  --enable-managed-virtual-network   # Adds --managed-virtual-network flag
```

### Managed VNet & Managed Private Endpoints
```bash
# Create a managed private endpoint to your ADLS Gen2
az synapse managed-private-endpoints create \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --pe-name my-adls-mpe \
  --file @mpe.json

# mpe.json:
# {
#   "name": "my-adls-mpe",
#   "properties": {
#     "privateLinkResourceId": "/subscriptions/.../storageAccounts/mydata",
#     "groupId": "dfs"
#   }
# }

# List managed private endpoints
az synapse managed-private-endpoints list \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg
```

**Key**: When managed VNet is enabled, all Spark and pipeline activities run inside the
managed VNet. Data exfiltration protection blocks outbound traffic not through approved
managed private endpoints.

---

## Dedicated SQL Pool

A dedicated SQL pool is a provisioned MPP (Massively Parallel Processing) cluster with
60 fixed compute nodes that distributes data across **distributions**.

### DWU Sizing
| DWU       | Compute Nodes | Distributions/Node | Approx Memory |
|-----------|--------------|-------------------|---------------|
| DW100c    | 1            | 60                | 60 GB         |
| DW500c    | 5            | 12                | 300 GB        |
| DW1000c   | 10           | 6                 | 600 GB        |
| DW5000c   | 50           | ~1                | 3 TB          |
| DW30000c  | 60           | 1                 | 18 TB         |

```bash
# Create dedicated SQL pool
az synapse sql pool create \
  --name MySQLPool \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --performance-level DW500c

# Pause (stops compute billing — storage still charged)
az synapse sql pool pause \
  --name MySQLPool \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg

# Resume
az synapse sql pool resume \
  --name MySQLPool \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg

# Scale DWU (can do while running — brief interruption)
az synapse sql pool update \
  --name MySQLPool \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --performance-level DW1000c
```

### Table Distributions
Every table must be distributed. Choose incorrectly and you'll pay in shuffle costs.

```sql
-- Hash distribution (best for large fact tables with even key distribution)
CREATE TABLE dbo.FactSales (
    SaleId       BIGINT       NOT NULL,
    CustomerId   INT          NOT NULL,
    ProductId    INT          NOT NULL,
    SaleDate     DATE         NOT NULL,
    Amount       DECIMAL(18,2)
)
WITH (
    DISTRIBUTION = HASH(CustomerId),   -- join key to customers
    CLUSTERED COLUMNSTORE INDEX         -- CCI: best compression + analytics perf
);

-- Round-robin (best for staging tables, unknown join patterns)
CREATE TABLE dbo.Staging_Orders (...)
WITH (DISTRIBUTION = ROUND_ROBIN, HEAP);

-- Replicated (best for small dimension tables < ~2 GB)
CREATE TABLE dbo.DimProduct (
    ProductId   INT           NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Category    NVARCHAR(100)
)
WITH (
    DISTRIBUTION = REPLICATE,
    CLUSTERED COLUMNSTORE INDEX
);

-- Check distribution skew
SELECT
    pnp.pdw_node_id,
    pnp.distribution_id,
    SUM(pnp.used_page_count) * 8 / 1024.0 AS used_space_MB
FROM sys.pdw_nodes_db_partition_stats AS pnp
JOIN sys.pdw_distributions AS pd ON pnp.distribution_id = pd.distribution_id
WHERE pnp.object_id = OBJECT_ID('dbo.FactSales')
GROUP BY pnp.pdw_node_id, pnp.distribution_id
ORDER BY used_space_MB DESC;
```

### Indexing
```sql
-- Clustered Columnstore Index (CCI): default recommendation for analytics
-- Best compression (5-10x), best scan performance for large tables

-- Clustered Rowstore Index: only when heavy point lookups or small tables
CREATE TABLE dbo.LookupTable (
    Id   INT NOT NULL PRIMARY KEY,
    Val  NVARCHAR(200)
)
WITH (
    DISTRIBUTION = REPLICATE,
    CLUSTERED INDEX (Id)    -- rowstore for PK lookups
);

-- Heap: staging / ELT intermediary tables (no index overhead on insert)
WITH (DISTRIBUTION = ROUND_ROBIN, HEAP)

-- Rebuild fragmented CCI
ALTER INDEX ALL ON dbo.FactSales REBUILD;

-- Check CCI health (row group quality)
SELECT
    OBJECT_NAME(rg.object_id) AS table_name,
    rg.state_desc,
    COUNT(*) AS row_group_count,
    SUM(rg.total_rows) AS total_rows,
    SUM(rg.deleted_rows) AS deleted_rows
FROM sys.pdw_nodes_column_store_row_groups rg
GROUP BY rg.object_id, rg.state_desc
ORDER BY table_name, state_desc;
```

### Partitioning
```sql
-- Partition by date for large fact tables (improves partition elimination)
CREATE TABLE dbo.FactSales (
    SaleId     BIGINT,
    SaleDate   DATE,
    Amount     DECIMAL(18,2)
)
WITH (
    DISTRIBUTION = HASH(CustomerId),
    CLUSTERED COLUMNSTORE INDEX,
    PARTITION (SaleDate RANGE RIGHT FOR VALUES (
        '2023-01-01', '2023-04-01', '2023-07-01', '2023-10-01',
        '2024-01-01', '2024-04-01', '2024-07-01', '2024-10-01'
    ))
);

-- Switch in a new partition (zero-copy when both tables same structure)
ALTER TABLE dbo.FactSales_Staging
    SWITCH PARTITION 1 TO dbo.FactSales PARTITION 5;
```

### Workload Management
```sql
-- Create workload group for BI users (isolated resources)
CREATE WORKLOAD GROUP BiUsers
WITH (
    MIN_PERCENTAGE_RESOURCE = 20,   -- always reserves 20% of DWUs
    CAP_PERCENTAGE_RESOURCE = 50,   -- cannot exceed 50%
    REQUEST_MIN_RESOURCE_GRANT_PERCENT = 5
);

-- Create classifier to route users to group
CREATE WORKLOAD CLASSIFIER BiClassifier
WITH (
    WORKLOAD_GROUP = 'BiUsers',
    MEMBERNAME = 'bi_login',
    IMPORTANCE = NORMAL
);

-- Monitor active requests
SELECT * FROM sys.dm_pdw_exec_requests
WHERE status = 'Running'
ORDER BY submit_time;

-- Find waiting requests and why
SELECT
    r.request_id,
    r.status,
    r.submit_time,
    w.type AS wait_type,
    w.object_type,
    w.object_name
FROM sys.dm_pdw_exec_requests r
JOIN sys.dm_pdw_waits w ON r.request_id = w.request_id
WHERE r.status = 'Suspended';
```

**Gotcha — Statistics**: Dedicated pool does NOT auto-create or auto-update statistics.
Missing or stale statistics cause terrible query plans. Create after load:
```sql
CREATE STATISTICS stats_FactSales_CustomerId ON dbo.FactSales (CustomerId);
-- Or auto-create for all columns (use sp_create_stats or a maintenance job):
EXEC sp_create_stats;
```

---

## Serverless SQL Pool

The serverless pool is **always available**, billed per TB of data scanned, and needs
no provisioning. No CCI, no distributions — it's a distributed query engine over ADLS.

### OPENROWSET (Ad-Hoc Queries)
```sql
-- Query Parquet files
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://myaccount.dfs.core.windows.net/mycontainer/sales/2024/**',
    FORMAT = 'PARQUET'
) AS r;

-- Query CSV with explicit schema
SELECT *
FROM OPENROWSET(
    BULK 'https://myaccount.dfs.core.windows.net/mycontainer/raw/orders.csv',
    FORMAT = 'CSV',
    PARSER_VERSION = '2.0',
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n'
) WITH (
    order_id    INT         1,
    customer_id INT         2,
    order_date  DATE        3,
    amount      DECIMAL(18,2) 4
) AS r;

-- Query Delta Lake (reads _delta_log for schema + latest snapshot)
SELECT *
FROM OPENROWSET(
    BULK 'https://myaccount.dfs.core.windows.net/mycontainer/delta/orders/',
    FORMAT = 'DELTA'
) AS r
WHERE order_date >= '2024-01-01';
```

### Credential Management
```sql
-- Create credential for managed identity access (workspace MSI)
CREATE DATABASE SCOPED CREDENTIAL WorkspaceIdentity
WITH IDENTITY = 'Managed Identity';

-- Create credential with SAS token
CREATE DATABASE SCOPED CREDENTIAL SasCredential
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = 'sv=2020-08-04&ss=b&srt=co&sp=r...';  -- SAS token (no leading ?)

-- Create data source referencing credential
CREATE EXTERNAL DATA SOURCE MyADLS
WITH (
    LOCATION = 'https://myaccount.dfs.core.windows.net/mycontainer',
    CREDENTIAL = WorkspaceIdentity
);

-- Use data source in queries
SELECT * FROM OPENROWSET(
    BULK 'sales/2024/',
    DATA_SOURCE = 'MyADLS',
    FORMAT = 'PARQUET'
) AS r;
```

### CETAS (CREATE EXTERNAL TABLE AS SELECT)
CETAS is the serverless pool's primary way to persist query results:
```sql
-- Create external file format
CREATE EXTERNAL FILE FORMAT ParquetSnappy
WITH (FORMAT_TYPE = PARQUET, DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec');

-- Run transformation and save result as Parquet
CREATE EXTERNAL TABLE dbo.monthly_sales_summary
WITH (
    LOCATION = 'aggregates/monthly_sales/',
    DATA_SOURCE = MyADLS,
    FILE_FORMAT = ParquetSnappy
)
AS
SELECT
    FORMAT(order_date, 'yyyy-MM') AS month,
    region,
    SUM(amount) AS total_sales,
    COUNT(*) AS order_count
FROM OPENROWSET(
    BULK 'sales/2024/',
    DATA_SOURCE = 'MyADLS',
    FORMAT = 'PARQUET'
) AS r
GROUP BY FORMAT(order_date, 'yyyy-MM'), region;
```

**Gotcha — CETAS drops old files**: Re-running CETAS on the same LOCATION does NOT
truncate — it will mix new and old Parquet files. Drop and recreate the external table,
or use a new LOCATION with a timestamp/version prefix.

**Gotcha — Serverless charges per TB scanned**: A full table scan of a large Parquet
dataset is expensive. Always add partition filter predicates and ensure Parquet files
use efficient column pruning. Delta format with partition pruning drastically reduces cost.

---

## Spark Pools

```bash
# Create Spark pool
az synapse spark pool create \
  --name mysparkpool \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --spark-version 3.4 \
  --node-count 3 \
  --node-size Medium \
  --enable-auto-scale true \
  --min-node-count 3 \
  --max-node-count 10 \
  --delay 15              # autoscale delay minutes
```

### Library Management
```bash
# Upload requirements.txt to workspace storage, then:
az synapse spark pool update \
  --name mysparkpool \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --library-requirements /tmp/requirements.txt
```

### Spark-SQL Interop
```python
# In a Synapse notebook (PySpark)

# Read from ADLS Gen2 (linked service handles auth)
df = spark.read.parquet("abfss://mycontainer@myaccount.dfs.core.windows.net/sales/")

# Read from dedicated SQL pool (Synapse connector — parallel read via PolyBase)
df = spark.read \
    .format("com.microsoft.azure.synapse.spark") \
    .option("url", "jdbc:sqlserver://my-synapse-ws.sql.azuresynapse.net:1433;database=MySQLPool") \
    .option("dbTable", "dbo.FactSales") \
    .option("tempDir", "abfss://mycontainer@myaccount.dfs.core.windows.net/spark-tmp/") \
    .load()

# Write to dedicated SQL pool
df.write \
    .format("com.microsoft.azure.synapse.spark") \
    .option("url", "jdbc:sqlserver://my-synapse-ws.sql.azuresynapse.net:1433;database=MySQLPool") \
    .option("dbTable", "dbo.StagingTable") \
    .option("tempDir", "abfss://mycontainer@myaccount.dfs.core.windows.net/spark-tmp/") \
    .mode("overwrite") \
    .save()

# Delta Lake in Spark pool
df.write.format("delta").mode("overwrite").save(
    "abfss://mycontainer@myaccount.dfs.core.windows.net/delta/orders/"
)
spark.sql("CREATE TABLE orders USING DELTA LOCATION 'abfss://...'")
```

---

## Pipelines

Synapse Pipelines are ADF-compatible (same JSON schema, same activity types). Most
ADF knowledge transfers directly.

```bash
# List pipelines
az synapse pipeline list \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg

# Trigger a pipeline run
az synapse pipeline create-run \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --name MyIngestionPipeline \
  --parameters '{"date": "2024-01-15"}'

# Monitor run status
az synapse pipeline-run query-by-workspace \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --last-updated-after 2024-01-15T00:00:00Z \
  --last-updated-before 2024-01-16T00:00:00Z
```

### Integration Runtimes
- **AutoResolveIntegrationRuntime**: serverless, managed by Synapse (default for cloud activities)
- **Self-Hosted IR**: deploy on-premises or in a VM for on-prem data sources
- **Managed IR (VNet)**: only available when workspace managed VNet is enabled; runs inside managed VNet

---

## Synapse Link

Synapse Link creates a near-real-time analytical replica of operational data without
impacting the transactional system.

### Synapse Link for Azure Cosmos DB
```sql
-- After enabling Synapse Link on Cosmos DB container:
-- Query Cosmos DB analytical store from serverless pool

SELECT TOP 10 *
FROM OPENROWSET(
    'CosmosDB',
    'Account=mycosmosdb;Database=mydb;Key=<key>',
    orders
) WITH (
    id          VARCHAR(50)   '$.id',
    customerId  INT           '$.customerId',
    amount      FLOAT         '$.amount',
    orderDate   VARCHAR(30)   '$.orderDate'
) AS orders;
```

### Synapse Link for SQL Server / Azure SQL
Replicates tables via Change Feed to dedicated or serverless pool.
Provisioned via the Azure Portal or az cli:
```bash
az synapse link-connection create \
  --workspace-name my-synapse-ws \
  --resource-group synapse-rg \
  --name sql-link \
  --file @link-connection.json
```

---

## Security

### Column-Level Security
```sql
-- Grant column-level SELECT (dedicated pool)
GRANT SELECT ON dbo.Customers (customer_id, customer_name) TO [analyst_login];
-- Deny sensitive columns
DENY SELECT ON dbo.Customers (ssn, credit_card) TO [analyst_login];
```

### Row-Level Security
```sql
-- Create predicate function
CREATE FUNCTION dbo.fn_security_predicate(@region NVARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @region = SESSION_CONTEXT(N'region')
   OR IS_MEMBER('db_owner') = 1;

-- Apply security policy
CREATE SECURITY POLICY RegionFilter
ADD FILTER PREDICATE dbo.fn_security_predicate(region) ON dbo.FactSales,
ADD BLOCK  PREDICATE dbo.fn_security_predicate(region) ON dbo.FactSales AFTER INSERT
WITH (STATE = ON);

-- Set session context (application must set this per login)
EXEC sp_set_session_context N'region', N'West';
```

### Dynamic Data Masking
```sql
-- Add masking to sensitive columns
ALTER TABLE dbo.Customers
ALTER COLUMN EmailAddress ADD MASKED WITH (FUNCTION = 'email()');

ALTER TABLE dbo.Customers
ALTER COLUMN CreditCard ADD MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)');

-- Grant UNMASK to privileged role
GRANT UNMASK TO [db_datareader];
```

---

## Monitoring

### DMVs for Dedicated SQL Pool
```sql
-- Active queries with elapsed time
SELECT
    r.request_id,
    r.status,
    r.submit_time,
    DATEDIFF(SECOND, r.submit_time, GETDATE()) AS elapsed_sec,
    r.command
FROM sys.dm_pdw_exec_requests r
WHERE r.status NOT IN ('Completed', 'Failed', 'Cancelled')
ORDER BY submit_time;

-- Query steps (which step is running / slowest)
SELECT
    r.request_id,
    rs.step_index,
    rs.operation_type,
    rs.status,
    rs.total_elapsed_time / 1000 AS elapsed_sec,
    rs.row_count
FROM sys.dm_pdw_exec_requests r
JOIN sys.dm_pdw_request_steps rs ON r.request_id = rs.request_id
WHERE r.request_id = 'QID12345'
ORDER BY rs.step_index;

-- Data movement waits (common bottleneck: shuffle/broadcast DMS operations)
SELECT
    rs.request_id,
    rs.step_index,
    rs.operation_type,
    rs.total_elapsed_time / 1000 AS step_sec,
    rs.row_count,
    rs.status
FROM sys.dm_pdw_request_steps rs
WHERE rs.operation_type IN ('BroadcastMoveOperation', 'ShuffleMoveOperation', 'TrimMoveOperation')
  AND rs.total_elapsed_time > 5000
ORDER BY rs.total_elapsed_time DESC;
```

### Spark Application Monitoring
```bash
# List Spark applications
az synapse spark job list \
  --workspace-name my-synapse-ws \
  --spark-pool-name mysparkpool \
  --resource-group synapse-rg

# Get Spark application logs
az synapse spark job show \
  --workspace-name my-synapse-ws \
  --spark-pool-name mysparkpool \
  --resource-group synapse-rg \
  --livy-id <livy-id>
```

---

## Gotchas & Operational Tips

1. **Dedicated pool storage costs when paused**: Compute stops but you pay for storage
   (DWU-hours × storage overhead). Data stored as proprietary distributed format — you
   cannot access it directly from ADLS while the pool is paused.

2. **DWU scaling takes time**: Scaling up from DW500c to DW2000c takes 5-10 minutes
   and causes a brief connection interruption. Plan maintenance windows.

3. **Distribution skew kills performance**: If 90% of your hash-distributed table rows
   land on 5 of 60 distributions, queries become single-node bottlenecks. Verify with
   `sys.pdw_nodes_db_partition_stats`. Choose distribution keys carefully.

4. **Statistics must be maintained manually**: Run `EXEC sp_update_stats` or schedule
   a weekly job. Stale stats after large loads cause optimizer to choose wrong join types.

5. **Serverless charges per TB scanned (not per query)**: A complex multi-join over
   10 TB of CSV files will cost ~$50 per run. Convert CSV to Parquet with appropriate
   partitioning to reduce by 10-50x. Use Delta for partition pruning.

6. **CETAS location collision**: Re-running CETAS to the same path does not overwrite
   — it appends new files alongside old ones. Always manage output locations explicitly.

7. **Managed VNet is all-or-nothing**: You cannot enable managed VNet after workspace
   creation. If you need private endpoints later, you must recreate the workspace.

8. **Spark pool cold start**: First Spark job on a cold pool takes 3-5 minutes. Set
   auto-pause delay to at least 30 minutes if you have interactive users.

9. **Synapse Link CDC lag**: Synapse Link for Cosmos DB typically has 2-5 minute lag.
   For SQL Server, initial full load can take hours for large tables. It is NOT a
   replacement for real-time streaming (use Event Hubs + Spark Streaming for that).

10. **Workspace managed identity (MSI)**: The workspace has a system-assigned managed
    identity that should be granted Storage Blob Data Contributor on your ADLS Gen2.
    Many "access denied" errors in pipelines are due to missing MSI role assignments.
