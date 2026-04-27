---
name: azure-cosmosdb
description: Azure Cosmos DB — multi-model globally distributed database, SQL API, MongoDB API, Cassandra, Gremlin, Table, partitioning, consistency levels, change feed
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Cosmos DB — Comprehensive Reference

Azure Cosmos DB is Microsoft's globally distributed, multi-model NoSQL database with
single-digit millisecond latency, elastic scalability, and five consistency models.

---

## 1. API Models

| API | Wire Protocol | Best For |
|-----|--------------|----------|
| **NoSQL (SQL)** | Cosmos-native | JSON documents, SQL-like queries |
| **MongoDB** | MongoDB 4.x–6.x | Existing Mongo apps, ODM compatibility |
| **Cassandra** | CQL v4 | Wide-column, time-series, IoT |
| **Gremlin** | Apache TinkerPop | Graph traversal, fraud detection |
| **Table** | Azure Table Storage | Key-value, lift-and-shift |
| **PostgreSQL** | PostgreSQL wire | Relational + Citus distributed extension |

Choose **NoSQL** for new projects unless you have an existing driver dependency.

---

## 2. Throughput Modes

### 2.1 Provisioned Throughput

Capacity reserved in **Request Units per second (RU/s)**, billed per hour.
Minimum: 400 RU/s per container (or shared at database level).

```bash
az cosmosdb create \
  --name mycosmosaccount --resource-group myRG \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true

az cosmosdb sql database create \
  --account-name mycosmosaccount --resource-group myRG \
  --name mydb --throughput 400

az cosmosdb sql container create \
  --account-name mycosmosaccount --resource-group myRG \
  --database-name mydb --name mycontainer \
  --partition-key-path "/tenantId" --throughput 1000
```

### 2.2 Autoscale Throughput

Scales RU/s between 10% of max and max automatically. Pay for peak reached each hour.

```bash
az cosmosdb sql container create \
  --account-name mycosmosaccount --resource-group myRG \
  --database-name mydb --name mycontainer-auto \
  --partition-key-path "/userId" --max-throughput 4000  # autoscale 400–4000 RU/s

# Migrate manual → autoscale
az cosmosdb sql container throughput migrate \
  --account-name mycosmosaccount --resource-group myRG \
  --database-name mydb --name mycontainer \
  --throughput-type autoscale
```

### 2.3 Serverless

Pay per RU consumed, no reservation. Single region only, max ~5,000 RU/s burst.
Best for dev/test and very spiky workloads.

```bash
az cosmosdb create \
  --name myserverlessaccount --resource-group myRG \
  --capabilities EnableServerless \
  --locations regionName=eastus failoverPriority=0
```

---

## 3. Partitioning Strategy

### 3.1 Partition Key Selection

The partition key is **immutable** after container creation. Good keys have:
- High cardinality (many distinct values)
- Even read/write distribution
- Appear in most query WHERE clauses

| Scenario | Partition Key |
|----------|--------------|
| Multi-tenant SaaS | `/tenantId` |
| User data | `/userId` |
| IoT telemetry | `/deviceId` |
| Orders | `/customerId` |

### 3.2 Synthetic Partition Keys

Combine fields when no single field has high enough cardinality:

```python
def get_partition_key(user_id: str, region: str) -> str:
    return f"{user_id}_{region}"

item = {
    "id": "order-001",
    "partitionKey": get_partition_key("u123", "us-east"),  # synthetic
    "amount": 49.99
}
```

### 3.3 Hot Partition Mitigation

Suffix randomization for write-heavy keys; query all suffixes and merge on read:

```python
import random
def get_suffixed_key(base_key: str, n: int = 10) -> str:
    return f"{base_key}_{random.randint(0, n - 1)}"
```

---

## 4. Consistency Levels

Set at account level; overridable per request. Weaker = lower latency + lower RU cost.

| Level | Guarantee | RU Cost |
|-------|-----------|---------|
| **Strong** | Linearizability — always reads latest write | 2× |
| **Bounded Staleness** | Lag ≤ K versions or T seconds | 2× |
| **Session** | Monotonic reads/writes within a session token | 1× |
| **Consistent Prefix** | No out-of-order reads | 1× |
| **Eventual** | Lowest latency; reads may be stale | 1× |

**Session** is the default and recommended for most apps.

```bash
az cosmosdb update --name mycosmosaccount --resource-group myRG \
  --default-consistency-level Session
```

```python
from azure.cosmos import CosmosClient, ConsistencyLevel

client = CosmosClient(url, credential, consistency_level=ConsistencyLevel.Session)

# Override for a single read
item = container.read_item(
    item="doc-001", partition_key="tenant-a",
    consistency_level=ConsistencyLevel.Strong
)
```

---

## 5. Global Distribution & Multi-Region Writes

```bash
# Add a read region; enable multi-region writes (active-active)
az cosmosdb update \
  --name mycosmosaccount --resource-group myRG \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
              regionName=westeurope failoverPriority=1 \
  --enable-multiple-write-locations true \
  --enable-automatic-failover true

# Trigger manual failover (DR drill)
az cosmosdb failover-priority-change \
  --name mycosmosaccount --resource-group myRG \
  --failover-policies eastus=0 westeurope=1
```

```python
# Route reads to nearest region
client = CosmosClient(url, key,
    preferred_regions=["West Europe", "East US"])
```

### Conflict Resolution (Multi-Region Write)

| Policy | Description |
|--------|-------------|
| **Last Write Wins (LWW)** | Default — uses `_ts` or custom numeric path |
| **Custom** | Stored procedure resolves conflicts |
| **Manual** | Conflicts written to conflicts feed |

```bash
az cosmosdb sql container create ... \
  --conflict-resolution-policy \
  '{"mode":"lastWriterWins","conflictResolutionPath":"/priority"}'
```

---

## 6. Change Feed

### 6.1 Change Feed Processor (.NET)

```csharp
var processor = container.GetChangeFeedProcessorBuilder<MyDoc>(
        processorName: "myProcessor",
        onChangesDelegate: HandleChangesAsync)
    .WithInstanceName("host-1")
    .WithLeaseContainer(leaseContainer)
    .Build();
await processor.StartAsync();

static async Task HandleChangesAsync(
    ChangeFeedProcessorContext ctx,
    IReadOnlyCollection<MyDoc> changes, CancellationToken ct)
{
    foreach (var doc in changes) Console.WriteLine($"Changed: {doc.Id}");
}
```

### 6.2 Change Feed in Python (pull model)

```python
feed = container.query_items_change_feed(is_start_from_beginning=True)
for item in feed:
    print(f"Changed: {item['id']}")
# Save etag header as continuation token for resume
```

### 6.3 Azure Functions Trigger

```python
import azure.functions as func
def main(documents: func.DocumentList) -> None:
    for doc in documents:
        print(f"Changed id: {doc['id']}")
```

```json
{
  "type": "cosmosDBTrigger", "name": "documents", "direction": "in",
  "connectionStringSetting": "CosmosDBConnection",
  "databaseName": "mydb", "collectionName": "mycontainer",
  "leaseCollectionName": "leases", "createLeaseCollectionIfNotExists": true
}
```

---

## 7. Indexing Policies

By default Cosmos DB indexes **all** properties. Customize to reduce write RU cost.

```json
{
  "indexingMode": "consistent",
  "includedPaths": [{ "path": "/*" }],
  "excludedPaths": [
    { "path": "/largeBlob/*" },
    { "path": "/_etag/?" }
  ],
  "compositeIndexes": [
    [
      { "path": "/timestamp", "order": "descending" },
      { "path": "/tenantId", "order": "ascending" }
    ]
  ],
  "spatialIndexes": [
    { "path": "/location/*", "types": ["Point", "Polygon"] }
  ]
}
```

Composite indexes are **required** for ORDER BY on multiple properties.

```bash
az cosmosdb sql container update \
  --account-name mycosmosaccount --resource-group myRG \
  --database-name mydb --name mycontainer --idx @indexing-policy.json
```

---

## 8. Stored Procedures, Triggers, UDFs (JavaScript)

All server-side logic runs transactionally within a **single partition**.

```javascript
// Stored procedure: bulk insert
function bulkInsert(docs) {
    var ctx = getContext(), col = ctx.getCollection(), res = ctx.getResponse(), n = 0;
    function insert(i) {
        if (i >= docs.length) { res.setBody(n); return; }
        var ok = col.createDocument(col.getSelfLink(), docs[i], function(err) {
            if (err) throw err; n++; insert(i + 1);
        });
        if (!ok) res.setBody(n);
    }
    insert(0);
}

// Pre-trigger: stamp createdAt
function stampCreatedAt() {
    var req = getContext().getRequest(), doc = req.getBody();
    doc.createdAt = new Date().toISOString();
    req.setBody(doc);
}

// UDF: tax calculation (used in SELECT)
function calculateTax(price, region) {
    var rates = { "us-east": 0.08, "eu-west": 0.20 };
    return price * (rates[region] || 0.10);
}
```

```sql
-- Use UDF in query
SELECT c.id, udf.calculateTax(c.price, c.region) AS tax FROM c
```

---

## 9. SDK Patterns

### Python (azure-cosmos)

```python
from azure.cosmos import CosmosClient, PartitionKey

client = CosmosClient(url=COSMOS_ENDPOINT, credential=COSMOS_KEY)
container = client.get_database_client("mydb").get_container_client("mycontainer")

# Point read — cheapest (1 RU / 1 KB)
item = container.read_item(item="user-001", partition_key="u001")

# Upsert
container.upsert_item({"id": "user-001", "userId": "u001", "name": "Alice"})

# Parameterized query
for row in container.query_items(
    query="SELECT * FROM c WHERE c.name = @n",
    parameters=[{"name": "@n", "value": "Alice"}],
    enable_cross_partition_query=False
):
    print(row)

# Patch (partial update)
container.patch_item("user-001", partition_key="u001", patch_operations=[
    {"op": "replace", "path": "/email", "value": "new@example.com"},
    {"op": "add", "path": "/tags/0", "value": "premium"}
])

# Transactional batch (same partition only)
container.execute_item_batch([
    ("upsert", {"id": "a", "pk": "p1"}, {}),
    ("delete", "b", {"partitionKey": "p1"}),
], partition_key="p1")
```

### .NET (Microsoft.Azure.Cosmos)

```csharp
var client = new CosmosClient(endpoint, key, new CosmosClientOptions {
    ApplicationPreferredRegions = new[] { "East US", "West Europe" }
});
var container = client.GetContainer("mydb", "mycontainer");

// Point read
var resp = await container.ReadItemAsync<MyDoc>("id", new PartitionKey("pk"));
Console.WriteLine($"RU: {resp.RequestCharge}");

// Paged query
var query = new QueryDefinition("SELECT * FROM c WHERE c.status = @s")
    .WithParameter("@s", "active");
using var feed = container.GetItemQueryIterator<MyDoc>(query,
    requestOptions: new QueryRequestOptions { MaxItemCount = 100 });
while (feed.HasMoreResults) {
    var page = await feed.ReadNextAsync();
    foreach (var item in page) Process(item);
}

// Bulk upsert
await Task.WhenAll(items.Select(i => container.UpsertItemAsync(i)));
```

### Node.js (@azure/cosmos)

```javascript
const { CosmosClient } = require("@azure/cosmos");
const container = new CosmosClient({ endpoint, key })
    .database("mydb").container("mycontainer");

await container.items.upsert({ id: "doc1", pk: "part1", value: 42 });

const { resources } = await container.items
    .query("SELECT * FROM c WHERE c.value > 10", { partitionKey: "part1" })
    .fetchAll();
```

---

## 10. RU Optimization & Query Tuning

| Operation | Approx RU Cost |
|-----------|---------------|
| Point read 1 KB | **1 RU** |
| Write/upsert 1 KB | **~5 RU** |
| Indexed single-partition query | **~2.5 RU** base + data |
| Cross-partition query | Higher (fan-out multiplier) |
| Strong consistency read | **2×** session cost |

```sql
-- BAD: SELECT * (all fields, more bandwidth)
SELECT * FROM c WHERE c.userId = "u123"

-- GOOD: project only needed fields
SELECT c.id, c.name, c.email FROM c WHERE c.userId = "u123"

-- BAD: UDF in WHERE (no index)
SELECT * FROM c WHERE udf.isActive(c.status) = true

-- GOOD: direct predicate
SELECT * FROM c WHERE c.status = "active"

-- Requires composite index for multi-field ORDER BY
SELECT * FROM c ORDER BY c.timestamp DESC, c.tenantId ASC
```

```python
# Log RU charge per request
response = container.upsert_item(item)
print(container.client_connection.last_response_headers["x-ms-request-charge"])
```

---

## 11. TTL & RBAC

```bash
# Enable TTL on container (-1 = items need their own _ttl field)
az cosmosdb sql container update \
  --account-name mycosmosaccount --resource-group myRG \
  --database-name mydb --name mycontainer --ttl -1
```

```python
# Set per-item TTL (seconds from last modified)
container.upsert_item({"id": "session-abc", "userId": "u001", "ttl": 3600})
```

```bash
# Assign built-in Data Contributor role to a managed identity
az cosmosdb sql role assignment create \
  --account-name mycosmosaccount --resource-group myRG \
  --role-definition-name "Cosmos DB Built-in Data Contributor" \
  --principal-id <managed-identity-object-id> \
  --scope "/dbs/mydb/colls/mycontainer"
```

---

## 12. Bicep Provisioning

```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' = {
  name: 'mycosmosaccount'
  location: resourceGroup().location
  kind: 'GlobalDocumentDB'
  properties: {
    consistencyPolicy: { defaultConsistencyLevel: 'Session' }
    locations: [{ locationName: 'East US', failoverPriority: 0, isZoneRedundant: true }]
    databaseAccountOfferType: 'Standard'
    enableAutomaticFailover: true
  }
}

resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2024-05-15' = {
  parent: cosmosAccount
  name: 'mydb'
  properties: { resource: { id: 'mydb' } }
}

resource cosmosContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2024-05-15' = {
  parent: cosmosDb
  name: 'mycontainer'
  properties: {
    resource: {
      id: 'mycontainer'
      partitionKey: { paths: ['/tenantId'], kind: 'Hash', version: 2 }
      defaultTtl: -1
      indexingPolicy: {
        indexingMode: 'consistent'
        includedPaths: [{ path: '/*' }]
        excludedPaths: [{ path: '/_etag/?' }]
        compositeIndexes: [[
          { path: '/timestamp', order: 'descending' }
          { path: '/tenantId', order: 'ascending' }
        ]]
      }
    }
    options: { autoscaleSettings: { maxThroughput: 4000 } }
  }
}
```

---

## 13. Terraform Provisioning

```hcl
resource "azurerm_cosmosdb_account" "main" {
  name                = "mycosmosaccount"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  consistency_policy { consistency_level = "Session" }

  geo_location { location = "East US",    failover_priority = 0, zone_redundant = true }
  geo_location { location = "West Europe", failover_priority = 1 }

  automatic_failover_enabled = true
}

resource "azurerm_cosmosdb_sql_container" "main" {
  name                  = "mycontainer"
  resource_group_name   = azurerm_resource_group.main.name
  account_name          = azurerm_cosmosdb_account.main.name
  database_name         = azurerm_cosmosdb_sql_database.main.name
  partition_key_paths   = ["/tenantId"]
  partition_key_version = 2
  default_ttl           = -1

  autoscale_settings { max_throughput = 4000 }

  indexing_policy {
    indexing_mode = "consistent"
    included_path { path = "/*" }
    excluded_path { path = "/_etag/?" }
    composite_index {
      index { path = "/timestamp" order = "Descending" }
      index { path = "/tenantId"  order = "Ascending" }
    }
  }
}
```

---

## 14. Cosmos DB for MongoDB (vCore)

vCore is a **dedicated cluster** model — full MongoDB 6.0 wire protocol, no RU billing.

```bash
az cosmosdb mongocluster create \
  --cluster-name mymongocluster --resource-group myRG --location eastus \
  --administrator-login adminuser \
  --administrator-login-password "P@ssw0rd!" \
  --shard-count 1 --node-tier M30 --storage 128

# Connect with standard mongosh
mongosh "mongodb+srv://adminuser:P%40ssw0rd%21@mymongocluster.mongocluster.cosmos.azure.com/?tls=true"
```

| Feature | RU-based MongoDB API | vCore |
|---------|---------------------|-------|
| Billing | Per RU consumed | Per cluster/hour |
| MongoDB compat | 4.x–6.x wire | 6.0 full |
| Scaling | Elastic RU | Vertical tier |
| Best for | Spiky workloads | Large, predictable datasets |

---

## 15. Diagnostics, Metrics & Alerts

```bash
# Enable diagnostic logs to Log Analytics
az monitor diagnostic-settings create \
  --name cosmos-diag \
  --resource $(az cosmosdb show -n mycosmosaccount -g myRG --query id -o tsv) \
  --workspace $(az monitor log-analytics workspace show -g myRG -n myWorkspace --query id -o tsv) \
  --logs '[
    {"category":"DataPlaneRequests","enabled":true},
    {"category":"QueryRuntimeStatistics","enabled":true},
    {"category":"PartitionKeyStatistics","enabled":true}
  ]'
```

```kusto
// Top expensive queries (Log Analytics KQL)
AzureDiagnostics
| where Category == "QueryRuntimeStatistics" and TimeGenerated > ago(1h)
| project TimeGenerated, collectionName_s, queryText_s, requestCharge_s
| order by todouble(requestCharge_s) desc | take 20

// Throttled (429) requests over time
AzureDiagnostics
| where Category == "DataPlaneRequests" and statusCode_s == "429"
| summarize count() by bin(TimeGenerated, 5m), collectionName_s

// Hot partitions
AzureDiagnostics
| where Category == "PartitionKeyStatistics"
| project partitionKey_s, sizeKb_d | order by sizeKb_d desc
```

```bash
# Alert when RU consumption > 80%
az monitor metrics alert create \
  --name "HighRUConsumption" --resource-group myRG \
  --scopes $(az cosmosdb show -n mycosmosaccount -g myRG --query id -o tsv) \
  --condition "avg NormalizedRUConsumption > 80" \
  --window-size 5m --evaluation-frequency 1m
```

---

## 16. Cost Optimization

| Strategy | Impact |
|----------|--------|
| **Reserved capacity** (1-yr/3-yr) | 20–47% discount on provisioned RU/s |
| **Serverless** for dev/test or < 1M ops/month | Zero idle cost |
| **Autoscale** for spiky production | No over-provisioning |
| **Exclude paths from indexing** | Reduces write RU cost |
| **Point reads over queries** | 1 RU vs 2.5+ RU |
| **TTL** to auto-delete stale data | Reduces storage cost |
| **Project fields in SELECT** | Less bandwidth, fewer RUs |
| **Shared database throughput** | Distribute 400 RU/s across multiple light containers |

```bash
# Check actual utilization to right-size
az monitor metrics list \
  --resource $(az cosmosdb show -n mycosmosaccount -g myRG --query id -o tsv) \
  --metric NormalizedRUConsumption \
  --aggregation Average Maximum \
  --start-time 2026-04-17T00:00:00Z --end-time 2026-04-24T00:00:00Z \
  --interval PT1H --output table

# Enable free tier (1000 RU/s + 25 GB free, one account per subscription)
az cosmosdb create --name mycosmosaccount --resource-group myRG \
  --enable-free-tier true \
  --locations regionName=eastus failoverPriority=0

# Point-in-time restore: enable continuous backup (up to 30 days)
az cosmosdb update --name mycosmosaccount --resource-group myRG \
  --backup-policy-type Continuous --continuous-tier Continuous30Days
```
