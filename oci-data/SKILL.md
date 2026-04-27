---
name: oci-data
description: OCI data services — Autonomous Database, MySQL HeatWave, NoSQL, Object Storage, GoldenGate, Streaming
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

# OCI Data Services

## Autonomous Database (ADB)

ADB comes in two workload types:
- **ATP** (Autonomous Transaction Processing) — `db_workload = OLTP`
- **ADW** (Autonomous Data Warehouse) — `db_workload = DW`

And two infrastructure models:
- **Serverless** — auto-scales, shared infrastructure, pay per ECPU-second
- **Dedicated** — on dedicated Exadata infrastructure, more control/isolation

### Create Autonomous Database
```bash
# Serverless ATP (ECPU model — newer, recommended)
oci db autonomous-database create \
  --compartment-id $C \
  --display-name "prod-atp" \
  --db-name "PRODATP" \
  --db-workload "OLTP" \
  --compute-model "ECPU" \
  --compute-count 2 \
  --data-storage-size-in-tbs 1 \
  --admin-password 'MyStr0ng#Pass!' \
  --is-auto-scaling-enabled true \
  --is-auto-scaling-for-storage-enabled true \
  --license-model "LICENSE_INCLUDED" \
  --whitelisted-ips '["0.0.0.0/0"]'   # or restrict to specific IPs

# Always Free ADB (1 OCPU, 20 GB, no auto-scaling)
oci db autonomous-database create \
  --compartment-id $C \
  --display-name "free-atp" \
  --db-name "FREEATP" \
  --db-workload "OLTP" \
  --is-free-tier true \
  --admin-password 'MyStr0ng#Pass!'

# ADW (Data Warehouse)
oci db autonomous-database create \
  --compartment-id $C \
  --display-name "prod-adw" \
  --db-name "PRODADW" \
  --db-workload "DW" \
  --compute-model "ECPU" \
  --compute-count 4 \
  --data-storage-size-in-tbs 2 \
  --admin-password 'MyStr0ng#Pass!'
```

### Manage ADB
```bash
# List databases
oci db autonomous-database list --compartment-id $C --all --output table \
  --query 'data[*].{"Name":"display-name","State":"lifecycle-state","CPU":"compute-count","TB":"data-storage-size-in-tbs"}'

# Scale compute (ECPU)
oci db autonomous-database update \
  --autonomous-database-id $ADB_ID \
  --compute-count 8

# Stop/start (billing stops when stopped)
oci db autonomous-database stop --autonomous-database-id $ADB_ID
oci db autonomous-database start --autonomous-database-id $ADB_ID

# Download wallet (connection credentials)
oci db autonomous-database generate-wallet \
  --autonomous-database-id $ADB_ID \
  --password 'WalletPass#1' \
  --file ./wallet.zip
unzip wallet.zip -d ./wallet/

# Set wallet path for SQLcl / SQL*Plus
export TNS_ADMIN=./wallet/
export ADB_PASSWORD='MyStr0ng#Pass!'
sql admin/$ADB_PASSWORD@PRODATP_HIGH

# Clone database
oci db autonomous-database create-from-clone \
  --compartment-id $C \
  --source-id $ADB_ID \
  --clone-type "FULL" \
  --display-name "atp-clone" \
  --db-name "ATPCLONE" \
  --admin-password 'MyStr0ng#Pass!'
```

### ADB Connection Strings
```python
# Python with python-oracledb (thick mode for wallet)
import oracledb
oracledb.init_oracle_client()

conn = oracledb.connect(
    user="admin",
    password="MyStr0ng#Pass!",
    dsn="PRODATP_HIGH",
    config_dir="/path/to/wallet",
    wallet_location="/path/to/wallet",
    wallet_password="WalletPass#1"
)
```

```bash
# SQLcl connection
sql admin@PRODATP_HIGH
```

## MySQL HeatWave

MySQL HeatWave adds in-database analytics and ML directly to MySQL — no ETL needed.

```bash
# Create MySQL HeatWave cluster
oci mysql db-system create \
  --compartment-id $C \
  --display-name "prod-mysql" \
  --availability-domain "kWVD:US-ASHBURN-AD-1" \
  --shape-name "MySQL.HeatWave.VM.Standard.E3" \
  --subnet-id $PRIVATE_SUBNET_ID \
  --admin-username "admin" \
  --admin-password 'MyStr0ng#Pass!' \
  --data-storage-size-in-gbs 512 \
  --is-highly-available true

# Enable HeatWave cluster (in-memory analytics)
oci mysql heat-wave-cluster add \
  --db-system-id $MYSQL_ID \
  --shape-name "MySQL.HeatWave.VM.Standard.E3" \
  --cluster-size 2     # Number of HeatWave nodes

# List MySQL systems
oci mysql db-system list --compartment-id $C --all --output table

# Get endpoint
oci mysql db-system get --db-system-id $MYSQL_ID \
  --query 'data.endpoints[0]."ip-address"' --raw-output
```

```sql
-- Load table into HeatWave (in-memory)
ALTER TABLE orders SECONDARY_ENGINE=RAPID;
ALTER TABLE orders SECONDARY_LOAD;

-- Force query to use HeatWave
SET SESSION use_secondary_engine=FORCED;
SELECT region, SUM(total) FROM orders GROUP BY region;

-- HeatWave AutoML
CALL sys.ML_TRAIN('mydb.orders', 'total', 
  JSON_OBJECT('task', 'regression'), 'my_model');
CALL sys.ML_PREDICT_ROW(JSON_OBJECT('quantity', 5, 'price', 100), 
  'my_model', NULL);
```

## OCI NoSQL Database

Compatible with Oracle NoSQL / Cassandra-style workloads. Serverless pay-per-use.

```bash
# Create table
oci nosql table create \
  --compartment-id $C \
  --name "UserEvents" \
  --ddl-statement "CREATE TABLE UserEvents (
    userId STRING,
    timestamp LONG,
    eventType STRING,
    payload JSON,
    PRIMARY KEY (SHARD(userId), timestamp)
  )" \
  --table-limits '{"maxReadUnits":50,"maxWriteUnits":50,"maxStorageInGBs":25}'

# List tables
oci nosql table list --compartment-id $C --all

# Execute query via CLI
oci nosql query execute \
  --compartment-id $C \
  --statement "SELECT * FROM UserEvents WHERE userId = 'user123' LIMIT 10"
```

## Object Storage

### Buckets
```bash
# Get namespace (tenancy-specific, needed for API calls)
NAMESPACE=$(oci os ns get --query 'data' --raw-output)

# Create buckets with different tiers
oci os bucket create --compartment-id $C --name "hot-data" \
  --storage-tier Standard --versioning Enabled

oci os bucket create --compartment-id $C --name "archive-data" \
  --storage-tier Archive

oci os bucket create --compartment-id $C --name "infrequent-data" \
  --storage-tier InfrequentAccess

# Make bucket public (read-only)
oci os bucket update --name "public-assets" --public-access-type ObjectRead
```

### Lifecycle Rules
```bash
oci os object-lifecycle-policy put \
  --bucket-name "hot-data" \
  --items '[
    {
      "name": "archive-old-logs",
      "action": "ARCHIVE",
      "timeAmount": 30,
      "timeUnit": "DAYS",
      "objectNameFilter": {"inclusionPrefixes": ["logs/"]},
      "isEnabled": true
    },
    {
      "name": "delete-archived-after-1y",
      "action": "DELETE",
      "timeAmount": 365,
      "timeUnit": "DAYS",
      "isEnabled": true
    }
  ]'
```

### Pre-Authenticated Requests (PARs)
```bash
# Create PAR for reading a single object (expires in 7 days)
oci os preauth-request create \
  --bucket-name "reports" \
  --name "weekly-report-par" \
  --access-type "ObjectRead" \
  --object-name "2024/week-01-report.pdf" \
  --time-expires "$(date -u -d '+7 days' '+%Y-%m-%dT%H:%M:%SZ')"

# Create PAR for any object in bucket (upload)
oci os preauth-request create \
  --bucket-name "uploads" \
  --name "upload-par" \
  --access-type "AnyObjectWrite" \
  --time-expires "$(date -u -d '+1 hour' '+%Y-%m-%dT%H:%M:%SZ')"

# Use PAR URL to upload (no auth needed)
curl -T ./file.csv "https://objectstorage.us-ashburn-1.oraclecloud.com/p/<par-token>/n/$NAMESPACE/b/uploads/o/file.csv"
```

### Replication
```bash
# Cross-region replication for DR
oci os replication create-replication-policy \
  --bucket-name "prod-data" \
  --name "dr-replication" \
  --destination-region "eu-frankfurt-1" \
  --destination-bucket-name "prod-data-dr"
```

## GoldenGate (Real-time Replication/CDC)

OCI GoldenGate is managed change data capture and replication.

```bash
# Create GoldenGate deployment
oci goldengate deployment create \
  --compartment-id $C \
  --display-name "prod-gg" \
  --license-model "LICENSE_INCLUDED" \
  --deployment-type "DATABASE_ORACLE" \
  --cpu-core-count 2 \
  --is-auto-scaling-enabled false \
  --subnet-id $PRIVATE_SUBNET_ID \
  --fqdn "gg.internal.example.com" \
  --admin-username "oggadmin" \
  --admin-password 'GGPass#123!' \
  --ogg-data '{"deploymentName":"prod-gg","adminUsername":"oggadmin"}'

# Create source connection (ATP → ATP replication example)
oci goldengate connection create-oracle-connection \
  --compartment-id $C \
  --display-name "source-atp" \
  --connection-type "ORACLE" \
  --technology-type "AUTONOMOUS_DATABASE" \
  --username "ggadmin" \
  --password 'GGPass#123!' \
  --wallet "$( base64 -w0 wallet.zip )" \
  --database-id $SOURCE_ADB_ID
```

## OCI Streaming (Kafka-Compatible)

Managed Kafka-compatible event streaming.

```bash
# Create stream pool
POOL_ID=$(oci streaming admin stream-pool create \
  --compartment-id $C \
  --name "prod-streams" \
  --query 'data.id' --raw-output)

# Create stream (topic)
oci streaming admin stream create \
  --compartment-id $C \
  --name "user-events" \
  --partitions 3 \
  --retention-in-hours 24 \
  --stream-pool-id $POOL_ID

# Get Kafka-compatible endpoint
oci streaming admin stream-pool get --stream-pool-id $POOL_ID \
  --query 'data."kafka-settings"'

# Produce message via OCI CLI
STREAM_ID=$(oci streaming admin stream list --stream-pool-id $POOL_ID \
  --query 'data[0].id' --raw-output)

oci streaming stream message put \
  --stream-id $STREAM_ID \
  --messages '[{"key":"'$(echo -n "key1" | base64)'","value":"'$(echo -n '{"event":"login","userId":"123"}' | base64)'"}]'

# Consume messages
oci streaming stream cursor create-cursor \
  --stream-id $STREAM_ID \
  --partition "0" \
  --type "TRIM_HORIZON"

oci streaming stream message get \
  --stream-id $STREAM_ID \
  --cursor "<cursor-value>" \
  --limit 100
```

```python
# Kafka-compatible producer
from confluent_kafka import Producer

conf = {
    'bootstrap.servers': 'cell-1.streaming.us-ashburn-1.oci.oraclecloud.com:9092',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism': 'PLAIN',
    'sasl.username': f'{TENANCY_NAME}/{USERNAME}/{STREAM_POOL_OCID}',
    'sasl.password': AUTH_TOKEN,
}

p = Producer(conf)
p.produce('user-events', key='user123', value='{"action":"login"}')
p.flush()
```

## Data Catalog

```bash
# Create data catalog
oci data-catalog catalog create --compartment-id $C --display-name "prod-catalog"

# Harvest metadata from Object Storage
oci data-catalog data-asset create \
  --catalog-id $CATALOG_ID \
  --display-name "prod-data-lake" \
  --type-key "hive_data_asset"  # Use oci data-catalog type list to find keys
```

## Gotchas & Tips

- **ADB ECPU vs OCPU** — new databases should use ECPU (more granular, cheaper at low scales). OCPU is legacy. 1 OCPU ≈ 2 ECPUs in pricing equivalence.
- **ADB whitelisted IPs** — `"0.0.0.0/0"` means "allow all public IPs". For tighter security, list specific CIDRs or use private endpoint (no public IP at all).
- **Object Storage namespace** — the namespace is the tenancy name (for non-gov regions) or a generated string. Always fetch dynamically with `oci os ns get`.
- **Archive tier restore** — objects in Archive tier must be restored before download (takes ~1 hour). Use `oci os object restore` and wait for `RESTORED` state.
- **GoldenGate requires OGG user** — source databases need a GoldenGate-enabled user with LogMiner privileges. For ATP, run `DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('ggadmin', 'capture', grant_select_privileges=>true)`.
- **Streaming auth tokens** — use OCI Auth Tokens (not API keys) for Kafka client passwords. Generate via `oci iam auth-token create`.
- **HeatWave node count** — rule of thumb: 1 HeatWave node per ~400 GB of data. Start with 2 for HA; the system will recommend sizing after `SECONDARY_LOAD`.
- **NoSQL capacity units** — 1 Read Unit = 1 KB read/second; 1 Write Unit = 1 KB write/second. Start low and scale up; billing is per provisioned unit.
