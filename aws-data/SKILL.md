---
name: aws-data
description: AWS data services — RDS, Aurora, DynamoDB, S3, Redshift, ElastiCache, DocumentDB, Glue
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

# AWS Data Services — Comprehensive Reference

## RDS (Relational Database Service)

### Supported Engines

| Engine | Versions | Best for |
|---|---|---|
| MySQL | 8.0 | Web apps, WordPress |
| PostgreSQL | 15, 16 | General purpose, JSONB, extensions |
| MariaDB | 10.x | MySQL-compatible, community features |
| Oracle | 19c | Enterprise license portability |
| SQL Server | 2019, 2022 | Windows/Microsoft shops |

```bash
# Create PostgreSQL RDS instance
aws rds create-db-instance \
  --db-instance-identifier prod-postgres \
  --db-instance-class db.t3.medium \
  --engine postgres --engine-version 16.2 \
  --master-username dbadmin \
  --master-user-password "$(aws secretsmanager get-secret-value --secret-id db-master-pass --query SecretString --output text)" \
  --db-name myapp \
  --allocated-storage 100 --storage-type gp3 \
  --storage-encrypted --kms-key-id alias/aws/rds \
  --vpc-security-group-ids $SG \
  --db-subnet-group-name prod-db-subnet-group \
  --multi-az \
  --backup-retention-period 7 \
  --deletion-protection \
  --enable-performance-insights \
  --performance-insights-retention-period 7

# Wait for available
aws rds wait db-instance-available --db-instance-identifier prod-postgres

# Create read replica (same region)
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-read \
  --source-db-instance-identifier prod-postgres \
  --db-instance-class db.t3.medium

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres \
  --db-snapshot-identifier prod-postgres-snap-$(date +%Y%m%d)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier prod-postgres-snap-20240101
```

### Parameter Groups

```bash
# Create parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name prod-pg16-params \
  --db-parameter-group-family postgres16 \
  --description "Production PostgreSQL 16 params"

# Set parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-pg16-params \
  --parameters 'ParameterName=log_slow_statements,ParameterValue=all,ApplyMethod=pending-reboot' \
               'ParameterName=shared_buffers,ParameterValue={DBInstanceClassMemory/4},ApplyMethod=pending-reboot'
```

**Gotcha:** Static parameters require a reboot; dynamic parameters apply immediately with `ApplyMethod=immediate`.

---

## Aurora

### Aurora vs RDS

- **6-way replication** across 3 AZs in shared storage layer (no standby overhead)
- **Failover ~30s** (vs RDS Multi-AZ ~60-120s)
- **Storage auto-scales** up to 128 TB
- **Writer + up to 15 reader endpoints** — load balance reads automatically

```bash
# Create Aurora PostgreSQL cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora \
  --engine aurora-postgresql --engine-version 16.1 \
  --master-username dbadmin \
  --manage-master-user-password \   # secrets manager auto-rotation
  --db-subnet-group-name prod-db-subnet-group \
  --vpc-security-group-ids $SG \
  --backup-retention-period 14 \
  --deletion-protection \
  --enable-cloudwatch-logs-exports postgresql

# Add instance to cluster
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-1 \
  --db-cluster-identifier prod-aurora \
  --db-instance-class db.r6g.large \
  --engine aurora-postgresql
```

### Aurora Serverless v2

```bash
# Add serverless instance (scales 0.5 to 128 ACUs)
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-serverless \
  --db-cluster-identifier prod-aurora \
  --db-instance-class db.serverless \
  --engine aurora-postgresql

# Set scaling config on cluster
aws rds modify-db-cluster \
  --db-cluster-identifier prod-aurora \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16
```

### Global Database (Multi-Region)

```bash
aws rds create-global-cluster \
  --global-cluster-identifier my-global-db \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123:cluster:prod-aurora

# Add secondary region
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora-eu \
  --global-cluster-identifier my-global-db \
  --engine aurora-postgresql \
  --region eu-west-1
# RPO ~1s, RTO <1min for managed failover
```

---

## DynamoDB

### Key Concepts

- **Partition key** — hash key, distributes data. Choose high cardinality (user_id, order_id).
- **Sort key** — optional range key, enables range queries within a partition.
- **GSI** — Global Secondary Index: different PK/SK, eventual consistency, separate capacity.
- **LSI** — Local Secondary Index: same PK, different SK. Must be defined at table creation.
- **DAX** — DynamoDB Accelerator: in-memory cache, <1ms reads. Cluster, not serverless.

```bash
# Create table
aws dynamodb create-table \
  --table-name orders \
  --attribute-definitions \
    AttributeName=user_id,AttributeType=S \
    AttributeName=created_at,AttributeType=S \
    AttributeName=status,AttributeType=S \
  --key-schema \
    AttributeName=user_id,KeyType=HASH \
    AttributeName=created_at,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
    "IndexName": "status-created-index",
    "KeySchema": [
      {"AttributeName": "status","KeyType": "HASH"},
      {"AttributeName": "created_at","KeyType": "RANGE"}
    ],
    "Projection": {"ProjectionType": "ALL"}
  }]'

# Put item
aws dynamodb put-item \
  --table-name orders \
  --item '{"user_id":{"S":"u123"},"created_at":{"S":"2024-01-15T10:00:00Z"},"status":{"S":"pending"},"amount":{"N":"99.99"}}'

# Query (efficient — uses index)
aws dynamodb query \
  --table-name orders \
  --key-condition-expression "user_id = :uid AND created_at BETWEEN :start AND :end" \
  --expression-attribute-values '{":uid":{"S":"u123"},":start":{"S":"2024-01-01"},":end":{"S":"2024-12-31"}}'

# Query GSI
aws dynamodb query \
  --table-name orders \
  --index-name status-created-index \
  --key-condition-expression "#s = :status" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":status":{"S":"pending"}}'

# Conditional update (optimistic locking)
aws dynamodb update-item \
  --table-name orders \
  --key '{"user_id":{"S":"u123"},"created_at":{"S":"2024-01-15T10:00:00Z"}}' \
  --update-expression "SET #s = :new_status" \
  --condition-expression "#s = :expected" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":new_status":{"S":"shipped"},":expected":{"S":"pending"}}'
```

### Capacity Modes

```bash
# Switch to provisioned (for predictable workloads)
aws dynamodb update-table \
  --table-name orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50

# Enable auto-scaling (for provisioned mode)
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/orders" \
  --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --min-capacity 5 --max-capacity 1000
```

**Gotcha:** Hot partitions will throttle even with enough total capacity. Design for even distribution — avoid sequential IDs as partition keys.

---

## S3

### Storage Classes

| Class | Retrieval | Min Duration | Use case |
|---|---|---|---|
| Standard | Instant | None | Hot data |
| Standard-IA | Instant | 30 days | Infrequent (>30d) |
| One Zone-IA | Instant | 30 days | Non-critical, recreatable |
| Glacier Instant | Instant | 90 days | Archives with instant retrieval |
| Glacier Flexible | 1-12h | 90 days | Long archives |
| Glacier Deep Archive | 12-48h | 180 days | 7-year compliance archives |
| Intelligent-Tiering | Varies | None | Unknown access patterns |

```bash
# Lifecycle rule (archive after 30d, expire after 365d)
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-old",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"}
      ],
      "Expiration": {"Days": 365}
    }]
  }'

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Cross-region replication
aws s3api put-bucket-replication \
  --bucket my-bucket \
  --replication-configuration file://replication.json

# Presigned URL (24h)
aws s3 presign s3://my-bucket/private/report.pdf --expires-in 86400

# Block all public access (best practice)
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Server-side encryption default
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]
  }'
```

---

## Redshift

```bash
# Redshift Serverless (no cluster management)
aws redshift-serverless create-namespace \
  --namespace-name prod-ns \
  --admin-username admin \
  --manage-admin-password

aws redshift-serverless create-workgroup \
  --workgroup-name prod-wg \
  --namespace-name prod-ns \
  --base-capacity 32 \          # RPUs (Redshift Processing Units)
  --subnet-ids <priv-1a> <priv-1b> \
  --security-group-ids $SG

# Query via Data API (no JDBC needed)
aws redshift-data execute-statement \
  --workgroup-name prod-wg \
  --database mydb \
  --sql "SELECT COUNT(*) FROM events WHERE date >= '2024-01-01'"
```

---

## ElastiCache

```bash
# Redis cluster (cluster mode disabled = single shard, simpler)
aws elasticache create-replication-group \
  --replication-group-id prod-redis \
  --description "Prod Redis" \
  --cache-node-type cache.r6g.large \
  --engine redis --engine-version 7.1 \
  --num-cache-clusters 2 \           # primary + 1 replica
  --automatic-failover-enabled \
  --cache-subnet-group-name prod-cache-subnet \
  --security-group-ids $SG \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "$(openssl rand -base64 32)"
```

**Redis vs Memcached:**
- Redis: persistence, replication, pub/sub, sorted sets, Lua scripts, cluster mode
- Memcached: simpler, multi-threaded, slightly faster for pure caching

---

## AWS Glue

```bash
# Create crawler (discovers schema)
aws glue create-crawler \
  --name s3-logs-crawler \
  --role arn:aws:iam::123:role/GlueCrawlerRole \
  --database-name logs_db \
  --targets S3Targets=[{Path="s3://my-bucket/logs/"}] \
  --schedule "cron(0 2 * * ? *)"

# Run crawler
aws glue start-crawler --name s3-logs-crawler

# Create ETL job (Spark-based)
aws glue create-job \
  --name transform-logs \
  --role arn:aws:iam::123:role/GlueJobRole \
  --command Name=glueetl,ScriptLocation=s3://my-bucket/scripts/transform.py,PythonVersion=3 \
  --glue-version 4.0 \
  --worker-type G.1X --number-of-workers 10 \
  --default-arguments '{"--enable-job-bookmarks": "enable"}'

aws glue start-job-run --job-name transform-logs
```

---

## Guardrails & Gotchas

- **DynamoDB Scan** — scans entire table, extremely expensive at scale. Always use Query or GSI.
- **S3 eventual consistency** — S3 is now strongly consistent for all operations, but replicated buckets may lag.
- **RDS deletion protection** — enable in prod; `aws rds modify-db-instance --deletion-protection` won't apply until next maintenance unless `--apply-immediately`.
- **Aurora Serverless v2 minimum ACU = 0.5** — it doesn't scale to zero (use on-demand for true zero).
- **DynamoDB item limit** — 400 KB max per item. Split large items or use S3 + pointer pattern.
- **ElastiCache AUTH token** — set at creation, can't be changed without recreation. Store in Secrets Manager.
- **Redshift Spectrum** — query S3 directly from Redshift without loading data. Use for infrequent historical queries.
- **S3 Intelligent-Tiering** — adds $0.0025/1000 objects monitoring fee. Not worth it for <128 KB objects.
- **RDS Multi-AZ** — synchronous replication to standby (not readable). For read scaling, use read replicas.
- **Glue job bookmarks** — critical for incremental processing. Without them, jobs reprocess all data on each run.
