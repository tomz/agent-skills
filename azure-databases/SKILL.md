---
name: azure-databases
description: Azure managed databases — SQL Database, SQL Managed Instance, MySQL Flexible Server, PostgreSQL Flexible Server, MariaDB, SQL Server on VMs, database migration, elastic pools, Hyperscale, serverless compute
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Managed Databases

Comprehensive reference for all Azure managed relational database services. For
Cosmos DB (NoSQL, document, graph, table), see the dedicated **azure-cosmosdb** skill.

---

## Table of Contents

1. [Azure SQL Database](#azure-sql-database)
2. [Azure SQL Managed Instance](#azure-sql-managed-instance)
3. [Azure Database for PostgreSQL Flexible Server](#postgresql-flexible-server)
4. [Azure Database for MySQL Flexible Server](#mysql-flexible-server)
5. [Azure Database for MariaDB](#mariadb)
6. [SQL Server on Azure VMs](#sql-server-on-vms)
7. [Cross-Cutting Topics](#cross-cutting-topics)
8. [Infrastructure as Code](#infrastructure-as-code)
9. [Cost Optimization](#cost-optimization)

---

## Azure SQL Database

Fully managed PaaS SQL Server engine. Microsoft handles patching, backups, and HA.

### Purchasing Models

#### DTU Model (legacy, simpler)
Bundles compute + storage + I/O into a single unit. Good for predictable workloads.

| Tier     | DTUs      | Max storage | Use case                       |
|----------|-----------|-------------|--------------------------------|
| Basic    | 5         | 2 GB        | Dev/test, small apps           |
| Standard | 10–3000   | 1 TB        | General-purpose workloads      |
| Premium  | 125–4000  | 4 TB        | High I/O, mission-critical     |

#### vCore Model (recommended)
Independent compute and storage scaling. Required for Hyperscale and Serverless.

| Tier              | vCores  | Max storage | SLA     | Notes                               |
|-------------------|---------|-------------|---------|-------------------------------------|
| General Purpose   | 2–128   | 4 TB        | 99.99%  | Balanced; remote storage (Azure Premium) |
| Business Critical | 2–128   | 4 TB        | 99.99%  | Local SSD, built-in read replica    |
| Hyperscale        | 2–128   | 100 TB      | 99.99%  | Distributed storage; see below      |

### Serverless Compute Tier

Auto-scales compute and auto-pauses after inactivity. Only available on General Purpose vCore.

```bash
# Create serverless SQL Database
az sql db create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --compute-model Serverless \
  --edition GeneralPurpose \
  --family Gen5 \
  --min-capacity 0.5 \
  --capacity 4 \
  --auto-pause-delay 60   # minutes; -1 disables auto-pause

# Update auto-pause delay
az sql db update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --auto-pause-delay 120
```

Key serverless behaviours:
- Billing is per-second of active compute + always-on storage
- Cold start latency on resume: typically 30–60 seconds
- Not suitable for always-on or latency-sensitive workloads
- `min-capacity` can be as low as 0.5 vCores

### Elastic Pools

Share resources (eDTU or vCore pool) across multiple databases. Ideal when databases
have variable and non-overlapping peak times.

```bash
# Create vCore elastic pool
az sql elastic-pool create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name pool-main \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 8 \
  --db-min-capacity 0 \
  --db-max-capacity 4

# Add a database to the pool
az sql db create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name tenantdb1 \
  --elastic-pool pool-main

# Move existing database into pool
az sql db update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name existingdb \
  --elastic-pool pool-main

# List databases in a pool
az sql elastic-pool list-dbs \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name pool-main \
  --output table

# Show pool metrics
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Sql/servers/sql-prod-eastus/elasticPools/pool-main \
  --metric "dtu_consumption_percent" \
  --interval PT1H
```

### Hyperscale

Distributed storage architecture supporting up to 100 TB. Separate compute and page
server tiers allow near-instant scaling.

**Architecture:**
- Primary compute replica → Log Service → Page Servers (distributed storage)
- Reads served from page server cache; no I/O bottleneck on single storage
- Backups are near-instant (log-based, no full copy)
- Scale out read workloads with up to 30 named or anonymous read replicas

```bash
# Create Hyperscale database
az sql db create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name bigdb \
  --edition Hyperscale \
  --family Gen5 \
  --capacity 8 \
  --read-replicas 2       # high-availability replicas (0 or 2)

# Add named replica (independent read endpoint)
az sql db replica create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name bigdb \
  --partner-server sql-prod-westus \
  --partner-database bigdb-replica \
  --secondary-type Named

# Reverse migration: Hyperscale → General Purpose
# Only possible within 45 days of original migration TO Hyperscale
az sql db update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name bigdb \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 8
```

### Geo-Replication and Failover Groups

```bash
# Create active geo-replication secondary
az sql db replica create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --partner-server sql-prod-westus

# Create failover group (recommended over manual geo-replication)
az sql failover-group create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name fog-myapp \
  --partner-server sql-prod-westus \
  --failover-policy Automatic \
  --grace-period 1 \
  --add-db mydb

# Manual failover (e.g. for planned maintenance)
az sql failover-group set-primary \
  --resource-group rg-prod \
  --server sql-prod-westus \
  --name fog-myapp

# List failover groups
az sql failover-group list \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --output table
```

Failover group provides a **listener endpoint** independent of the server name:
- Read-write: `fog-myapp.database.windows.net`
- Read-only: `fog-myapp.secondary.database.windows.net`

### Server and Database Management

```bash
# Create logical server
az sql server create \
  --resource-group rg-prod \
  --name sql-prod-eastus \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "$(az keyvault secret show --vault-name kv-prod --name sql-password --query value -o tsv)" \
  --enable-public-network false

# Create database (vCore, General Purpose)
az sql db create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 4 \
  --zone-redundant true \
  --backup-storage-redundancy Geo

# Scale database
az sql db update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --capacity 8

# Copy database
az sql db copy \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --dest-resource-group rg-staging \
  --dest-server sql-staging-eastus \
  --dest-name mydb-copy

# Export to BACPAC
az sql db export \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --admin-user sqladmin \
  --admin-password "$SQL_PASSWORD" \
  --storage-key-type SharedAccessKey \
  --storage-key "$SAS_TOKEN" \
  --storage-uri "https://mystorage.blob.core.windows.net/backups/mydb.bacpac"

# List databases
az sql db list --resource-group rg-prod --server sql-prod-eastus --output table

# Show database details
az sql db show --resource-group rg-prod --server sql-prod-eastus --name mydb
```

### Security

#### Transparent Data Encryption (TDE)
Enabled by default. Customer-managed keys via Azure Key Vault:

```bash
az sql server tde-key set \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --server-key-type AzureKeyVault \
  --kid "https://kv-prod.vault.azure.net/keys/sql-tde-key/abc123"
```

#### Always Encrypted
Column-level encryption; plaintext never leaves the client. Keys stored in Key Vault
or Windows Certificate Store. Configure via SSMS or PowerShell — no CLI equivalent.

#### Dynamic Data Masking
```bash
# Add masking rule for email column
az sql db tde-key set ...   # (DDM is configured via portal or T-SQL)

-- T-SQL:
ALTER TABLE dbo.Customers
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
```

#### Auditing
```bash
az sql server audit-policy update \
  --resource-group rg-prod \
  --name sql-prod-eastus \
  --state Enabled \
  --storage-account mystorage \
  --retention-days 90

# Send audit logs to Log Analytics
az sql db audit-policy update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --state Enabled \
  --log-analytics-workspace-resource-id /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.OperationalInsights/workspaces/law-prod
```

#### Microsoft Defender for SQL
```bash
az sql server advanced-threat-protection-setting update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --state Enabled
```

#### Entra ID (Azure AD) Authentication
```bash
# Set Entra admin on server
az sql server ad-admin create \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --display-name "DBA-Group" \
  --object-id "$(az ad group show --group DBA-Group --query id -o tsv)"

# Disable password-based admin (Entra-only mode)
az sql server ad-only-auth enable \
  --resource-group rg-prod \
  --name sql-prod-eastus
```

Contained database users (no SQL logins required):
```sql
CREATE USER [user@domain.com] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [user@domain.com];
```

### Ledger Tables

Cryptographically verified, tamper-evident audit trail.

```sql
-- Append-only ledger table (inserts only, immutable)
CREATE TABLE dbo.Transactions
(
    TransactionId   BIGINT GENERATED ALWAYS AS ROW START,
    Amount          DECIMAL(18,2) NOT NULL,
    AccountId       INT NOT NULL
)
WITH (LEDGER = ON (APPEND_ONLY = ON));

-- Updatable ledger table (full history)
CREATE TABLE dbo.Customers
(
    CustomerId INT PRIMARY KEY,
    Name NVARCHAR(200)
)
WITH (LEDGER = ON);
```

### Temporal Tables

System-versioned tables with automatic history tracking.

```sql
CREATE TABLE dbo.Employee
(
    EmployeeId  INT PRIMARY KEY,
    Name        NVARCHAR(100),
    Salary      DECIMAL(18,2),
    ValidFrom   DATETIME2 GENERATED ALWAYS AS ROW START,
    ValidTo     DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.EmployeeHistory));

-- Query historical state
SELECT * FROM dbo.Employee
FOR SYSTEM_TIME AS OF '2025-01-01T00:00:00';
```

### Intelligent Performance

```bash
# Enable automatic tuning (CREATE INDEX, DROP INDEX, FORCE LAST GOOD PLAN)
az sql db update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --auto-tune-options '{"createIndex":"Auto","dropIndex":"Auto","forceLastGoodPlan":"Auto"}'

# Query Performance Insight — view in portal or via REST
# Query Store is enabled by default on Azure SQL Database
```

T-SQL to query Query Store:
```sql
-- Top 10 queries by total CPU
SELECT TOP 10
    q.query_id,
    qt.query_sql_text,
    SUM(rs.total_worker_time) AS total_cpu_us
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
GROUP BY q.query_id, qt.query_sql_text
ORDER BY total_cpu_us DESC;
```

---

## Azure SQL Managed Instance

Near 100% SQL Server compatibility in a fully managed PaaS offering. Ideal for
lift-and-shift migrations that require SQL Agent, CLR, linked servers, or cross-db queries.

### Key Differences from SQL Database

| Feature                  | SQL Database        | SQL Managed Instance  |
|--------------------------|---------------------|-----------------------|
| SQL Agent                | No                  | Yes                   |
| CLR                      | No                  | Yes                   |
| Cross-database queries   | No                  | Yes                   |
| Linked Servers           | No                  | Yes (limited)         |
| VNet native              | Optional            | Required              |
| Instance pools           | No                  | Yes                   |
| SSRS/SSAS                | No                  | No (use Azure-native) |

### Networking Requirements

SQL MI is VNet-native — it occupies a **dedicated subnet**. The subnet must:
- Be delegated to `Microsoft.Sql/managedInstances`
- Have a /27 or larger CIDR (minimum 32 addresses; /24 recommended for production)
- Have a route table and NSG attached per Microsoft's requirements
- Have no other resources in the subnet

```bash
# Create subnet for SQL MI
az network vnet subnet create \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name snet-sqlmi \
  --address-prefixes 10.0.1.0/24 \
  --delegations Microsoft.Sql/managedInstances

# Create managed instance (takes 4–6 hours for first deployment)
az sql mi create \
  --resource-group rg-prod \
  --name sqlmi-prod-eastus \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "$SQL_MI_PASSWORD" \
  --subnet /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-sqlmi \
  --capacity 8 \
  --storage-size 256 \
  --edition GeneralPurpose \
  --family Gen5 \
  --collation SQL_Latin1_General_CP1_CI_AS \
  --timezone-id "Eastern Standard Time"

# Scale vCores
az sql mi update \
  --resource-group rg-prod \
  --name sqlmi-prod-eastus \
  --capacity 16

# Create database on the instance
az sql midb create \
  --resource-group rg-prod \
  --managed-instance sqlmi-prod-eastus \
  --name mydb

# List managed instances
az sql mi list --resource-group rg-prod --output table
```

### Instance Pools

Pre-provision compute to host multiple small managed instances, reducing per-instance
deployment time (minutes vs hours).

```bash
# Create instance pool
az sql instance-pool create \
  --resource-group rg-prod \
  --name pool-mi-prod \
  --location eastus \
  --capacity 16 \
  --subnet /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-sqlmi-pool \
  --edition GeneralPurpose \
  --family Gen5

# Create MI inside pool
az sql mi create \
  --resource-group rg-prod \
  --name sqlmi-small-1 \
  --instance-pool-name pool-mi-prod \
  --capacity 4 \
  --storage-size 64 \
  --edition GeneralPurpose \
  --family Gen5 \
  --admin-user sqladmin \
  --admin-password "$SQL_MI_PASSWORD" \
  --subnet /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-sqlmi-pool
```

### Link Feature (Hybrid with SQL Server)

Replicates databases from on-premises SQL Server 2016+ to SQL MI in near real-time
using distributed availability groups.

```bash
# List links on managed instance
az sql mi link list \
  --resource-group rg-prod \
  --instance-name sqlmi-prod-eastus

# Failover link (make SQL MI primary — for migration cutover)
az sql mi link failover \
  --resource-group rg-prod \
  --instance-name sqlmi-prod-eastus \
  --link-name mylink \
  --failover-type PlannedFailover
```

### Migration from SQL Server

**Database Migration Service (DMS):**
```bash
# Create DMS instance
az dms create \
  --resource-group rg-prod \
  --name dms-prod \
  --location eastus \
  --sku-name Premium_4vCores \
  --subnet /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-dms

# Create migration project
az dms project create \
  --resource-group rg-prod \
  --service-name dms-prod \
  --name proj-sqlserver-to-mi \
  --source-platform SQL \
  --target-platform SQLMI \
  --location eastus
```

**Log Replay Service (LRS):** Alternative for large databases. Restore full backup
to SQL MI, then replay transaction log backups from blob storage until cutover.

---

## PostgreSQL Flexible Server

Recommended deployment model for Azure Database for PostgreSQL. The legacy Single
Server is retired as of March 2025 — all new deployments use Flexible Server.

### Server Creation

```bash
# Create PostgreSQL Flexible Server
az postgres flexible-server create \
  --resource-group rg-prod \
  --name pg-prod-eastus \
  --location eastus \
  --admin-user pgadmin \
  --admin-password "$PG_PASSWORD" \
  --sku-name Standard_D4s_v3 \
  --tier GeneralPurpose \
  --storage-size 128 \
  --version 16 \
  --high-availability ZoneRedundant \
  --standby-zone 2 \
  --backup-retention 35 \
  --geo-redundant-backup Enabled \
  --private-dns-zone /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Network/privateDnsZones/pg-prod-eastus.private.postgres.database.azure.com \
  --vnet vnet-prod \
  --subnet snet-pg

# Create database
az postgres flexible-server db create \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --database-name myapp

# Scale compute
az postgres flexible-server update \
  --resource-group rg-prod \
  --name pg-prod-eastus \
  --sku-name Standard_D8s_v3

# Scale storage (online, no downtime)
az postgres flexible-server update \
  --resource-group rg-prod \
  --name pg-prod-eastus \
  --storage-size 256

# Stop server (saves compute cost; auto-starts after 7 days)
az postgres flexible-server stop \
  --resource-group rg-prod \
  --name pg-prod-eastus

# Start server
az postgres flexible-server start \
  --resource-group rg-prod \
  --name pg-prod-eastus
```

### Compute Tiers

| Tier                | SKU series         | vCores | Use case                          |
|---------------------|--------------------|--------|-----------------------------------|
| Burstable           | Standard_B1ms–B16ms | 1–16  | Dev/test, low sustained CPU       |
| General Purpose     | Standard_D2s–D96s  | 2–96   | Most production workloads         |
| Memory Optimized    | Standard_E2ds–E96ds| 2–96   | High memory/cache workloads       |

### High Availability

- **Zone Redundant HA:** Primary and standby in different availability zones. Automatic
  failover in 60–120 seconds. Standby not readable — use read replicas for read scaling.
- **Same Zone HA:** Primary and standby in same AZ; lower latency replication. Failover
  is faster but no zone isolation.

```bash
# Enable HA after creation
az postgres flexible-server update \
  --resource-group rg-prod \
  --name pg-prod-eastus \
  --high-availability ZoneRedundant \
  --standby-zone 3

# Trigger manual failover (for testing)
az postgres flexible-server restart \
  --resource-group rg-prod \
  --name pg-prod-eastus \
  --failover-mode ForcedFailover
```

### Read Replicas

```bash
# Create read replica
az postgres flexible-server replica create \
  --resource-group rg-prod \
  --replica-name pg-prod-replica1 \
  --source-server /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.DBforPostgreSQL/flexibleServers/pg-prod-eastus \
  --location westus

# Promote replica to standalone (breaks replication)
az postgres flexible-server replica stop-replication \
  --resource-group rg-prod \
  --name pg-prod-replica1

# List replicas
az postgres flexible-server replica list \
  --resource-group rg-prod \
  --name pg-prod-eastus
```

### PgBouncer (Built-in Connection Pooling)

PgBouncer runs as a sidecar on the same Flexible Server host — no separate VM needed.

```bash
# Enable PgBouncer
az postgres flexible-server update \
  --resource-group rg-prod \
  --name pg-prod-eastus \
  --pgbouncer-enabled true

# Configure pool mode (session/transaction/statement)
az postgres flexible-server parameter set \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --source user-override \
  --name pgbouncer.pool_mode \
  --value transaction
```

PgBouncer listens on port **6432**. Application connects to the same hostname but
port 6432 instead of 5432.

### Extensions

```bash
# List allowed extensions
az postgres flexible-server parameter show \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --name azure.extensions

# Allow extensions (comma-separated)
az postgres flexible-server parameter set \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --name azure.extensions \
  --value "POSTGIS,PG_CRON,PGVECTOR,AZURE_AI"

# Install in database (after allowing)
# psql -h pg-prod-eastus.postgres.database.azure.com -U pgadmin -d myapp
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pgvector;
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS azure_ai;  -- integrates with Azure OpenAI
```

**azure_ai extension:** Enables direct calls to Azure OpenAI and Azure Cognitive
Services from SQL, useful for embedding generation and semantic search.

```sql
-- Set Azure OpenAI endpoint in azure_ai
SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://myoai.openai.azure.com/');
SELECT azure_ai.set_setting('azure_openai.subscription_key', 'mykey');

-- Generate embedding inline
SELECT azure_openai.create_embeddings('text-embedding-ada-002', description)
FROM products LIMIT 5;
```

### Server Parameters

```bash
# Tune shared_buffers
az postgres flexible-server parameter set \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --name shared_buffers \
  --value 131072   # 1 GB in 8kB pages

# Enable slow query log
az postgres flexible-server parameter set \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --name log_min_duration_statement \
  --value 1000   # ms

# List all parameters
az postgres flexible-server parameter list \
  --resource-group rg-prod \
  --server-name pg-prod-eastus \
  --output table
```

---

## MySQL Flexible Server

Azure Database for MySQL Flexible Server (v8.0+). Single Server retired 2024.

### Server Creation

```bash
# Create MySQL Flexible Server
az mysql flexible-server create \
  --resource-group rg-prod \
  --name mysql-prod-eastus \
  --location eastus \
  --admin-user mysqladmin \
  --admin-password "$MYSQL_PASSWORD" \
  --sku-name Standard_D4ds_v4 \
  --tier GeneralPurpose \
  --storage-size 128 \
  --storage-auto-grow Enabled \
  --version 8.0.21 \
  --high-availability ZoneRedundant \
  --backup-retention 35 \
  --geo-redundant-backup Enabled \
  --vnet vnet-prod \
  --subnet snet-mysql \
  --private-dns-zone mysql-prod-eastus.private.mysql.database.azure.com

# Create database
az mysql flexible-server db create \
  --resource-group rg-prod \
  --server-name mysql-prod-eastus \
  --database-name myapp

# Scale up
az mysql flexible-server update \
  --resource-group rg-prod \
  --name mysql-prod-eastus \
  --sku-name Standard_D8ds_v4

# Enable slow query log
az mysql flexible-server parameter set \
  --resource-group rg-prod \
  --server-name mysql-prod-eastus \
  --name slow_query_log \
  --value ON

az mysql flexible-server parameter set \
  --resource-group rg-prod \
  --server-name mysql-prod-eastus \
  --name long_query_time \
  --value 2
```

### High Availability and Read Replicas

```bash
# Create read replica
az mysql flexible-server replica create \
  --resource-group rg-prod \
  --replica-name mysql-prod-replica1 \
  --source-server mysql-prod-eastus

# Promote replica
az mysql flexible-server replica stop-replication \
  --resource-group rg-prod \
  --name mysql-prod-replica1

# Manual failover
az mysql flexible-server failover \
  --resource-group rg-prod \
  --name mysql-prod-eastus
```

### Data-in Replication

Replicate from external MySQL server (on-prem or another cloud) into Azure MySQL
Flexible Server using binlog-based replication:

```sql
-- On source MySQL:
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';

-- On Azure MySQL target:
CALL mysql.az_replication_change_master(
  'source-host.example.com',
  'replication_user',
  'password',
  3306,
  'mysql-bin.000001',  -- binlog file from source
  123,                  -- binlog position
  ''
);
CALL mysql.az_replication_start();
CALL mysql.az_replication_stop();
```

---

## MariaDB

> **Note:** Azure Database for MariaDB is **retiring September 19, 2025**.
> Migrate to Azure Database for MySQL Flexible Server (binary compatible for most workloads)
> or to MariaDB on an Azure VM. Do not start new MariaDB workloads.

```bash
# Migration path: export from MariaDB, import to MySQL Flexible Server
mysqldump -h mariadb-prod.mariadb.database.azure.com \
  -u admin@mariadb-prod -p mydb > mydb_dump.sql

mysql -h mysql-prod-eastus.mysql.database.azure.com \
  -u mysqladmin -p myapp < mydb_dump.sql
```

---

## SQL Server on VMs

For full SQL Server feature parity (SSRS, SSAS, SSIS, polybase, FCI, AG on IaaS).

```bash
# Create SQL Server VM from marketplace image
az vm create \
  --resource-group rg-prod \
  --name vm-sqlserver \
  --image MicrosoftSQLServer:sql2022-ws2022:sqldev-gen2:latest \
  --size Standard_E8ds_v5 \
  --admin-username sqladmin \
  --admin-password "$VM_PASSWORD" \
  --vnet-name vnet-prod \
  --subnet snet-sql \
  --public-ip-address "" \
  --nsg ""

# Register with SQL IaaS Agent Extension (enables automated backups, patching)
az sql vm create \
  --resource-group rg-prod \
  --name vm-sqlserver \
  --location eastus \
  --license-type PAYG \
  --sql-mgmt-type Full

# List SQL VMs
az sql vm list --resource-group rg-prod --output table
```

Azure Hybrid Benefit for SQL Server VMs with existing SA licenses:
```bash
az sql vm update \
  --resource-group rg-prod \
  --name vm-sqlserver \
  --license-type AHUB
```

---

## Cross-Cutting Topics

### Private Link / Private Endpoints

Disable public access and route all traffic over the Azure backbone:

```bash
# Create private endpoint for SQL Database
az network private-endpoint create \
  --resource-group rg-prod \
  --name pe-sqldb \
  --vnet-name vnet-prod \
  --subnet snet-app \
  --private-connection-resource-id /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Sql/servers/sql-prod-eastus \
  --group-id sqlServer \
  --connection-name conn-sqldb

# Create private DNS zone and link
az network private-dns zone create \
  --resource-group rg-prod \
  --name "privatelink.database.windows.net"

az network private-dns link vnet create \
  --resource-group rg-prod \
  --zone-name "privatelink.database.windows.net" \
  --name link-vnet-prod \
  --virtual-network vnet-prod \
  --registration-enabled false

# Create DNS record for the private endpoint
az network private-endpoint dns-zone-group create \
  --resource-group rg-prod \
  --endpoint-name pe-sqldb \
  --name zone-group \
  --private-dns-zone "privatelink.database.windows.net" \
  --zone-name sql

# Disable public access on SQL server
az sql server update \
  --resource-group rg-prod \
  --name sql-prod-eastus \
  --enable-public-network false
```

Private DNS zone names by service:
| Service | Private DNS Zone |
|---------|-----------------|
| SQL Database / SQL MI | `privatelink.database.windows.net` |
| PostgreSQL Flexible Server | `<server>.private.postgres.database.azure.com` |
| MySQL Flexible Server | `<server>.private.mysql.database.azure.com` |

### Backup and Restore (PITR)

```bash
# Point-in-time restore — SQL Database
az sql db restore \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb-restored \
  --source-database mydb \
  --time "2026-04-20T12:00:00Z"

# Geo-restore SQL Database to new server
az sql db restore \
  --resource-group rg-dr \
  --server sql-dr-westus \
  --name mydb-restored \
  --source-database-id /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Sql/servers/sql-prod-eastus/databases/mydb \
  --geo-backup-id <geo-backup-id>

# PITR — PostgreSQL Flexible Server
az postgres flexible-server restore \
  --resource-group rg-prod \
  --name pg-prod-restored \
  --source-server pg-prod-eastus \
  --restore-time "2026-04-20T12:00:00Z"

# PITR — MySQL Flexible Server
az mysql flexible-server restore \
  --resource-group rg-prod \
  --name mysql-prod-restored \
  --source-server mysql-prod-eastus \
  --restore-time "2026-04-20T12:00:00Z"
```

**Long-Term Retention (LTR) for SQL Database:**
```bash
az sql db ltr-policy set \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --database mydb \
  --weekly-retention P4W \
  --monthly-retention P12M \
  --yearly-retention P7Y \
  --week-of-year 1

# Restore from LTR backup
az sql db ltr-backup restore \
  --dest-database mydb-from-ltr \
  --dest-resource-group rg-prod \
  --dest-server sql-prod-eastus \
  --backup-id <ltr-backup-resource-id>
```

### Monitoring

```bash
# Enable diagnostic settings for SQL Database → Log Analytics
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.Sql/servers/sql-prod-eastus/databases/mydb \
  --name diag-sqldb \
  --workspace /subscriptions/<sub>/resourceGroups/rg-prod/providers/Microsoft.OperationalInsights/workspaces/law-prod \
  --logs '[{"category":"SQLInsights","enabled":true},{"category":"QueryStoreRuntimeStatistics","enabled":true},{"category":"Errors","enabled":true},{"category":"Timeouts","enabled":true},{"category":"Deadlocks","enabled":true}]' \
  --metrics '[{"category":"Basic","enabled":true}]'

# Useful KQL queries in Log Analytics:
# -- Top slow queries (PostgreSQL)
# AzureDiagnostics
# | where Category == "PostgreSQLSlowLogs"
# | where duration_s > 1
# | order by duration_s desc
# | take 50

# -- SQL Database deadlocks
# AzureDiagnostics
# | where Category == "Deadlocks"
# | project TimeGenerated, databaseName_s, deadlock_xml_s
```

### Connection Strings

```bash
# Azure SQL Database
Server=sql-prod-eastus.database.windows.net;Database=mydb;User Id=sqladmin;Password=...;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;

# SQL Database with Entra (managed identity)
Server=sql-prod-eastus.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity;

# SQL Database via failover group
Server=fog-myapp.database.windows.net;Database=mydb;User Id=sqladmin;Password=...;Encrypt=True;

# PostgreSQL Flexible Server
Host=pg-prod-eastus.postgres.database.azure.com;Port=5432;Database=myapp;Username=pgadmin;Password=...;Ssl Mode=Require;

# PostgreSQL via PgBouncer
Host=pg-prod-eastus.postgres.database.azure.com;Port=6432;Database=myapp;Username=pgadmin;Password=...;Ssl Mode=Require;

# MySQL Flexible Server
Server=mysql-prod-eastus.mysql.database.azure.com;Port=3306;Database=myapp;Uid=mysqladmin;Pwd=...;SslMode=Required;

# Python (psycopg2 — PostgreSQL)
import psycopg2
conn = psycopg2.connect(
    host="pg-prod-eastus.postgres.database.azure.com",
    dbname="myapp",
    user="pgadmin",
    password=os.environ["PG_PASSWORD"],
    sslmode="require"
)

# Python (pyodbc — SQL Database with Entra managed identity)
import pyodbc, struct
from azure.identity import ManagedIdentityCredential
credential = ManagedIdentityCredential()
token = credential.get_token("https://database.windows.net/")
token_bytes = struct.pack("<i", len(token.token) * 2) + token.token.encode("utf-16-le")
conn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=sql-prod-eastus.database.windows.net;"
    "Database=mydb;",
    attrs_before={1256: token_bytes}
)
```

### Database Migration Service (DMS) Patterns

| Source                        | Target                   | Method          |
|-------------------------------|--------------------------|-----------------|
| SQL Server on-prem            | Azure SQL Database        | DMS online/offline |
| SQL Server on-prem            | SQL Managed Instance      | DMS, LRS, backup/restore |
| Oracle                        | Azure SQL Database        | SSMA (SQL Server Migration Assistant) |
| MySQL on-prem                 | Azure MySQL Flexible      | DMS, mysqldump, Data-in replication |
| PostgreSQL on-prem            | Azure PostgreSQL Flexible | DMS, pg_dump, logical replication |
| RDS PostgreSQL/MySQL          | Azure Flexible Server     | DMS online migration |
| MariaDB                       | Azure MySQL Flexible      | mysqldump/restore |

```bash
# DMS online migration task (PostgreSQL example)
az dms project task create \
  --resource-group rg-prod \
  --service-name dms-prod \
  --project-name proj-pg-migration \
  --name task-online-pg \
  --task-type OnlineMigration \
  --source-connection-json @pg-source-conn.json \
  --target-connection-json @pg-target-conn.json \
  --database-options-json @pg-db-options.json
```

---

## Infrastructure as Code

### Bicep — Azure SQL Database

```bicep
param location string = resourceGroup().location
param sqlServerName string
param sqlDbName string
@secure()
param adminPassword string

resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: sqlServerName
  location: location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: adminPassword
    publicNetworkAccess: 'Disabled'
    minimalTlsVersion: '1.2'
  }
}

resource sqlDb 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: sqlDbName
  location: location
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 4
  }
  properties: {
    autoPauseDelay: -1
    zoneRedundant: true
    backupStorageRedundancy: 'Geo'
    requestedBackupStorageRedundancy: 'Geo'
  }
}

resource auditingPolicy 'Microsoft.Sql/servers/auditingSettings@2023-05-01-preview' = {
  parent: sqlServer
  name: 'default'
  properties: {
    state: 'Enabled'
    isAzureMonitorTargetEnabled: true
    retentionDays: 90
  }
}
```

### Bicep — PostgreSQL Flexible Server

```bicep
param location string = resourceGroup().location
param serverName string
@secure()
param adminPassword string

resource pgServer 'Microsoft.DBforPostgreSQL/flexibleServers@2023-12-01-preview' = {
  name: serverName
  location: location
  sku: {
    name: 'Standard_D4s_v3'
    tier: 'GeneralPurpose'
  }
  properties: {
    administratorLogin: 'pgadmin'
    administratorLoginPassword: adminPassword
    version: '16'
    storage: {
      storageSizeGB: 128
      autoGrow: 'Enabled'
    }
    backup: {
      backupRetentionDays: 35
      geoRedundantBackup: 'Enabled'
    }
    highAvailability: {
      mode: 'ZoneRedundant'
      standbyAvailabilityZone: '2'
    }
    network: {
      delegatedSubnetResourceId: subnetId
      privateDnsZoneArmResourceId: privateDnsZoneId
    }
  }
}

resource pgDb 'Microsoft.DBforPostgreSQL/flexibleServers/databases@2023-12-01-preview' = {
  parent: pgServer
  name: 'myapp'
  properties: {
    charset: 'utf8'
    collation: 'en_US.utf8'
  }
}
```

### Terraform — MySQL Flexible Server

```hcl
resource "azurerm_mysql_flexible_server" "prod" {
  name                   = "mysql-prod-eastus"
  resource_group_name    = azurerm_resource_group.prod.name
  location               = azurerm_resource_group.prod.location
  administrator_login    = "mysqladmin"
  administrator_password = var.mysql_admin_password
  sku_name               = "GP_Standard_D4ds_v4"
  version                = "8.0.21"

  storage {
    size_gb           = 128
    auto_grow_enabled = true
    iops              = 684
  }

  backup {
    retention_days            = 35
    geo_redundant_backup_enabled = true
  }

  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }

  network {
    delegated_subnet_id    = azurerm_subnet.mysql.id
    private_dns_zone_id    = azurerm_private_dns_zone.mysql.id
  }

  depends_on = [azurerm_private_dns_zone_virtual_network_link.mysql]
}

resource "azurerm_mysql_flexible_database" "myapp" {
  name                = "myapp"
  resource_group_name = azurerm_resource_group.prod.name
  server_name         = azurerm_mysql_flexible_server.prod.name
  charset             = "utf8mb4"
  collation           = "utf8mb4_unicode_ci"
}
```

---

## Cost Optimization

### Reserved Capacity

Save up to 33% (1-year) or 56% (3-year) vs pay-as-you-go:

```bash
# Purchase reserved capacity via CLI (requires Reservation Purchaser role)
az reservations reservation-order purchase \
  --applied-scope-type Shared \
  --billing-scope /subscriptions/<sub> \
  --display-name "SQL DB Gen5 4vCore 1yr" \
  --quantity 1 \
  --reserved-resource-type SqlDatabases \
  --sku Standard_D4 \
  --term P1Y \
  --billing-plan Upfront
```

Reserved capacity is available for:
- Azure SQL Database (vCore)
- Azure SQL Managed Instance
- Azure Database for PostgreSQL
- Azure Database for MySQL

### Serverless and Auto-Pause

Use serverless compute tier for dev/test or intermittent workloads:
- Set `--auto-pause-delay 60` to pause after 1 hour idle
- Billing drops to storage-only during pause
- Not applicable to Hyperscale or Business Critical tiers

### Right-Sizing Checklist

```bash
# Check SQL Database DTU/vCore consumption
az monitor metrics list \
  --resource <db-resource-id> \
  --metric "cpu_percent,dtu_consumption_percent,storage_percent" \
  --interval PT1H \
  --start-time "$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --aggregation Average Maximum \
  --output table
```

- If `cpu_percent` avg < 20% and max < 50% → scale down one tier
- If `storage_percent` > 80% → increase storage or enable auto-grow
- Use elastic pools when individual db peak times don't overlap
- Enable `storage-auto-grow` on PostgreSQL and MySQL to avoid manual intervention

### Azure Hybrid Benefit

Apply existing SQL Server licenses with Software Assurance to reduce SQL MI and SQL DB costs:

```bash
az sql mi update \
  --resource-group rg-prod \
  --name sqlmi-prod-eastus \
  --license-type BasePrice   # BasePrice = AHB; LicenseIncluded = pay full

az sql db update \
  --resource-group rg-prod \
  --server sql-prod-eastus \
  --name mydb \
  --license-type BasePrice
```

---

## Quick Reference

| Service | Default Port | Connection hostname pattern |
|---------|-------------|----------------------------|
| Azure SQL Database | 1433 | `<server>.database.windows.net` |
| SQL Managed Instance | 1433 | `<mi>.{dns-zone}.database.windows.net` |
| PostgreSQL Flexible | 5432 (direct) / 6432 (PgBouncer) | `<server>.postgres.database.azure.com` |
| MySQL Flexible | 3306 | `<server>.mysql.database.azure.com` |

**Useful resource group listing commands:**
```bash
az sql server list -o table
az sql mi list -o table
az postgres flexible-server list -o table
az mysql flexible-server list -o table
```

**See also:** The `azure-cosmosdb` skill for NoSQL, document, graph, and table API databases.
