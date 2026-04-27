---
name: gcp-data
description: GCP data services — Cloud SQL, Firestore, Bigtable, BigQuery, Cloud Storage, Pub/Sub, Dataflow, Spanner, Memorystore
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Data Services Skill

Choose the right storage and data processing service for your workload.

---

## Service Selection Guide

| Need | Service |
|------|---------|
| Relational DB (OLTP) | Cloud SQL (MySQL/PostgreSQL/SQL Server) |
| Relational, globally distributed | Cloud Spanner |
| Document / NoSQL | Firestore |
| Wide-column (IoT, time-series) | Cloud Bigtable |
| Analytics / OLAP | BigQuery |
| Object storage (files, backups) | Cloud Storage |
| Message queue / event bus | Pub/Sub |
| Stream/batch data pipelines | Dataflow (Apache Beam) |
| Redis / Memcached cache | Memorystore |

---

## Cloud SQL

### Instance Creation

```bash
# PostgreSQL (recommended over MySQL for new projects)
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --region=us-central1 \
  --tier=db-n1-standard-2 \
  --storage-size=50GB \
  --storage-type=SSD \
  --storage-auto-increase \
  --backup-start-time=03:00 \
  --enable-bin-log \
  --deletion-protection \
  --no-assign-ip \
  --network=projects/my-project/global/networks/my-vpc

# Create a database and user
gcloud sql databases create mydb --instance=my-db
gcloud sql users create myuser \
  --instance=my-db \
  --password=SECURE_PASSWORD

# Connect using Cloud SQL Proxy (recommended — no public IP needed)
cloud-sql-proxy my-project:us-central1:my-db &
psql -h 127.0.0.1 -U myuser -d mydb
```

### High Availability

```bash
# Enable HA (creates a standby replica in a different zone)
gcloud sql instances patch my-db \
  --availability-type=REGIONAL

# Create a read replica
gcloud sql instances create my-db-replica \
  --master-instance-name=my-db \
  --region=us-central1 \
  --tier=db-n1-standard-2

# Promote replica to primary (for DR)
gcloud sql instances promote-replica my-db-replica

# Point-in-time recovery
gcloud sql instances restore-backup my-db \
  --backup-instance=my-db \
  --backup-id=BACKUP_ID
```

**Gotcha:** HA failover can take 60–120 seconds. It's not a zero-downtime operation.
Applications must handle connection retries.

---

## Firestore

```bash
# Create a database (native mode — recommended)
gcloud firestore databases create \
  --location=us-central1 \
  --type=firestore-native \
  --database=my-database

# Or use Datastore mode (for legacy Datastore apps)
gcloud firestore databases create \
  --location=us-central1 \
  --type=datastore-mode

# Export data to GCS
gcloud firestore export gs://my-bucket/firestore-backup \
  --collection-ids=users,orders

# Import data from GCS
gcloud firestore import gs://my-bucket/firestore-backup

# List databases
gcloud firestore databases list
```

### Key Concepts

- **Native mode** = true NoSQL document DB with real-time listeners and mobile SDK support.
- **Datastore mode** = legacy Datastore API, no real-time, no multi-doc transactions across entity groups.
- Documents in Firestore are automatically indexed on every field. For large collections, create composite indexes for complex queries.
- Firestore is **regional** — choose location carefully (cannot change after creation).
- **Subcollections** don't count toward parent document size (1 MB limit per doc).

---

## Cloud Bigtable

Wide-column NoSQL for high-throughput, low-latency (IoT, time-series, ad tech).

```bash
# Create an instance
gcloud bigtable instances create my-bigtable \
  --cluster=my-cluster \
  --cluster-zone=us-central1-a \
  --cluster-num-nodes=3 \
  --instance-type=PRODUCTION \
  --display-name="My Bigtable"

# Create a table
cbt -project=my-project -instance=my-bigtable createtable my-table

# Create column family
cbt -project=my-project -instance=my-bigtable createfamily my-table cf1

# Write a row
cbt -project=my-project -instance=my-bigtable set my-table row1 cf1:col1=value1

# Read a row
cbt -project=my-project -instance=my-bigtable read my-table prefix=row1

# Scale cluster (add nodes for more throughput)
gcloud bigtable clusters update my-cluster \
  --instance=my-bigtable \
  --num-nodes=6

# List instances
gcloud bigtable instances list
```

**Row key design is critical.** Avoid sequential timestamps as row keys (hotspots).
Use hash prefixes or reverse timestamps.

---

## BigQuery

### Datasets and Tables

```bash
# Create a dataset
bq mk --dataset \
  --location=US \
  --description="Production analytics" \
  my-project:analytics

# Create a table with schema
bq mk --table \
  my-project:analytics.events \
  schema.json

# Create a partitioned + clustered table
bq mk --table \
  --time_partitioning_field=event_date \
  --time_partitioning_type=DAY \
  --clustering_fields=event_type,user_id \
  my-project:analytics.events \
  schema.json

# Run a query
bq query --use_legacy_sql=false --project_id=my-project \
  'SELECT COUNT(*) as cnt FROM `analytics.events` WHERE DATE(event_date) = CURRENT_DATE()'

# Query with parameterized values
bq query --use_legacy_sql=false \
  --parameter='start_date:DATE:2024-01-01' \
  'SELECT * FROM `analytics.events` WHERE event_date >= @start_date LIMIT 100'

# Run an async job and check status
bq query --use_legacy_sql=false \
  --nosync \
  'SELECT ...' 
bq wait <job_id>
```

### Partitioning and Clustering

```sql
-- Ingestion-time partitioning (auto, no column needed)
CREATE TABLE analytics.events_partitioned
PARTITION BY _PARTITIONDATE
OPTIONS (
  partition_expiration_days = 365,
  require_partition_filter = true  -- forces queries to filter on partition
)
AS SELECT * FROM analytics.events WHERE FALSE;

-- Column-based date partitioning
CREATE TABLE analytics.sessions (
  session_id STRING,
  user_id STRING,
  created_at TIMESTAMP,
  revenue FLOAT64
)
PARTITION BY DATE(created_at)
CLUSTER BY user_id, revenue;
```

**Gotcha:** `require_partition_filter = true` prevents full table scans — good for cost control.

### Scheduled Queries

```bash
# Create a scheduled query (via bq or console)
bq query --use_legacy_sql=false \
  --schedule="every 24 hours" \
  --display_name="Daily aggregation" \
  --destination_table=my-project:analytics.daily_agg \
  --replace \
  'SELECT DATE(event_date) as d, COUNT(*) as cnt FROM `analytics.events` GROUP BY 1'
```

### Slots and Reservations

```bash
# Buy slot commitments (for workloads with consistent usage)
bq mk --capacity_commitment \
  --location=US \
  --plan=FLEX \
  --slots=100

# Create a reservation
bq mk --reservation \
  --location=US \
  --slots=100 \
  my-project:US.my-reservation

# Assign project to reservation
bq mk --reservation_assignment \
  --reservation_id=my-project:US.my-reservation \
  --job_type=QUERY \
  --assignee_type=PROJECT \
  --assignee_id=my-project
```

---

## Cloud Storage

### Storage Classes

| Class | Use Case | Min Duration |
|-------|---------|-------------|
| Standard | Frequently accessed | None |
| Nearline | Once/month access | 30 days |
| Coldline | Once/quarter access | 90 days |
| Archive | Once/year access | 365 days |

```bash
# Create bucket with class
gsutil mb -l us-central1 -c NEARLINE gs://my-archive-bucket

# Set lifecycle policy
cat > lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90, "matchesStorageClass": ["NEARLINE"]}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
EOF
gsutil lifecycle set lifecycle.json gs://my-bucket

# Generate a signed URL (7 days, for sharing private objects)
gcloud storage sign-url gs://my-bucket/private-file.zip \
  --duration=7d \
  --impersonate-service-account=my-sa@my-project.iam.gserviceaccount.com
```

---

## Pub/Sub

```bash
# Create topic and subscription
gcloud pubsub topics create my-topic
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --ack-deadline=60 \
  --message-retention-duration=7d

# Push subscription (delivers to an HTTP endpoint)
gcloud pubsub subscriptions create my-push-sub \
  --topic=my-topic \
  --push-endpoint=https://my-service.run.app/pubsub

# Publish messages
gcloud pubsub topics publish my-topic \
  --message='{"event":"user_signup","user_id":123}' \
  --attribute=event_type=user_signup

# Pull messages
gcloud pubsub subscriptions pull my-sub --auto-ack --limit=10

# Create a snapshot (for replay)
gcloud pubsub snapshots create my-snapshot --subscription=my-sub

# Seek to a snapshot (replay messages)
gcloud pubsub subscriptions seek my-sub --snapshot=my-snapshot

# Dead letter topic
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --dead-letter-topic=my-dead-letter-topic \
  --max-delivery-attempts=5
```

---

## Dataflow (Apache Beam)

```bash
# Run a Dataflow template (WordCount example)
gcloud dataflow jobs run my-wordcount \
  --gcs-location=gs://dataflow-templates/latest/Word_Count \
  --region=us-central1 \
  --staging-location=gs://my-bucket/staging \
  --parameters=inputFile=gs://dataflow-samples/shakespeare/kinglear.txt,output=gs://my-bucket/output/

# Submit a custom Python Beam pipeline
python3 my_pipeline.py \
  --runner=DataflowRunner \
  --project=my-project \
  --region=us-central1 \
  --temp_location=gs://my-bucket/temp \
  --staging_location=gs://my-bucket/staging \
  --job_name=my-pipeline

# List Dataflow jobs
gcloud dataflow jobs list --region=us-central1

# Cancel a job
gcloud dataflow jobs cancel JOB_ID --region=us-central1

# Drain a streaming job (graceful shutdown — processes buffered data)
gcloud dataflow jobs drain JOB_ID --region=us-central1
```

---

## Cloud Spanner

Globally distributed relational DB with ACID transactions. Expensive but unique.

```bash
# Create an instance
gcloud spanner instances create my-instance \
  --config=regional-us-central1 \
  --nodes=1 \
  --description="My Spanner instance"

# Create a database
gcloud spanner databases create my-db \
  --instance=my-instance

# Execute SQL
gcloud spanner databases execute-sql my-db \
  --instance=my-instance \
  --sql="SELECT * FROM Users LIMIT 10"

# Create a database with DDL
gcloud spanner databases create my-db \
  --instance=my-instance \
  --ddl="CREATE TABLE Users (
    UserId STRING(36) NOT NULL,
    Name STRING(255),
    CreatedAt TIMESTAMP
  ) PRIMARY KEY (UserId)"
```

---

## Memorystore (Redis / Memcached)

```bash
# Create a Redis instance
gcloud redis instances create my-redis \
  --size=1 \
  --region=us-central1 \
  --redis-version=redis_7_0 \
  --network=projects/my-project/global/networks/my-vpc \
  --tier=STANDARD_HA

# Get Redis host/port
gcloud redis instances describe my-redis \
  --region=us-central1 \
  --format="value(host,port)"

# Create Memcached instance
gcloud memcache instances create my-memcached \
  --node-count=1 \
  --node-cpu=1 \
  --node-memory=1 \
  --region=us-central1 \
  --network=my-vpc
```

---

## Guardrails

- **Cloud SQL: always enable `--deletion-protection`** on production instances.
- **BigQuery: use `require_partition_filter`** on large tables to prevent expensive full scans.
- **Pub/Sub: always configure a dead-letter topic** for production subscriptions.
- **Firestore: indexes cost money and slow writes** — disable indexing on large blob fields.
- **Bigtable: row key design can't be changed** — model it carefully before creating tables.
- **Spanner: pay per processing unit** — even idle instances cost ~$0.90/node-hour. Scale down dev instances.
- **Memorystore has no public IP** — must access from VPC. Deploy in the same region as your app.
- **Cloud Storage: `uniform_bucket_level_access = true`** — disable per-object ACLs and use IAM only.
