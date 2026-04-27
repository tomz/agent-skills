---
name: azure-hdinsight
description: Azure HDInsight — managed Hadoop ecosystem (Spark, Hive, HBase, Kafka, Storm), Enterprise Security Package, autoscale, and HDInsight on AKS
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

# Azure HDInsight

Azure HDInsight is a fully managed, open-source analytics service hosting the Hadoop
ecosystem on Azure. It runs Apache Spark, Hive (with LLAP), HBase, Kafka, Storm, and
Interactive Query — each as separately provisioned cluster types.

**Strategic note**: Microsoft is positioning HDInsight as a migration source toward
Microsoft Fabric and Azure Databricks. New projects should strongly consider these
alternatives. HDInsight on AKS is the newer trajectory for open-source workloads.

---

## Cluster Types

| Cluster Type       | Components                          | Typical Use Case                 |
|--------------------|-------------------------------------|----------------------------------|
| Hadoop             | HDFS, YARN, MapReduce, Hive, Pig    | Batch ETL (legacy)               |
| Spark              | Spark, Hive, Livy, Jupyter, Zeppelin| Data engineering, ML             |
| Interactive Query  | LLAP, Hive, Tez                     | Low-latency analytical SQL       |
| HBase              | HBase, Phoenix, ZooKeeper           | NoSQL wide-column store          |
| Kafka              | Kafka, ZooKeeper, MirrorMaker       | Event streaming                  |
| Storm              | Storm, ZooKeeper                    | Real-time event processing       |

### HDInsight 5.x vs HDInsight on AKS

| Feature              | HDInsight 5.x                    | HDInsight on AKS                      |
|----------------------|----------------------------------|---------------------------------------|
| Infra                | IaaS VMs                         | AKS (Kubernetes)                      |
| Cluster types        | All classic types                | Spark, Trino, Flink (others roadmap)  |
| Scaling              | Manual / autoscale               | AKS node pools (auto)                 |
| Network              | VNet injection                   | AKS VNet integration                  |
| Auth                 | SSH, LDAP, Kerberos (ESP)        | Entra ID / Azure AD workload identity |
| Status               | Generally Available              | GA (as of 2024)                       |

---

## Cluster Creation (HDInsight 5.x)

### az CLI
```bash
# Register provider
az provider register --namespace Microsoft.HDInsight

# Create storage account (ADLS Gen2 — recommended over Blob for HDInsight 4.0+)
az storage account create \
  --name myhdistorage \
  --resource-group hdi-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true

# Get storage key
STORAGE_KEY=$(az storage account keys list \
  --account-name myhdistorage \
  --resource-group hdi-rg \
  --query '[0].value' -o tsv)

# Create Spark cluster (minimal)
az hdinsight create \
  --name my-spark-cluster \
  --resource-group hdi-rg \
  --location eastus \
  --cluster-type Spark \
  --component-version Spark=3.3 \
  --http-password "YourP@ssw0rd!" \
  --http-user admin \
  --workernode-count 3 \
  --workernode-size Standard_D4_v2 \
  --headnode-size Standard_D4_v2 \
  --ssh-password "YourSSHP@ssw0rd!" \
  --ssh-user sshuser \
  --storage-account myhdistorage \
  --storage-account-key "$STORAGE_KEY" \
  --storage-container myhdicontainer

# Create Kafka cluster with specific storage
az hdinsight create \
  --name my-kafka-cluster \
  --resource-group hdi-rg \
  --cluster-type Kafka \
  --component-version Kafka=2.4 \
  --workernode-count 3 \
  --workernode-size Standard_D4_v2 \
  --zookeepernode-size Standard_A4_v2 \
  --storage-account myhdistorage \
  --storage-account-key "$STORAGE_KEY" \
  --storage-container kafka-container \
  --http-user admin \
  --http-password "YourP@ssw0rd!"

# Monitor creation (takes 15-25 minutes)
az hdinsight show \
  --name my-spark-cluster \
  --resource-group hdi-rg \
  --query 'properties.clusterState' -o tsv

# Wait until state = "Running"
az hdinsight wait \
  --name my-spark-cluster \
  --resource-group hdi-rg \
  --created
```

**Gotcha**: Cluster creation takes **20-30 minutes**. There is no way to speed this up.
Script cluster setup and let it run — do not wait interactively.

### SSH Access
```bash
# SSH to head node 0
ssh sshuser@my-spark-cluster-ssh.azurehdinsight.net

# SSH tunnel for Ambari/Spark UI (port forwarding)
ssh -L 8080:headnodehost:8080 \
    -L 8088:headnodehost:8088 \
    -L 18080:headnodehost:18080 \
    sshuser@my-spark-cluster-ssh.azurehdinsight.net -N
# Then browse: http://localhost:8080 (Ambari), http://localhost:8088 (YARN), 
#              http://localhost:18080 (Spark History)
```

---

## Spark on HDInsight

### spark-submit
```bash
# From head node (after SSH)
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 4 \
  --executor-cores 2 \
  --executor-memory 4g \
  --driver-memory 2g \
  --conf spark.yarn.submit.waitAppCompletion=true \
  --jars /usr/hdp/current/spark2-client/jars/azure-storage-*.jar \
  wasbs://container@myhdistorage.blob.core.windows.net/jars/myapp.jar \
  arg1 arg2

# PySpark job
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --py-files wasbs://container@storage.blob.core.windows.net/deps.zip \
  wasbs://container@storage.blob.core.windows.net/scripts/my_job.py \
  --date 2024-01-15
```

### Livy REST API (Remote Spark Submission)
```bash
# Submit a batch job via Livy (no SSH required)
curl -s -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  https://my-spark-cluster.azurehdinsight.net/livy/batches \
  -d '{
    "file": "wasbs://container@storage.blob.core.windows.net/scripts/my_job.py",
    "args": ["--date", "2024-01-15"],
    "numExecutors": 4,
    "executorMemory": "4g",
    "executorCores": 2,
    "driverMemory": "2g",
    "conf": {
      "spark.sql.shuffle.partitions": "200"
    }
  }'

# Returns {"id": 42, "state": "starting", ...}
BATCH_ID=42

# Poll status
curl -s -u admin:password \
  https://my-spark-cluster.azurehdinsight.net/livy/batches/$BATCH_ID \
  | python3 -m json.tool | grep '"state"'

# Fetch logs
curl -s -u admin:password \
  "https://my-spark-cluster.azurehdinsight.net/livy/batches/$BATCH_ID/log?from=0&size=50" \
  | python3 -c "import sys,json; [print(l) for l in json.load(sys.stdin)['log']]"
```

### PySpark on ADLS Gen2
```python
# Configure ADLS Gen2 access in Spark (if not using cluster-level config)
spark.conf.set(
    "fs.azure.account.auth.type.myaccount.dfs.core.windows.net", "OAuth"
)
spark.conf.set(
    "fs.azure.account.oauth.provider.type.myaccount.dfs.core.windows.net",
    "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider"
)
spark.conf.set(
    "fs.azure.account.oauth2.client.id.myaccount.dfs.core.windows.net",
    "<app-id>"
)
spark.conf.set(
    "fs.azure.account.oauth2.client.secret.myaccount.dfs.core.windows.net",
    "<client-secret>"
)
spark.conf.set(
    "fs.azure.account.oauth2.client.endpoint.myaccount.dfs.core.windows.net",
    "https://login.microsoftonline.com/<tenant-id>/oauth2/token"
)

# Read / write
df = spark.read.parquet("abfss://container@myaccount.dfs.core.windows.net/path/")
df.write.mode("overwrite").parquet("abfss://container@myaccount.dfs.core.windows.net/out/")
```

---

## Hive / Interactive Query (LLAP)

Interactive Query uses the **LLAP (Live Long and Process)** daemon for sub-second query
latency via persistent in-memory execution daemons.

### Beeline (HiveServer2)
```bash
# Connect via Beeline (from SSH or local with JDBC tunnel)
beeline -u "jdbc:hive2://localhost:10001/;transportMode=http;httpPath=cliservice" \
        -n admin \
        -p password

# Or via HTTPS (from outside cluster)
beeline -u "jdbc:hive2://my-iq-cluster.azurehdinsight.net:443/;ssl=true;transportMode=http;httpPath=/hive2" \
        -n admin \
        -p password
```

### Hive DDL & ACID Tables
```sql
-- Enable ACID (required for Interactive Query LLAP)
-- cluster-level config; usually pre-set in HDInsight

-- Create ACID ORC table (transactional)
CREATE TABLE IF NOT EXISTS transactions (
    txn_id     BIGINT,
    account_id BIGINT,
    amount     DECIMAL(18,2),
    txn_date   DATE
)
CLUSTERED BY (account_id) INTO 16 BUCKETS
STORED AS ORC
TBLPROPERTIES (
    "transactional" = "true",
    "compactor.mapreduce.map.memory.mb" = "2048"
);

-- MERGE (Hive 3.x+)
MERGE INTO transactions AS t
USING transactions_staging AS s
ON (t.txn_id = s.txn_id)
WHEN MATCHED THEN UPDATE SET amount = s.amount, txn_date = s.txn_date
WHEN NOT MATCHED THEN INSERT VALUES (s.txn_id, s.account_id, s.amount, s.txn_date);

-- Partition and bucket for performance
CREATE TABLE sales_partitioned (
    sale_id    BIGINT,
    product_id INT,
    amount     DECIMAL(18,2)
)
PARTITIONED BY (sale_date STRING)
CLUSTERED BY (product_id) INTO 8 BUCKETS
STORED AS ORC TBLPROPERTIES ("transactional" = "true");

-- Load with dynamic partitioning
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;

INSERT INTO sales_partitioned PARTITION (sale_date)
SELECT sale_id, product_id, amount, FORMAT_DATE(sale_date, 'yyyy-MM-dd')
FROM sales_staging;

-- Compaction (merge delta files for ACID tables)
ALTER TABLE transactions COMPACT 'MAJOR';
SHOW COMPACTIONS;
```

---

## Kafka on HDInsight

### Topic Management
```bash
# SSH to broker node
ssh sshuser@my-kafka-cluster-ssh.azurehdinsight.net

# Get ZooKeeper host list (Kafka 2.x — older style)
ZOOKEEPER=$(grep -i zookeeper /etc/kafka/conf/kafka.properties | head -1)

# Get broker list (from Ambari API)
BROKERS=$(curl -s -u admin:password \
  "https://my-kafka-cluster.azurehdinsight.net/api/v1/clusters/my-kafka-cluster/services/KAFKA/components/KAFKA_BROKER" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(','.join([h['HostRoles']['host_name']+':9092' for h in d['host_components']]))")

# Create topic
/usr/hdp/current/kafka-broker/bin/kafka-topics.sh \
  --bootstrap-server $BROKERS \
  --create \
  --topic my-topic \
  --partitions 6 \
  --replication-factor 3

# List topics
/usr/hdp/current/kafka-broker/bin/kafka-topics.sh \
  --bootstrap-server $BROKERS \
  --list

# Describe topic (check ISR — in-sync replicas)
/usr/hdp/current/kafka-broker/bin/kafka-topics.sh \
  --bootstrap-server $BROKERS \
  --describe \
  --topic my-topic

# Produce test messages
echo "test message 1" | /usr/hdp/current/kafka-broker/bin/kafka-console-producer.sh \
  --bootstrap-server $BROKERS \
  --topic my-topic

# Consume (from beginning)
/usr/hdp/current/kafka-broker/bin/kafka-console-consumer.sh \
  --bootstrap-server $BROKERS \
  --topic my-topic \
  --from-beginning \
  --max-messages 10

# Consumer group offset management
/usr/hdp/current/kafka-broker/bin/kafka-consumer-groups.sh \
  --bootstrap-server $BROKERS \
  --describe \
  --group my-consumer-group

# Reset offset (re-process from beginning)
/usr/hdp/current/kafka-broker/bin/kafka-consumer-groups.sh \
  --bootstrap-server $BROKERS \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --to-earliest \
  --execute
```

### MirrorMaker (Cross-Cluster Replication)
```bash
# MirrorMaker 2 config (recommended over MM1)
# consumer.properties
bootstrap.servers=source-broker1:9092,source-broker2:9092
group.id=mirrormaker

# producer.properties
bootstrap.servers=target-broker1:9092,target-broker2:9092

# Run MirrorMaker 2
/usr/hdp/current/kafka-broker/bin/kafka-mirror-maker.sh \
  --producer.config producer.properties \
  --consumer.config consumer.properties \
  --whitelist "my-topic|other-topic" \
  --num.streams 2
```

---

## HBase on HDInsight

### HBase Shell
```bash
# Open shell
hbase shell

# Create table with column families
create 'sensor_data', {NAME => 'cf', VERSIONS => 3, COMPRESSION => 'SNAPPY'}
create 'user_profiles', {NAME => 'info', VERSIONS => 1}, {NAME => 'activity', VERSIONS => 10}

# Put (insert/update)
put 'sensor_data', 'device001|20240115T120000', 'cf:temperature', '22.5'
put 'sensor_data', 'device001|20240115T120000', 'cf:humidity', '65.2'

# Get a single row
get 'sensor_data', 'device001|20240115T120000'
get 'sensor_data', 'device001|20240115T120000', {COLUMN => 'cf:temperature', VERSIONS => 3}

# Scan with prefix filter (row key design is critical!)
scan 'sensor_data', {STARTROW => 'device001|', STOPROW => 'device001|~', LIMIT => 100}

# Delete
deleteall 'sensor_data', 'device001|20240115T120000'

# Admin
list
describe 'sensor_data'
count 'sensor_data'           # slow for large tables; use YARN job instead
alter 'sensor_data', {NAME => 'cf', COMPRESSION => 'SNAPPY'}
major_compact 'sensor_data'   # trigger compaction

# Flush and compact
flush 'sensor_data'
```

### Phoenix SQL over HBase
```bash
# Start Phoenix shell (from head node)
/usr/hdp/current/phoenix-client/bin/sqlline.py localhost:2181:/hbase-unsecure

# Create Phoenix table on existing HBase table
CREATE TABLE IF NOT EXISTS SENSOR_DATA (
    ROW_KEY VARCHAR PRIMARY KEY,
    "cf"."temperature" FLOAT,
    "cf"."humidity"    FLOAT
) COLUMN_ENCODED_BYTES=0;

-- Standard SQL
SELECT row_key, "cf"."temperature"
FROM SENSOR_DATA
WHERE row_key LIKE 'device001%'
ORDER BY row_key DESC
LIMIT 100;

-- Upsert
UPSERT INTO SENSOR_DATA VALUES ('device001|20240115T120000', 22.5, 65.2);
```

### HBase REST Gateway
```bash
# Query via REST (JSON)
curl -s -X GET \
  -H "Accept: application/json" \
  "https://my-hbase-cluster.azurehdinsight.net/hbaserest/sensor_data/device001%7C20240115T120000" \
  -u admin:password | python3 -m json.tool
```

---

## Security: Enterprise Security Package (ESP)

ESP enables Active Directory integration, Kerberos, Ranger authorization, and Auditing.

### Prerequisites
- Azure AD Domain Services (AAD DS) provisioned in the same VNet (or peered VNet)
- Cluster VNet must have DNS pointing to AAD DS controllers
- Service account with AAD DS join permissions
- Ranger admin account

```bash
# Create ESP-enabled Spark cluster
az hdinsight create \
  --name my-esp-cluster \
  --resource-group hdi-rg \
  --cluster-type Spark \
  --esp \
  --domain /subscriptions/$SUB/resourceGroups/aadds-rg/providers/Microsoft.AAD/domainServices/mydomain.onmicrosoft.com \
  --domain-user-name "hdiadmin@mydomain.onmicrosoft.com" \
  --domain-password "DomainP@ss!" \
  --cluster-users-group-dns "HDInsightUsers" \
  --assign-identity /subscriptions/$SUB/resourceGroups/hdi-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/hdi-identity \
  --storage-account myhdistorage \
  --http-user admin \
  --http-password "AdminP@ss!"
```

### Ranger Policies (Hive example)
```
Ranger Policy: Allow analysts group to SELECT on database "sales", tables "*", columns "*"
Deny: columns matching "ssn|credit_card|email" for non-PII group
```
Configure via Ranger UI at: `https://<cluster>.azurehdinsight.net/ranger/`

**Gotcha — ESP complexity**: ESP clusters take significantly longer to provision (45-60 min),
are harder to debug (Kerberos ticket issues), and require AAD DS — which is expensive
(~$100/month) and complex to set up. Consider whether your org's Entra ID capabilities
are sufficient before committing.

---

## Autoscale

### Schedule-Based Autoscale
Scale to known capacity at fixed times (e.g., more workers during business hours):
```bash
az hdinsight autoscale create \
  --resource-group hdi-rg \
  --cluster-name my-spark-cluster \
  --type Schedule \
  --timezone "Pacific Standard Time" \
  --schedule '{"days":["Monday","Tuesday","Wednesday","Thursday","Friday"],"timeAndCapacity":{"time":"08:00","minInstanceCount":6,"maxInstanceCount":6}}' \
  --schedule '{"days":["Monday","Tuesday","Wednesday","Thursday","Friday"],"timeAndCapacity":{"time":"18:00","minInstanceCount":2,"maxInstanceCount":2}}'
```

### Load-Based Autoscale
```bash
az hdinsight autoscale create \
  --resource-group hdi-rg \
  --cluster-name my-spark-cluster \
  --type Load \
  --min-workernode-count 2 \
  --max-workernode-count 10
```

### Graceful Decommission
When scaling down, HDInsight uses YARN graceful decommission to let running containers
finish before removing the node (up to 60 minutes by default). Configure:
```bash
az hdinsight autoscale update \
  --resource-group hdi-rg \
  --cluster-name my-spark-cluster \
  --graceful-decommission-timeout 120   # minutes; -1 = unlimited
```

---

## HDInsight on AKS

HDInsight on AKS (HIoA) is the next-generation platform built on Kubernetes.

### Key Concepts
- **Cluster Pool**: AKS cluster that hosts multiple HIoA clusters (shared node pools)
- **HIoA Cluster**: Trino, Flink, or Spark cluster provisioned inside a pool
- No SSH by default — managed via Azure Portal / REST API / ARM

### Create Cluster Pool & Cluster
```bash
# Register provider
az provider register --namespace Microsoft.HDInsight

# Create cluster pool (AKS-backed)
az hdinsight-on-aks pool create \
  --cluster-pool-name my-hdi-pool \
  --resource-group hdi-rg \
  --location eastus \
  --cluster-pool-version "1.1" \
  --vm-size Standard_D4ds_v5

# Create Trino cluster in pool
az hdinsight-on-aks cluster create \
  --cluster-pool-name my-hdi-pool \
  --resource-group hdi-rg \
  --cluster-name my-trino \
  --cluster-type Trino \
  --cluster-version "1.1.1" \
  --nodes '[{"type":"Worker","vmSize":"Standard_D8ds_v5","count":3}]'

# Create Spark cluster in pool
az hdinsight-on-aks cluster create \
  --cluster-pool-name my-hdi-pool \
  --resource-group hdi-rg \
  --cluster-name my-spark \
  --cluster-type Spark \
  --cluster-version "1.1.1" \
  --nodes '[{"type":"Worker","vmSize":"Standard_D4ds_v5","count":3}]'
```

### Trino Query Pattern
```bash
# Connect to Trino coordinator (via AKS service endpoint)
trino --server https://<trino-endpoint>:443 \
      --user me@example.com \
      --password

# Trino SQL — query ADLS Gen2
SELECT region, SUM(amount)
FROM hive.sales.orders     -- hive catalog backed by ADLS
WHERE month = '2024-01'
GROUP BY region;

-- Cross-catalog query
SELECT * FROM hive.sales.orders o
JOIN iceberg.ml_catalog.predictions p ON o.order_id = p.order_id;
```

---

## Common Operational Tasks

```bash
# List all HDInsight clusters in a subscription
az hdinsight list --output table

# Resize a cluster (manual scale)
az hdinsight resize \
  --name my-spark-cluster \
  --resource-group hdi-rg \
  --workernode-count 5

# Delete a cluster (does NOT delete storage — data persists in ADLS)
az hdinsight delete \
  --name my-spark-cluster \
  --resource-group hdi-rg

# Script action (run custom bash script on cluster nodes)
az hdinsight script-action execute \
  --cluster-name my-spark-cluster \
  --resource-group hdi-rg \
  --name install-custom-libs \
  --roles headnode workernode \
  --script-uri "https://myaccount.blob.core.windows.net/scripts/install_libs.sh" \
  --persist-on-success
```

---

## Gotchas & Operational Tips

1. **Cluster creation takes 20+ minutes**: Always automate cluster creation via ARM
   templates or CLI scripts. Do not spin up clusters interactively if you need them fast.

2. **Head node costs even when idle**: Unlike Databricks/Synapse, HDInsight has no
   "pause" for Spark clusters — head nodes run 24/7. For intermittent workloads, delete
   the cluster and recreate it (data persists in ADLS).

3. **Storage is separate from compute**: All data should be in ADLS Gen2 or Azure Blob,
   NOT in HDFS. Clusters are ephemeral; local HDFS is lost on cluster delete.

4. **ESP complexity is significant**: AAD DS setup, VNet peering, DNS configuration,
   and Ranger policies all need to be right before ESP clusters work. Budget 2-3 days
   of setup time for a production ESP environment.

5. **Kafka head node memory pressure**: The HDInsight Kafka cluster head nodes run
   Ambari, ZooKeeper, and broker management services. Undersized head nodes (< D4) cause
   stability issues. Use D4 or larger even for small Kafka clusters.

6. **HBase row key design is everything**: HBase is a sorted key-value store. Without
   careful row key design (prefix for scan locality, hash/salt for write distribution),
   you get hot regions and poor performance. Design row keys before schema creation.

7. **LLAP cache sizing**: Interactive Query performance depends on the LLAP daemon's
   in-memory cache. Tune `hive.llap.io.memory.size` to fill 70-80% of available worker
   node memory. Ambari's config wizard often sets it too conservatively.

8. **MirrorMaker offset translation**: When using MirrorMaker across clusters, consumer
   group offsets do NOT automatically translate. Consumers need to use
   `RemoteClusterAlias.ConsumerGroup` naming or reset offsets after cutover.

9. **Migration path**: Microsoft recommends migrating HDInsight Spark → Azure Databricks
   or Microsoft Fabric, HDInsight Interactive Query → Synapse Serverless SQL or Fabric
   Warehouse, HDInsight Kafka → Azure Event Hubs (Kafka-compatible). Plan migrations
   before HDInsight 5.x end-of-support dates.

10. **HDInsight on AKS startup**: HIoA Spark/Trino/Flink clusters still take 10-15 min
    to provision but can be stopped/started faster than classic HDInsight. Cluster pool
    stays running (billed for AKS system node pool) even when no HIoA clusters are active.
