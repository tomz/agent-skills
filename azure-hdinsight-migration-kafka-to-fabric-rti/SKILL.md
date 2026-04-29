---
name: azure-hdinsight-migration-kafka-to-fabric-rti
description: End-to-end migration playbook for moving Apache Kafka workloads from Azure HDInsight Kafka clusters to Microsoft Fabric Real-Time Intelligence (Eventstreams, Eventhouse/KQL DB, Activator, Real-Time Dashboards) and/or Azure Event Hubs Kafka endpoint — assessment, topic + consumer-group inventory, schema/serde migration, MirrorMaker 2 cutover, producer/consumer code rewrites, security mapping (ESP/Kerberos → Entra ID + Managed Identity), KQL ingestion mappings, validation.
license: MIT
version: 1.0.0
updated: 2026-04-29
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
triggers: hdinsight kafka, kafka migration, fabric rti, real-time intelligence, eventstream, eventhouse, kql, mirrormaker, schema registry, event hubs kafka, kafka to fabric, hdi kafka
requires: azure-hdinsight, azure-fabric
---
# Migrating HDInsight Kafka → Microsoft Fabric Real-Time Intelligence

A field guide for migrating production Apache Kafka workloads from **Azure HDInsight
Kafka** clusters to **Microsoft Fabric Real-Time Intelligence (RTI)** — Eventstreams,
Eventhouse / KQL Database, Activator, and Real-Time Dashboards — with **Azure Event
Hubs (Kafka endpoint)** as the recommended message-bus replacement when downstream
producers/consumers must keep speaking Kafka protocol.

> **Why this migration?** HDInsight Kafka 4.0 reached end-of-standard support; 5.x is
> the last classic generation. Microsoft's recommended landing zones are:
> - **Azure Event Hubs (Kafka endpoint)** — drop-in for the broker tier (keeps Kafka clients).
> - **Fabric Eventstream** — managed ingestion + routing + transformation (no broker management).
> - **Fabric Eventhouse / KQL DB** — analytics + storage tier (replaces Spark/Hive on stream).
> - **Fabric Activator** — alerting / triggers (replaces custom Storm/Spark Streaming alerters).

---

## 1. Decision Matrix — Which Fabric Path?

| Existing HDI Kafka Pattern | Recommended Target | Reason |
|----------------------------|-------------------|--------|
| Pure broker (apps produce + consume) | **Event Hubs Kafka endpoint** | Drop-in; clients keep `bootstrap.servers` semantics |
| Kafka → HDI Spark Streaming → ADLS/Hive | **Eventstream → Eventhouse (KQL)** | Native, no Spark cluster needed |
| Kafka → custom alerter / webhook | **Eventstream → Activator** | No-code rules + Teams/Email/HTTP outputs |
| Kafka → Power BI streaming dataset | **Eventstream → KQL DB → Direct Lake / Direct Query** | Sub-second dashboards |
| Kafka with MirrorMaker (DC replication) | **Event Hubs geo-DR** | Built-in; or keep MM2 with EH Kafka endpoint |
| Kafka Connect (CDC, S3 sink, etc.) | **Eventstream sources/destinations** OR keep Kafka Connect on AKS | Eventstream covers ~80% of common connectors |
| ksqlDB / kSQL transformations | **Eventstream Event Processor (drag-drop)** OR **KQL update policies** | Re-write logic |
| Kafka Streams apps (Java) | **Fabric Spark Structured Streaming** OR keep on AKS | No Kafka Streams equivalent in Fabric |
| Schema Registry (Confluent) | **Self-host SR on AKS** OR embed schema in payload | Fabric has no managed Schema Registry |
| Heavy throughput (>1 GB/s sustained) | **Event Hubs Premium / Dedicated** | Fabric Eventstream has CU-based throughput limits |
| Strict on-prem / air-gapped | **Stay on Kafka (managed via Confluent Platform on AKS)** | Fabric is SaaS only |

**Decision shortcut**: If the workload's downstream is analytics + dashboards in the
Fabric/Power BI ecosystem, go full **Eventstream → Eventhouse**. If it's an event bus
serving many polyglot clients, go **Event Hubs Kafka endpoint** and bridge selected
topics into Fabric Eventstream for analytics.

---

## 2. Assessment Phase

### 2.1 Inventory the HDInsight Kafka Cluster

Run on a head/broker node (SSH).

```bash
ssh sshuser@<cluster>-ssh.azurehdinsight.net

# Cluster + broker metadata
curl -s -u admin:$PASS \
  "https://<cluster>.azurehdinsight.net/api/v1/clusters/<cluster>" \
  | python3 -m json.tool > cluster_meta.json

# Broker list (resolve once, used everywhere)
BROKERS=$(curl -s -u admin:$PASS \
  "https://<cluster>.azurehdinsight.net/api/v1/clusters/<cluster>/services/KAFKA/components/KAFKA_BROKER" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(','.join([h['HostRoles']['host_name']+':9092' for h in d['host_components']]))")
echo $BROKERS > brokers.txt

KBIN=/usr/hdp/current/kafka-broker/bin

# Kafka version
$KBIN/kafka-topics.sh --version | tee kafka_version.txt

# All topics + per-topic config
$KBIN/kafka-topics.sh --bootstrap-server $BROKERS --list > topics.txt
while read t; do
  $KBIN/kafka-topics.sh --bootstrap-server $BROKERS --describe --topic "$t" \
    >> topic_describe.txt
done < topics.txt

# Topic-level configs (retention, compaction, etc.)
$KBIN/kafka-configs.sh --bootstrap-server $BROKERS --entity-type topics --describe \
  > topic_configs.txt

# Broker-level configs
$KBIN/kafka-configs.sh --bootstrap-server $BROKERS --entity-type brokers --describe \
  > broker_configs.txt

# Consumer groups + lag
$KBIN/kafka-consumer-groups.sh --bootstrap-server $BROKERS --list > consumer_groups.txt
while read g; do
  echo "=== $g ===" >> consumer_groups_describe.txt
  $KBIN/kafka-consumer-groups.sh --bootstrap-server $BROKERS \
    --describe --group "$g" >> consumer_groups_describe.txt
done < consumer_groups.txt

# ACLs (if SASL/SSL enabled — ESP clusters)
$KBIN/kafka-acls.sh --bootstrap-server $BROKERS --list > acls.txt 2>/dev/null || true

# Per-topic size + message count estimate
$KBIN/kafka-log-dirs.sh --bootstrap-server $BROKERS --describe \
  | python3 -c "
import sys, json, re
data = ''.join(l for l in sys.stdin if l.startswith('{'))
d = json.loads(data)
totals = {}
for b in d['brokers']:
  for ld in b['logDirs']:
    for p in ld['partitions']:
      topic = p['partition'].rsplit('-', 1)[0]
      totals[topic] = totals.get(topic, 0) + p['size']
for t, s in sorted(totals.items(), key=lambda x: -x[1]):
  print(f'{s/1e9:8.2f} GB  {t}')
" > topic_sizes.txt
```

### 2.2 Inventory the Producer / Consumer Codebase

```bash
cd /path/to/repos

# 1) Bootstrap server / connection strings
grep -RInE 'bootstrap[_.-]?servers|kafka\.host|broker[_.-]?list' \
  --include='*.py' --include='*.java' --include='*.scala' --include='*.go' \
  --include='*.js' --include='*.ts' --include='*.cs' --include='*.yaml' --include='*.yml'

# 2) Auth mechanisms in use
grep -RInE 'security\.protocol|sasl\.mechanism|sasl\.jaas\.config|kerberos|GSSAPI|PLAIN|SCRAM' \
  --include='*.properties' --include='*.conf' --include='*.py' --include='*.java'

# 3) Schema Registry usage (Confluent / Apicurio)
grep -RInE 'schema[_.-]?registry|confluent[_.-]?serializer|KafkaAvroSerializer|apicurio' \
  --include='*.py' --include='*.java' --include='*.scala' --include='*.properties'

# 4) Kafka Streams / ksqlDB
grep -RInE 'org\.apache\.kafka\.streams|StreamsBuilder|ksql' \
  --include='*.java' --include='*.scala' --include='*.sql'

# 5) Kafka Connect connectors
find . -name 'connect-*.properties' -o -name 'connector*.json' \
  -o -name 'sink-*.json' -o -name 'source-*.json'

# 6) MirrorMaker configs
find . -name 'mm2.properties' -o -name 'mirror-maker*.properties' \
  -o -name 'connect-mirror-source*.properties'

# 7) Spark Streaming consumers (HDI Spark talking to HDI Kafka)
grep -RInE 'spark\.readStream.*kafka|KafkaUtils\.create' \
  --include='*.py' --include='*.scala'

# 8) Storm topologies
find . -name 'topology*.yaml' -o -name '*.flux'
```

### 2.3 Categorize Each Workload

Build a CSV: `topic, throughput_mbps, retention_h, compacted, partitions, replication, producers, consumers, schema_type, target, complexity`.

| Complexity | Definition | Migration Effort |
|------------|------------|------------------|
| **Low**    | One producer + one consumer, plain JSON, low throughput | hours |
| **Medium** | Multiple consumers, Avro w/ Schema Registry, partitioned by key, ESP enabled | days |
| **High**   | Kafka Streams app, exactly-once semantics, ksqlDB queries | 1-3 weeks |
| **Blocker**| MirrorMaker2 multi-cluster bidirectional, custom serializers in Scala JARs | re-architect or stay on Kafka |

---

## 3. Component Mapping

### 3.1 Brokers / Topics

| HDI Kafka | Fabric / Azure Equivalent | Notes |
|-----------|--------------------------|-------|
| Broker (`<host>:9092`) | **Event Hubs namespace Kafka endpoint** (`<ns>.servicebus.windows.net:9093`) | TLS-only; SASL/PLAIN with namespace conn string |
| Kafka topic | **Event Hub** (one EH = one topic; partitions configured at create) | Eventstream consumes from EH source |
| Topic retention (time-based) | EH retention: Basic 1d / Standard 7d / Premium & Dedicated 90d | Standard caps at 7d — choose Premium/Dedicated for longer retention |
| Topic retention (size-based) | **Not directly supported** — EH is time-based only | Workaround: tune partitions + retention days |
| Log compaction | **Event Hubs has no compaction** | Re-design as KQL update policy or Eventhouse materialized view |
| Partition count | EH partitions: Basic/Standard up to 32 per EH; Premium up to 100 per EH (200 per PU at namespace level); Dedicated up to 1024 per EH | Basic/Standard partition count is immutable after creation; Premium/Dedicated allow increase only |
| Replication factor | EH internal replication (3, managed) | Not configurable |
| ACLs / Ranger | Entra ID RBAC + namespace SAS policies | Topic-level RBAC is namespace-level only on Standard |
| ZooKeeper | Removed — EH is KRaft-style internally | No client-side change |

### 3.2 Streaming Compute

| HDI Component | Fabric Equivalent |
|---------------|-------------------|
| Spark Structured Streaming on HDI Spark cluster | **Fabric Spark Structured Streaming** (notebook or SJD) |
| Kafka Streams (Java) | No 1:1 — use **Fabric Spark** or run Kafka Streams on **AKS** |
| ksqlDB | **Eventstream Event Processor** (drag-drop) OR **KQL update policies** |
| Storm topology | **Eventstream + KQL** (re-architect) |
| HDI Kafka Connect | **Eventstream sources/destinations** (built-in for common connectors) |

### 3.3 Storage / Sinks

| HDI Pattern | Fabric Pattern |
|-------------|---------------|
| Spark stream → ADLS Parquet → Hive table | Eventstream → **Lakehouse destination** (Delta with V-Order) |
| Spark stream → HBase | Eventstream → **KQL DB** (replaces HBase for time-series) |
| Spark stream → Cosmos DB | Eventstream → **Cosmos DB destination** OR Eventstream → KQL → CDM |
| Spark stream → Power BI streaming dataset | Eventstream → **KQL DB → Power BI Direct Query / Direct Lake** |
| Spark stream → custom HTTP webhook | Eventstream → **Custom endpoint destination** OR **Activator → HTTP action** |

---

## 4. Topic + Schema Migration

### 4.1 Topic-by-Topic Provisioning Script

Create equivalent Event Hubs from your topic inventory.

```bash
# Pre-reqs: Event Hubs namespace exists, az CLI logged in
NS=my-eh-namespace
RG=eh-rg

# Read topics.txt + topic_describe.txt and create event hubs
while read topic; do
  # Pull partition count from describe output
  partitions=$(grep -A1 "Topic: $topic\b" topic_describe.txt | head -1 | \
               grep -oE 'PartitionCount: [0-9]+' | awk '{print $2}')
  retention_h=$(grep -B0 -A0 "^Topic: $topic\b" topic_configs.txt | \
               grep -oE 'retention.ms=[0-9]+' | head -1 | cut -d= -f2)
  retention_d=$(( ${retention_h:-86400000} / 86400000 ))
  retention_d=$(( retention_d < 1 ? 1 : retention_d ))

  echo "Creating EH '$topic' with $partitions partitions, $retention_d days retention"

  az eventhubs eventhub create \
    --resource-group $RG \
    --namespace-name $NS \
    --name "$topic" \
    --partition-count "${partitions:-4}" \
    --retention-time "$retention_d" \
    --cleanup-policy Delete \
    --status Active \
    --output none
done < topics.txt
```

> **Naming gotcha**: Event Hub names must match `^[a-zA-Z0-9][\w.-]{0,255}$`. Kafka
> topic names with `_` / `.` mix are usually fine, but topics starting with `_`
> (Confluent internal, `__consumer_offsets`) cannot be created — you don't need to
> migrate those.

### 4.2 Schema Migration

| Source Pattern | Target Pattern |
|----------------|---------------|
| **Confluent Schema Registry + Avro** | Either (a) keep SR on AKS, point producers there; or (b) embed schema in payload (Avro single-object encoding); or (c) switch to **JSON** + KQL `decode_*` functions |
| **Plain JSON** | No change — KQL parses JSON natively |
| **Protobuf** | Keep Protobuf serialization; KQL can decode via `parse_*` functions or staged via Spark |
| **Custom binary (Scala/Java serde)** | Re-implement in producer; Eventstream + KQL prefer text/JSON formats |

**Recommended**: For new ingestion paths, prefer **JSON Lines** or **Avro single-object
encoded** (schema embedded). This eliminates the Schema Registry dependency that Fabric
has no managed equivalent for.

### 4.3 Configure Producers/Consumers for Event Hubs Kafka Endpoint

```properties
# OLD — HDInsight Kafka (PLAINTEXT or SASL/GSSAPI)
bootstrap.servers=broker0:9092,broker1:9092,broker2:9092
security.protocol=PLAINTEXT

# NEW — Event Hubs Kafka endpoint (SASL_SSL + connection string)
bootstrap.servers=my-eh-namespace.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="$ConnectionString" \
  password="Endpoint=sb://my-eh-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=...";

# Also recommended:
session.timeout.ms=30000
request.timeout.ms=60000   # EH default broker-side is high; client should match
```

```python
# Python (confluent-kafka or kafka-python) — Event Hubs Kafka endpoint
from confluent_kafka import Producer
producer = Producer({
    'bootstrap.servers': 'my-eh-namespace.servicebus.windows.net:9093',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism':    'PLAIN',
    'sasl.username':     '$ConnectionString',
    'sasl.password':     'Endpoint=sb://my-eh-namespace...;SharedAccessKey=...',
})
producer.produce('orders', key='42', value=b'{"order_id":42}')
producer.flush()
```

```python
# Better: Entra ID OAuth (no static keys; aligns with Managed Identity story)
from confluent_kafka import Producer
from azure.identity import DefaultAzureCredential

cred = DefaultAzureCredential()

def oauth_cb(_):
    token = cred.get_token("https://eventhubs.azure.net/.default")
    return token.token, token.expires_on

producer = Producer({
    'bootstrap.servers': 'my-eh-namespace.servicebus.windows.net:9093',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism':    'OAUTHBEARER',
    'oauth_cb':          oauth_cb,
})
```

> **Gotcha**: Event Hubs Kafka endpoint is **TLS-only** (port 9093, never 9092). Plain
> `PLAINTEXT` configurations from HDI **must** be changed.

---

## 5. Cutover Strategy: MirrorMaker 2 Bridge

The safest cutover preserves both clusters in parallel using **MirrorMaker 2** (MM2) to
replicate from HDI Kafka → Event Hubs Kafka endpoint, letting you cut producers and
consumers over independently.

### 5.1 MM2 Source-Target Config

Run MM2 from a small AKS pod / VM that has network access to both endpoints.

```properties
# mm2.properties
clusters = hdi, eh

hdi.bootstrap.servers = broker0:9092,broker1:9092,broker2:9092
hdi.security.protocol = PLAINTEXT
# (or SASL/GSSAPI if HDI is ESP)

eh.bootstrap.servers  = my-eh-namespace.servicebus.windows.net:9093
eh.security.protocol  = SASL_SSL
eh.sasl.mechanism     = PLAIN
eh.sasl.jaas.config   = org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="$ConnectionString" \
  password="Endpoint=sb://my-eh-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=...";

# Direction: hdi → eh (one-way)
hdi->eh.enabled = true
hdi->eh.topics = .*
hdi->eh.topics.exclude = __consumer_offsets,_schemas,connect-.*

# Replication factor on target — EH manages internally, but client config still needs it
replication.factor = 3
checkpoints.topic.replication.factor = 3
heartbeats.topic.replication.factor  = 3
offset-syncs.topic.replication.factor= 3

# Translate consumer group offsets too
hdi->eh.sync.group.offsets.enabled = true
hdi->eh.emit.checkpoints.enabled   = true

# Don't auto-rename topics with cluster prefix (default would be "hdi.orders")
hdi->eh.replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy
```

```bash
# Start MM2 (Kafka 2.7+ ships connect-mirror-maker.sh)
$KBIN/connect-mirror-maker.sh mm2.properties
```

> **Critical gotcha**: The default `DefaultReplicationPolicy` prefixes topics with the
> source cluster alias (e.g., `hdi.orders`). For an in-place migration where consumers
> point at the same topic name, you **must** set `IdentityReplicationPolicy`. Test
> consumer behavior carefully — IdentityReplicationPolicy is incompatible with
> bidirectional replication.

### 5.2 Cutover Sequence

```
T-7d  : Stand up Event Hubs namespace + provision all topics
T-6d  : Start MM2 hdi → eh; verify lag stays ~0
T-5d  : Cut consumers ONE GROUP AT A TIME to EH endpoint
        - Stop HDI consumer
        - Use MM2 checkpoint to translate offset (see 5.3)
        - Start EH consumer at translated offset
        - Validate counts, then process forward
T-2d  : All consumers on EH; HDI consumer-side empty
T-1d  : Cut PRODUCERS to EH endpoint (config swap + restart)
        - Stop HDI producer
        - Wait for MM2 to drain remaining records
        - Start EH producer
T-0   : Stop MM2; HDI cluster has no traffic
T+7d  : Tear down HDI Kafka cluster (keep ADLS / commit offsets backup)
```

### 5.3 Translating Consumer-Group Offsets

MM2 maintains an `offset-syncs` internal topic that maps source offsets → target offsets.
On cutover, query the checkpoint to get the target offset for each consumer group:

```bash
$KBIN/kafka-console-consumer.sh \
  --bootstrap-server $EH_BROKERS --consumer.config eh-client.properties \
  --topic mm2-checkpoints.hdi.internal \
  --from-beginning --max-messages 1000 \
  --formatter org.apache.kafka.connect.mirror.formatters.CheckpointFormatter \
  > checkpoints.txt

# Each line: group=X, topic=Y, partition=Z, upstream-offset=A, downstream-offset=B
# Use B (downstream-offset) as the seek target on the EH consumer
```

For each consumer group, after stopping it on HDI:

```python
from confluent_kafka import Consumer, TopicPartition
c = Consumer({
    'bootstrap.servers': 'my-eh-namespace.servicebus.windows.net:9093',
    'security.protocol': 'SASL_SSL', 'sasl.mechanism': 'PLAIN',
    'sasl.username': '$ConnectionString', 'sasl.password': '...',
    'group.id': 'orders-consumer-v1',
    'enable.auto.commit': False,
})

# Set group offsets from MM2 checkpoint
checkpoints = {('orders', 0): 1024567, ('orders', 1): 1019988, ...}
tps = [TopicPartition(t, p, off) for (t, p), off in checkpoints.items()]
c.commit(offsets=tps, asynchronous=False)
print("Offsets committed; safe to start consumer normally")
```

### 5.4 Producer Cutover Idempotence
Use **producer transactions** OR **idempotent producers** during cutover to avoid
duplicates if a producer has in-flight records during config swap:

```properties
enable.idempotence=true
acks=all
max.in.flight.requests.per.connection=5
retries=2147483647
```

---

## 6. Fabric Eventstream Wiring

### 6.1 Create Eventstream Pointing at Event Hubs

```python
# Fabric REST API — create Eventstream with EH source
import requests
ws = "<workspace-id>"
hdrs = {"Authorization": f"Bearer {fabric_token}", "Content-Type": "application/json"}

resp = requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{ws}/eventstreams",
    headers=hdrs,
    json={
        "displayName": "orders_stream",
        "description": "Orders ingestion from EH",
    },
)
es_id = resp.json()["id"]

# Add EH source via topology PATCH (UI-driven flow shown for reference)
# Topology JSON includes:
#   sources:        [{name: "orders_eh", type: "AzureEventHub",
#                     properties: {endpoint, eventHubName, sharedAccessKeyName,
#                                  sharedAccessKey OR managedIdentity}}]
#   destinations:   [{name: "to_eventhouse", type: "Eventhouse",
#                     properties: {workspaceId, eventhouseId, kqlDatabase, table}}]
#   operators:      [{name: "filter1", type: "Filter", inputs:["orders_eh"],
#                     properties: {expression: "amount > 0"}}, ...]
#   streams:        edges connecting sources → operators → destinations
```

### 6.2 Eventstream Sources (Replacing HDI Kafka Producers)

| HDI Producer Source | Eventstream Source |
|---------------------|-------------------|
| Custom app → HDI Kafka | **Custom App** source (uses EH SAS or managed identity) |
| Kafka Connect JDBC source | **Database CDC** source (SQL, Postgres, Cosmos DB CDC) |
| Filebeat / Logstash → Kafka | **Azure Blob/ADLS** source OR custom app |
| IoT devices → Kafka | **Azure IoT Hub** source |
| App Insights / Azure Monitor logs | **Azure Monitor** source |
| Twitter / external API streams | **Sample data** + Custom App via webhook |

### 6.3 Eventstream Destinations (Replacing HDI Spark Sinks)

| HDI Sink | Eventstream Destination |
|----------|-------------------------|
| ADLS Gen2 Parquet (via Spark) | **Lakehouse** (writes Delta with V-Order) |
| Hive table (via Spark) | **Lakehouse** (table appears in SQL endpoint) |
| HBase | **KQL Database** (Eventhouse) |
| Power BI streaming dataset | **KQL DB** + Power BI Direct Query |
| Cosmos DB (via Spark) | **Custom endpoint** + Function/Logic App OR direct KQL |
| Webhook / HTTP | **Custom endpoint** OR **Activator → HTTP action** |
| Another Kafka topic | **Custom endpoint** (re-publish to EH) |

### 6.4 Eventstream Event Processor (Replacing ksqlDB / Light Spark Logic)

Drag-drop operators replace simple stream transformations:

| ksqlDB / Spark Stream Operation | Eventstream Operator |
|---------------------------------|---------------------|
| `WHERE amount > 100`            | **Filter** |
| `SELECT col1, col2 AS new_name` | **Manage fields** (rename / project) |
| `CASE WHEN ... THEN ...`        | **Manage fields** with derived field expression |
| `GROUP BY window`               | **Group by** (tumbling, hopping, session windows) |
| `JOIN reference_table`          | **Join** with reference data (uploaded CSV/JSON) |
| `SPLIT BY type`                 | **Conditional split** (multiple downstream branches) |
| Custom UDF                      | Not directly supported — push to KQL update policy or Spark |

> **Limitation**: Eventstream Event Processor handles single-event and windowed
> aggregations well, but **stream-stream joins** and **stateful exactly-once** logic
> belong in **Spark Structured Streaming** or **KQL update policies**.

---

## 7. Eventhouse / KQL Database Setup

### 7.1 Create Eventhouse + Database

```python
# Fabric REST — create eventhouse (bundle of KQL DB + queryset)
resp = requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{ws}/eventhouses",
    headers=hdrs,
    json={"displayName": "orders_eh", "description": "Orders telemetry"},
)
```

### 7.2 KQL Table + Ingestion Mapping

```kql
// Create table matching expected ingest schema
.create table OrdersRaw (
    EventTime: datetime,
    OrderId:   long,
    CustomerId:long,
    Amount:    real,
    Status:    string,
    Region:    string,
    RawJson:   dynamic
)

// JSON ingestion mapping — maps Eventstream JSON → table columns
.create table OrdersRaw ingestion json mapping 'OrdersRawMapping'
'['
'  {"column":"EventTime",  "Properties":{"Path":"$.event_time","Transform":"DateTimeFromUnixSeconds"}},'
'  {"column":"OrderId",    "Properties":{"Path":"$.order_id"}},'
'  {"column":"CustomerId", "Properties":{"Path":"$.customer_id"}},'
'  {"column":"Amount",     "Properties":{"Path":"$.amount"}},'
'  {"column":"Status",     "Properties":{"Path":"$.status"}},'
'  {"column":"Region",     "Properties":{"Path":"$.region"}},'
'  {"column":"RawJson",    "Properties":{"Path":"$"}}'
']'

// Streaming ingestion policy — required for sub-5s latency from Eventstream
.alter table OrdersRaw policy streamingingestion enable
```

### 7.3 Update Policies (Replacing ksqlDB / Spark Stream Transformations)

KQL update policies run automatically on every ingestion, populating downstream
"silver/gold" tables — equivalent to a Spark Structured Streaming aggregation but
declarative and serverless.

```kql
// Silver: cleansed
.create table OrdersSilver (
    EventTime: datetime, OrderId: long, CustomerId: long,
    Amount: decimal, Status: string, Region: string
)

.create-or-alter function ParseRawOrders() {
    OrdersRaw
    | where isnotempty(OrderId) and Amount > 0
    | extend Amount = todecimal(Amount)
    | project EventTime, OrderId, CustomerId, Amount, Status, Region
}

.alter table OrdersSilver policy update
'[{"Source": "OrdersRaw", "Query": "ParseRawOrders()", "IsEnabled": true, "IsTransactional": true}]'

// Gold: aggregated by 5-minute tumbling window — runs automatically
.create table OrdersAgg5m (
    Window: datetime, Region: string, OrderCount: long, Revenue: decimal
)

.create-or-alter function AggOrders5m() {
    OrdersSilver
    | summarize OrderCount = count(), Revenue = sum(Amount)
        by Window = bin(EventTime, 5m), Region
}

// Drive aggregation via materialized view (preferred over update policy for aggregations)
.create materialized-view AggOrders5mMV on table OrdersSilver
{
    OrdersSilver
    | summarize Revenue = sum(Amount), OrderCount = count()
        by Window = bin(EventTime, 5m), Region
}
```

### 7.4 Spark Structured Streaming Equivalent (When KQL Isn't Enough)

Some HDI Spark patterns (stateful joins, complex windowing, ML scoring inline) need
to stay in Spark. Run them in a **Fabric Spark notebook** consuming from Event Hubs:

```python
# Fabric notebook — Spark Structured Streaming from Event Hubs Kafka endpoint
EH_BOOTSTRAP = "my-eh-namespace.servicebus.windows.net:9093"
EH_CONNSTR   = mssparkutils.credentials.getSecret("https://my-kv.vault.azure.net/", "eh-conn")
JAAS = (f'org.apache.kafka.common.security.plain.PlainLoginModule required '
        f'username="$ConnectionString" password="{EH_CONNSTR}";')

stream = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", EH_BOOTSTRAP)
    .option("subscribe", "orders")
    .option("kafka.security.protocol", "SASL_SSL")
    .option("kafka.sasl.mechanism", "PLAIN")
    .option("kafka.sasl.jaas.config", JAAS)
    .option("startingOffsets", "earliest")
    .load()
    .selectExpr("CAST(value AS STRING) as json", "timestamp")
)

from pyspark.sql.functions import from_json
from pyspark.sql.types import *
schema = StructType([
    StructField("order_id",    LongType()),
    StructField("customer_id", LongType()),
    StructField("amount",      DoubleType()),
    StructField("status",      StringType()),
    StructField("region",      StringType()),
])

parsed = stream.select(
    from_json("json", schema).alias("d"), "timestamp"
).select("d.*", "timestamp")

# Write to Lakehouse Delta with availableNow trigger (cost-efficient)
(parsed.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "Files/_checkpoints/orders_bronze/")
    .trigger(availableNow=True)
    .toTable("sales_lakehouse.orders_bronze"))
```

---

## 8. Security Mapping

### 8.1 Authentication

| HDI Kafka Auth | Fabric / EH Equivalent |
|----------------|-----------------------|
| PLAINTEXT (no auth) | **Forbidden** — EH requires TLS + SASL |
| SASL/PLAIN with username/password | **SAS Connection String** (`username=$ConnectionString`, password=conn-str) |
| SASL/SCRAM-SHA-512 | Not supported on EH — use SAS or OAuth |
| SASL/GSSAPI (Kerberos / ESP) | **OAuth 2.0 (Entra ID)** — `sasl.mechanism=OAUTHBEARER` |
| mTLS client certs | Not supported on EH Kafka endpoint — use SAS or OAuth |

**Action item**: Remove `kinit`, krb5.conf, JAAS Kerberos sections, and keytab files
from your producer/consumer hosts. Replace with either SAS connection strings (simple)
or OAuth via DefaultAzureCredential (preferred — no rotation).

### 8.2 Authorization

| HDI Kafka ACLs / Ranger | Fabric / EH Equivalent |
|-------------------------|-----------------------|
| Topic-level READ ACL for principal | EH SAS policy (Listen / Send / Manage) at namespace OR EH level |
| Group-based ACLs via Ranger | Entra ID groups + Azure RBAC role assignments on namespace |
| Cluster admin (CREATE TOPIC) | `Azure Event Hubs Data Owner` role |
| Per-topic READ-only consumer | `Azure Event Hubs Data Receiver` role on specific EH |
| Per-topic WRITE-only producer | `Azure Event Hubs Data Sender` role on specific EH |

```bash
# Grant a managed identity Send rights on a specific event hub
az role assignment create \
  --assignee $MI_OBJECT_ID \
  --role "Azure Event Hubs Data Sender" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.EventHub/namespaces/$NS/eventhubs/orders"
```

### 8.3 Network

| HDI Kafka | Fabric / EH |
|-----------|-------------|
| VNet-injected Kafka cluster | EH **Private Endpoint** (Premium / Dedicated) OR **VNet service endpoint** |
| NSG rules on broker subnet | NSG rules on PE subnet |
| ExpressRoute → Kafka | ExpressRoute → EH PE (same model) |
| Public Kafka endpoint | EH public endpoint with **IP firewall rules** |

---

## 9. Real-Time Dashboards & Alerting

### 9.1 Replacing HDI Spark → Power BI Streaming Datasets

Old pattern: Spark Structured Streaming pushes to Power BI streaming dataset via REST.

New pattern (faster, no code):
```
Eventstream → KQL DB → Real-Time Dashboard (sub-second refresh)
                    OR Power BI Direct Query / Direct Lake on KQL DB
```

```kql
// KQL query powering a Real-Time Dashboard tile
OrdersSilver
| where EventTime > ago(1h)
| summarize Revenue = sum(Amount) by bin(EventTime, 1m), Region
| render timechart
```

### 9.2 Replacing Custom Storm/Spark Alerters with Activator

```
Eventstream → Activator (no-code rule):
  WHEN Amount > 10000 OR Status == 'fraud_suspected'
  WITHIN 1 minute, GROUPED BY CustomerId
  THEN POST to https://hooks.example.com/fraud
       AND email security@example.com
       AND post Teams channel #alerts
```

Activator replaces hand-rolled Spark Streaming alerter pipelines, with built-in
de-duplication, rate-limiting, and audit trail.

---

## 10. Validation

### 10.1 Per-Topic Reconciliation

Run for each topic during MM2 bridge phase:

```bash
# Source: HDI Kafka — count records over a fixed window
SOURCE_COUNT=$($KBIN/kafka-run-class.sh kafka.tools.GetOffsetShell \
  --bootstrap-server $HDI_BROKERS --topic orders --time -1 \
  | awk -F: '{sum += $3} END {print sum}')

# Target: Event Hubs — same approach
TARGET_COUNT=$($KBIN/kafka-run-class.sh kafka.tools.GetOffsetShell \
  --bootstrap-server $EH_BROKERS --command-config eh-client.properties \
  --topic orders --time -1 \
  | awk -F: '{sum += $3} END {print sum}')

echo "Source: $SOURCE_COUNT / Target: $TARGET_COUNT  diff: $((SOURCE_COUNT - TARGET_COUNT))"
```

### 10.2 Content Equality (Sample-Based)

```python
# Pull last 1000 records from each side, compare by (key, value-hash)
# (use a small kafka client — confluent-kafka or kafka-python)
def fetch_sample(brokers, props, topic, n=1000):
    from confluent_kafka import Consumer, TopicPartition
    c = Consumer({**props, 'bootstrap.servers': brokers, 'group.id': f'verify-{topic}',
                  'auto.offset.reset': 'latest', 'enable.auto.commit': False})
    parts = c.list_topics(topic).topics[topic].partitions
    seek = []
    for pid in parts:
        tp = TopicPartition(topic, pid)
        lo, hi = c.get_watermark_offsets(tp)
        seek.append(TopicPartition(topic, pid, max(lo, hi - n // len(parts))))
    c.assign(seek)
    out = {}
    for _ in range(n):
        m = c.poll(2.0)
        if m is None: break
        out[(m.key(), hash(m.value()))] = (m.partition(), m.offset())
    c.close()
    return out

src = fetch_sample(HDI_BROKERS, {'security.protocol':'PLAINTEXT'}, 'orders')
tgt = fetch_sample(EH_BROKERS, EH_PROPS, 'orders')
intersection = set(src) & set(tgt)
print(f"src={len(src)} tgt={len(tgt)} intersection={len(intersection)} "
      f"agreement={len(intersection)/min(len(src),len(tgt)):.2%}")
```

### 10.3 Acceptance Criteria

```yaml
migration_pass_criteria:
  data_correctness:
    - per_topic_count_diff_pct: < 0.01    # MM2 lag-tolerant
    - sample_content_agreement_pct: > 99.9
    - consumer_group_offset_translated: all
  performance:
    - producer_p99_latency_ms_delta_pct: < 30
    - consumer_p99_lag_seconds: < 10
  reliability:
    - mm2_bridge_uptime_7d_pct: > 99.9
    - eh_throttle_events_7d: 0
    - eventstream_run_state: Active
  security:
    - all_clients_use_tls: true
    - no_plaintext_endpoints: true
    - all_kerberos_keytabs_removed: true
```

---

## 11. Cost Re-Modeling

| HDI Kafka Cost | Fabric / EH Equivalent | Optimization Tip |
|----------------|----------------------|------------------|
| Broker VMs (D4 × 3 × 24 × 30 = ~$870/mo) | EH Standard ($0.028/M events) OR EH Premium PU | Premium PU=1 (~$700/mo) ≈ 1 dedicated broker; pick by throughput |
| ZooKeeper VMs (~$220/mo) | None — managed | Pure savings |
| Storage (broker disks) | EH retention storage (Basic 1d / Standard 7d / Premium & Dedicated up to 90d) | Premium/Dedicated needed for >7-day retention |
| Inter-AZ replication | Built-in (free) | Pure savings |
| MirrorMaker DR | EH Geo-DR (paired namespace pairing) | Built-in; no MM2 ops cost |
| Spark cluster for stream processing | Fabric CU consumption by Eventstream + KQL | KQL is order(s) of magnitude cheaper than Spark for analytics |
| Dedicated bandwidth costs | Included in EH SKU; PE traffic separate | Plan PE bandwidth costs |

**Sizing rule of thumb**:
- HDI Kafka cluster of 3× D12 brokers (~24 TU equivalent, 30 MB/s ingress) ≈ **EH Standard 20 TUs** OR **EH Premium PU=1** (better for >50 MB/s sustained).
- Eventstream throughput is bounded by Fabric capacity CU: ~50 MB/s per F64 capacity for typical JSON. For higher throughput, use EH directly + Fabric Spark consumer.

---

## 12. Pitfalls & Lessons Learned

1. **Plain TCP / port 9092 will not work**. Event Hubs Kafka endpoint is **9093 + TLS**
   only. All client configs must change `bootstrap.servers` AND add SASL_SSL.

2. **Event Hubs has no log compaction**. If you rely on compacted topics (e.g., Kafka
   Streams state stores, change-log topics), you cannot drop in EH. Keep those topics
   on Kafka (managed Confluent on AKS) OR re-architect as KQL `materialized-view` /
   Lakehouse Delta MERGE.

3. **`__consumer_offsets` doesn't migrate via MM2**. MM2 mirrors checkpoints, not the
   `__consumer_offsets` topic itself. Use the offset translation pattern (§5.3) at
   cutover time.

4. **Confluent Schema Registry has no managed equivalent**. Either: keep SR on AKS,
   embed schema in payload (Avro single-object encoding), or move to schema-on-read
   in KQL with `parse_json()`.

5. **`IdentityReplicationPolicy` vs `DefaultReplicationPolicy`**. Default prefixes
   topics with cluster alias (`hdi.orders`); Identity does not. Pick BEFORE starting
   MM2 — switching mid-flight orphans target offsets.

6. **EH partition count is immutable on Basic and Standard tiers**. Pick wisely.
   Premium and Dedicated tiers allow partition INCREASE (not decrease) after
   creation. Spark/Kafka consumer parallelism per group is bounded by partition count.

7. **Idempotent producers + EH transactions**. `transactional.id` is supported on EH
   (since 2023) but with caveats — EOS across topic boundaries may not match Kafka
   guarantees exactly. Test transaction-heavy producers thoroughly.

8. **Kafka Streams apps don't run on Fabric**. There is no "Fabric Streams" runtime.
   Three options: (a) keep Kafka Streams app on AKS pointed at EH; (b) rewrite as
   Fabric Spark Structured Streaming; (c) re-implement in KQL update policies for
   simple cases.

9. **HDI ESP → no Kerberos support on EH**. ESP-secured Kafka clients with `GSSAPI`
   must be re-configured with `OAUTHBEARER` (recommended) or `PLAIN` (with SAS).
   Remove `krb5.conf`, JAAS Kerberos sections, and keytab volumes.

10. **MirrorMaker offset-syncs topic ACL**. The MM2 process needs `Send` rights on
    `mm2-offset-syncs.eh.internal`, `mm2-checkpoints.hdi.internal`, and
    `heartbeats` topics on both sides. Easy to miss in IAM setup.

11. **Eventstream operator language is not SQL**. The drag-drop operators map to a
    proprietary expression language. Complex transformations belong in KQL update
    policies (declarative SQL-like) or Spark notebooks (PySpark).

12. **Activator rules are not Spark UDFs**. Activator handles per-event and windowed
    rule evaluation but cannot run arbitrary code. For ML-based alerting, score in
    Spark/MLflow and write the score back to KQL, then alert on the score column.

13. **Geo-DR ≠ active-active**. EH Geo-DR uses **alias** + **paired secondary** for
    failover; only one side is writable at a time. If you used MM2 for active-active
    multi-region with HDI, you must redesign — keep MM2 between two EH namespaces, or
    use Confluent Cluster Linking between Confluent Cloud regions.

14. **JSON ingestion mapping is ordered**. KQL JSON ingestion mappings match by
    `Path`, but column types are inferred at table-create time. Schema drift in
    incoming JSON silently drops/coerces fields. Snapshot a `.show ingestion failures`
    report daily during early operation.

15. **KQL streaming ingestion has a ~5s latency floor**. If you need <1s end-to-end
    (e.g., real-time fraud blocking), use a Spark Structured Streaming consumer that
    writes directly to a downstream system, not Eventstream → KQL.

16. **Event Hubs send-batching matters**. Default Kafka producer batches by size +
    linger.ms. For low-throughput topics, set `linger.ms=20` to avoid sending one
    record per request — EH bills per-event AND per-operation.

17. **MM2 throughput needs sizing**. Each MM2 task handles N partitions. For a
    high-throughput topic, set `tasks.max` proportional to partitions. Default `1` is
    almost always too low — verify lag during initial sync.

---

## 13. Quick-Reference Migration Checklist

```
[ ] Inventory HDI Kafka: topics, configs, ACLs, consumer groups, sizes
[ ] Inventory clients: producer / consumer / Streams app code, schemas, auth modes
[ ] Categorize each topic: target = EH-only, Eventstream-bridge, or stay-on-Kafka
[ ] Decide schema strategy: keep SR on AKS, embed schema, or schema-on-read in KQL
[ ] Provision Event Hubs namespace (Standard or Premium based on throughput)
[ ] Provision per-topic event hubs (script from inventory)
[ ] Configure Entra ID + RBAC roles for producers/consumers (or generate SAS)
[ ] Stand up MM2 bridge HDI → EH, verify lag stays ~0 for ≥48h
[ ] Provision Fabric Eventhouse + KQL DB tables + ingestion mappings
[ ] Create Eventstream(s): EH source → operators → Eventhouse / Lakehouse destination
[ ] Validate per-topic counts + sample content equality
[ ] Cut consumers ONE GROUP AT A TIME, translate offsets, verify
[ ] Cut producers (config swap + restart with idempotent + acks=all)
[ ] Stop MM2; verify HDI cluster traffic dropped to zero
[ ] Switch dashboards: Spark/PBI streaming datasets → Real-Time Dashboards on KQL DB
[ ] Convert custom alerters → Activator rules
[ ] Schedule Eventhouse maintenance: retention policies, cache policy review
[ ] Set up alerts: EH throttling, Eventstream run-state, KQL ingestion failures
[ ] Tear down HDI Kafka cluster (keep ADLS / commit-offset backup for 30d)
[ ] Document new runbook + on-call procedures
```

---

## 14. Useful Commands & Snippets

```bash
# Get EH connection string (RootManageSharedAccessKey)
az eventhubs namespace authorization-rule keys list \
  --resource-group $RG --namespace-name $NS \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv

# List event hubs in a namespace
az eventhubs eventhub list -g $RG --namespace-name $NS --output table

# Tail an EH topic with kafkacat (kcat)
kcat -b my-eh-namespace.servicebus.windows.net:9093 \
     -X security.protocol=SASL_SSL \
     -X sasl.mechanisms=PLAIN \
     -X sasl.username='$ConnectionString' \
     -X sasl.password="$EH_CONNSTR" \
     -t orders -C -o end -q -J

# Check MM2 lag (it runs as a Connect cluster — query its REST API on port 8083)
curl -s http://mm2-host:8083/connectors/MirrorSourceConnector/status | python3 -m json.tool
curl -s http://mm2-host:8083/connectors/MirrorSourceConnector/topics

# KQL: ingestion failures (run in KQL queryset)
.show ingestion failures
| where FailedOn > ago(1h)
| summarize count() by Database, Table, Reason
```

---

## Rules

1. **Never expose plaintext (port 9092) on the new path** — EH is TLS-only by design.
2. **Bridge with MM2 before cutover** — minimum 48h zero-lag observation before swapping clients.
3. **Cut consumers before producers** — it's safe to drain old producers; it's not safe to leave new producers without a consumer.
4. **Use `IdentityReplicationPolicy` for in-place name migration** — and verify before starting MM2.
5. **Replace Kerberos/ESP auth with OAuth (Entra ID)** — or SAS connection strings — never both at once.
6. **Pick EH partition count based on consumer parallelism, not source topic** — Basic/Standard tiers can't change after creation; Premium/Dedicated allow increase only.
7. **Keep Kafka Streams apps where they live** — Fabric has no Streams runtime; rewrite or stay on AKS.
8. **Prefer KQL update policies + materialized views** for stream aggregations — orders of magnitude cheaper than Spark.
9. **Drop dependence on log compaction** for migrated topics — re-design as KQL materialized views or Lakehouse MERGE.
10. **Keep HDI Kafka cluster for 30 days post-cutover** as instant rollback.
