---
name: ibm-data
description: Db2 on Cloud, IBM Cloud Databases (PostgreSQL/MongoDB/Redis/etc via ibmcloud cdb), Cloud Object Storage (S3-compatible, HMAC, Aspera), Cloudant (CouchDB), Event Streams (Kafka)
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

# IBM Cloud Data Services Skill

Covers the major IBM Cloud data services: **Db2 on Cloud**, **IBM Cloud Databases** (ICD —
PostgreSQL, MongoDB, Redis, Elasticsearch, etcd, MySQL, RabbitMQ), **Cloud Object Storage** (COS),
**IBM Cloudant**, and **IBM Event Streams** (Kafka).

---

## IBM Cloud Databases (ICD) — General

All ICD services share a common CLI interface via the `cloud-databases` plugin.

```bash
# Install plugin
ibmcloud plugin install cloud-databases   # commands: ibmcloud cdb

# List all ICD instances (across all ICD service types)
ibmcloud cdb ls

# Supported service names for provisioning:
# databases-for-postgresql
# databases-for-mongodb
# databases-for-redis
# databases-for-elasticsearch
# databases-for-etcd
# databases-for-mysql
# databases-for-enterprisedb
# messages-for-rabbitmq
```

### Provisioning an ICD Instance

```bash
# PostgreSQL example (Standard plan is the only paid option — Lite is deprecated)
ibmcloud resource service-instance-create my-pg \
  databases-for-postgresql \
  standard \
  us-south \
  -g my-resource-group \
  -p '{
    "members_memory_allocation_mb": 4096,
    "members_disk_allocation_mb": 20480,
    "members_cpu_allocation_count": 0,
    "version": "15"
  }'

# MongoDB example
ibmcloud resource service-instance-create my-mongo \
  databases-for-mongodb \
  standard \
  us-south \
  -p '{"members_memory_allocation_mb": 4096, "members_disk_allocation_mb": 20480, "version": "7"}'

# Redis
ibmcloud resource service-instance-create my-redis \
  databases-for-redis \
  standard \
  us-south \
  -p '{"members_memory_allocation_mb": 2048, "members_disk_allocation_mb": 5120}'
```

### Connecting to an ICD Instance

```bash
# Create credentials (service key)
ibmcloud resource service-key-create my-pg-creds Administrator \
  --instance-name my-pg

# Extract connection info
ibmcloud resource service-key my-pg-creds --output json | \
  jq '.credentials.connection'

# Or use cdb plugin shortcut
ibmcloud cdb deployment-connections my-pg --user admin --password "$PG_PASSWORD"

# Get connection strings directly
ibmcloud cdb deployment-connections my-pg --endpoint-type private  # VPC private endpoint
ibmcloud cdb deployment-connections my-pg --endpoint-type public

# List users
ibmcloud cdb deployment-user-list my-pg

# Create a database user
ibmcloud cdb deployment-user-create my-pg myuser --password "StrongPass123!"

# Change user password
ibmcloud cdb deployment-user-password my-pg myuser --password "NewPass456!"

# Delete user
ibmcloud cdb deployment-user-delete my-pg myuser
```

### Scaling

```bash
# Scale memory (applies to all members)
ibmcloud cdb deployment-groups-set-members my-pg member \
  --memory 8192

# Scale disk
ibmcloud cdb deployment-groups-set-members my-pg member \
  --disk 40960

# Scale CPU
ibmcloud cdb deployment-groups-set-members my-pg member \
  --cpu 4

# Get current scaling config
ibmcloud cdb deployment-groups my-pg

# Check task status (scaling is async)
ibmcloud cdb tasks my-pg
```

### Backups

```bash
# List backups (point-in-time and on-demand)
ibmcloud cdb deployment-backups-list my-pg

# Trigger on-demand backup
ibmcloud cdb deployment-backup-create my-pg

# Restore from backup (creates a NEW instance)
ibmcloud cdb deployment-backup-restore $BACKUP_ID \
  --name my-pg-restored \
  --resource-group my-resource-group

# Set automatic backup schedule
ibmcloud cdb deployment-backup-schedule-set my-pg --hour 3 --minute 0
```

### PostgreSQL-Specific

```bash
# Connect via psql (use TLS cert from credentials)
PGPASSWORD="$PG_PASSWORD" psql \
  "sslmode=verify-ca sslrootcert=./pg-ca.crt \
   host=$PG_HOST port=$PG_PORT dbname=ibmclouddb user=admin"

# Python connection
pip install psycopg2-binary
```

```python
import psycopg2, ssl, json

creds = json.load(open("pg-creds.json"))["credentials"]["connection"]["postgres"]
conn = psycopg2.connect(
    host=creds["hosts"][0]["hostname"],
    port=creds["hosts"][0]["port"],
    database=creds["database"],
    user=creds["authentication"]["username"],
    password=creds["authentication"]["password"],
    sslmode="verify-ca",
    sslrootcert="pg-ca.crt",
)
```

---

## Db2 on Cloud

Db2 on Cloud is IBM's managed relational DB for SQL workloads with Db2 SQL dialect.

```bash
# Provision
ibmcloud resource service-instance-create my-db2 \
  dashdb-for-transactions \
  free \             # or performance-plus, flex
  us-south

# Get credentials
ibmcloud resource service-key-create my-db2-creds Manager \
  --instance-name my-db2
ibmcloud resource service-key my-db2-creds --output json | jq '.credentials'
```

```python
# Python connection via ibm_db
pip install ibm_db ibm_db_sa   # ibm_db_sa for SQLAlchemy

import ibm_db

conn_str = (
    f"DATABASE={db};"
    f"HOSTNAME={host};"
    f"PORT={port};"
    f"PROTOCOL=TCPIP;"
    f"UID={user};"
    f"PWD={password};"
    f"SECURITY=SSL;"
)
conn = ibm_db.connect(conn_str, "", "")
stmt = ibm_db.exec_immediate(conn, "SELECT CURRENT TIMESTAMP FROM SYSIBM.SYSDUMMY1")
row = ibm_db.fetch_assoc(stmt)
print(row)

# SQLAlchemy dialect
from sqlalchemy import create_engine
engine = create_engine(f"db2+ibm_db://{user}:{password}@{host}:{port}/{db}:SECURITY=SSL;")
```

---

## IBM Cloud Object Storage (COS)

COS is S3-compatible. Use AWS SDKs with custom endpoint, or the IBM COS SDK.

### Provisioning & Credentials

```bash
# Create COS instance (lite plan is free)
ibmcloud resource service-instance-create my-cos \
  cloud-object-storage \
  lite \
  global   # COS is global, not regional

# Create HMAC credentials (required for S3-compatible tools)
ibmcloud resource service-key-create my-cos-hmac Writer \
  --instance-name my-cos \
  -p '{"HMAC": true}'

# Extract HMAC keys
ibmcloud resource service-key my-cos-hmac --output json | \
  jq '.credentials.cos_hmac_keys'
# → { "access_key_id": "...", "secret_access_key": "..." }

# Non-HMAC (IAM-based — for SDK use)
ibmcloud resource service-key-create my-cos-iam Manager \
  --instance-name my-cos
```

### S3-Compatible CLI (AWS CLI pointed at COS)

```bash
# Configure AWS CLI profile for COS
aws configure --profile cos
# Access Key ID: <cos_hmac access_key_id>
# Secret Access Key: <cos_hmac secret_access_key>
# Default region: us-south (or us-east, eu-de, jp-tok, au-syd)

# Regional endpoints:
# us-south: s3.us-south.cloud-object-storage.appdomain.cloud
# us-east:  s3.us-east.cloud-object-storage.appdomain.cloud
# eu-de:    s3.eu-de.cloud-object-storage.appdomain.cloud

ENDPOINT="https://s3.us-south.cloud-object-storage.appdomain.cloud"

# Bucket operations
aws s3 --profile cos --endpoint-url $ENDPOINT mb s3://my-bucket
aws s3 --profile cos --endpoint-url $ENDPOINT ls
aws s3 --profile cos --endpoint-url $ENDPOINT ls s3://my-bucket/

# Upload / download
aws s3 --profile cos --endpoint-url $ENDPOINT cp local-file.txt s3://my-bucket/
aws s3 --profile cos --endpoint-url $ENDPOINT sync ./local-dir/ s3://my-bucket/prefix/
aws s3 --profile cos --endpoint-url $ENDPOINT cp s3://my-bucket/file.txt ./

# Presigned URL (7 days max)
aws s3 --profile cos --endpoint-url $ENDPOINT presign s3://my-bucket/file.txt \
  --expires-in 3600
```

### Python SDK (ibm-cos-sdk)

```bash
pip install ibm-cos-sdk
```

```python
import ibm_boto3
from ibm_botocore.client import Config

cos = ibm_boto3.client(
    "s3",
    ibm_api_key_id="YOUR_IBM_API_KEY",
    ibm_service_instance_id="YOUR_COS_CRN",   # from service credentials
    config=Config(signature_version="oauth"),
    endpoint_url="https://s3.us-south.cloud-object-storage.appdomain.cloud",
)

# List buckets
for b in cos.list_buckets()["Buckets"]:
    print(b["Name"])

# Upload file
cos.upload_file("local.txt", "my-bucket", "remote/path/file.txt")

# Upload large object (multipart automatic)
cos.upload_fileobj(
    open("bigfile.gz", "rb"),
    "my-bucket",
    "data/bigfile.gz",
    ExtraArgs={"ContentType": "application/gzip"},
)

# Download
cos.download_file("my-bucket", "remote/path/file.txt", "local-copy.txt")

# List objects with prefix
resp = cos.list_objects_v2(Bucket="my-bucket", Prefix="data/2026/")
for obj in resp.get("Contents", []):
    print(obj["Key"], obj["Size"])

# Generate presigned URL
url = cos.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-bucket", "Key": "file.txt"},
    ExpiresIn=3600,
)
```

### Bucket Lifecycle Policies

```python
cos.put_bucket_lifecycle_configuration(
    Bucket="my-bucket",
    LifecycleConfiguration={
        "Rules": [{
            "ID": "expire-old-logs",
            "Filter": {"Prefix": "logs/"},
            "Status": "Enabled",
            "Expiration": {"Days": 90},
        }]
    }
)
```

### Aspera High-Speed Transfer

For large datasets (>1 GB), Aspera provides 10–100x faster transfers than HTTP.

```bash
# Install IBM Aspera CLI
# Download from: https://www.ibm.com/products/aspera/downloads
pip install ibm-aspera-connect  # browser plugin; for CLI use ascp directly

# Transfer via ascp (after getting Aspera credentials from COS service key)
ascp -P 33001 -i <aspera_key_file> \
  --mode send \
  --host s3.us-south.cloud-object-storage.appdomain.cloud \
  --user iamapikey \
  localfile.tar.gz my-bucket/uploads/
```

---

## IBM Cloudant (CouchDB-Compatible)

```bash
# Provision
ibmcloud resource service-instance-create my-cloudant \
  cloudantnosqldb \
  lite \
  us-south

# Create credentials
ibmcloud resource service-key-create my-cloudant-creds Manager \
  --instance-name my-cloudant

# Extract URL (includes credentials embedded)
ibmcloud resource service-key my-cloudant-creds --output json | \
  jq -r '.credentials.url'
# → https://apikey-xxx:yyy@account.cloudantnosqldb.appdomain.cloud
```

```python
from ibmcloudant.cloudant_v1 import CloudantV1
from ibm_cloud_sdk_core.authenticators import IAMAuthenticator

authenticator = IAMAuthenticator(api_key="YOUR_API_KEY")
client = CloudantV1(authenticator=authenticator)
client.set_service_url("https://your-account.cloudantnosqldb.appdomain.cloud")

# Create database
client.put_database(db="my-database").get_result()

# Insert document
doc = {"_id": "doc1", "name": "Alice", "score": 42}
client.post_document(db="my-database", document=doc).get_result()

# Get document
resp = client.get_document(db="my-database", doc_id="doc1").get_result()
print(resp)

# Query (Mango)
result = client.post_find(
    db="my-database",
    selector={"score": {"$gt": 30}},
    fields=["_id", "name", "score"],
    limit=10,
).get_result()
for doc in result["docs"]:
    print(doc)

# Create index for Mango queries
client.post_index(
    db="my-database",
    index={"fields": [{"score": "asc"}]},
    name="score-index",
    type="json",
).get_result()
```

---

## IBM Event Streams (Kafka)

Event Streams is a managed Apache Kafka service.

```bash
# Provision
ibmcloud resource service-instance-create my-kafka \
  messagehub \
  standard \
  us-south   # or enterprise plan for dedicated

# Create credentials
ibmcloud resource service-key-create my-kafka-creds Manager \
  --instance-name my-kafka

# Extract broker list and API key
ibmcloud resource service-key my-kafka-creds --output json | jq '.credentials'
# → { "kafka_brokers_sasl": [...], "api_key": "...", "user": "token" }
```

### Admin CLI (ibmcloud event-streams plugin)

```bash
ibmcloud plugin install event-streams   # commands: ibmcloud es

# Initialize against your instance
ibmcloud es init --instance-name my-kafka

# List topics
ibmcloud es topics

# Create topic
ibmcloud es topic-create my-topic \
  --partitions 3 \
  --config retention.ms=86400000   # 24 hours

# Delete topic
ibmcloud es topic-delete my-topic -f

# List consumer groups
ibmcloud es groups
```

### Python (confluent-kafka)

```bash
pip install confluent-kafka
```

```python
from confluent_kafka import Producer, Consumer, KafkaError

brokers = ",".join(creds["kafka_brokers_sasl"])
sasl_config = {
    "bootstrap.servers": brokers,
    "security.protocol": "SASL_SSL",
    "sasl.mechanisms": "PLAIN",
    "sasl.username": "token",
    "sasl.password": creds["api_key"],
}

# Producer
producer = Producer(sasl_config)
producer.produce("my-topic", key="key1", value='{"event": "user_signup"}')
producer.flush()

# Consumer
consumer = Consumer({
    **sasl_config,
    "group.id": "my-consumer-group",
    "auto.offset.reset": "earliest",
})
consumer.subscribe(["my-topic"])
try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            raise Exception(msg.error())
        print(f"Received: {msg.key()}: {msg.value()}")
finally:
    consumer.close()
```

---

## Guardrails

- **ICD instances scale storage but never shrink** — over-provisioning disk is permanent; start conservatively.
- **COS HMAC keys vs IAM** — use HMAC for S3-compatible tools (AWS CLI, boto3); use IAM for IBM SDK. Don't mix them.
- **COS bucket names are globally unique** — across all IBM Cloud accounts; use a prefix like your account ID.
- **Cloudant Lite has 1 GB limit and 20 req/s** — upgrade to Standard before production.
- **Event Streams Standard = shared; Enterprise = dedicated** — topic isolation and throughput guarantees only on Enterprise.
- **ICD private endpoints** — always use private endpoints (VPE) in production; public endpoints add latency and risk.
- **Db2 free plan** is single-node, no HA — use Performance Plus for production.
- **Backup restores create a new instance** — plan for DNS/connection string updates in your app.
- **Event Streams max message size** is 1 MB by default — configure `message.max.bytes` at topic level if needed.
- **Cloudant replication** is built-in — use `_replicator` database for cross-region disaster recovery.
