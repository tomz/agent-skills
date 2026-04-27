---
name: azure-redis
description: Azure Cache for Redis — caching patterns, tiers, clustering, geo-replication, data persistence, Redis Enterprise, session store
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Cache for Redis Skill

Azure Cache for Redis is a fully managed in-memory data store based on Redis,
offering sub-millisecond latency for caching, session stores, pub/sub,
distributed locks, leaderboards, and real-time analytics.

---

## Tiers Comparison

| Tier | Max Memory | Clustering | Geo-replication | Persistence | Modules | Notes |
|------|-----------|------------|----------------|-------------|---------|-------|
| **Basic** | 53 GB | No | No | No | No | Single node, no SLA — dev/test only |
| **Standard** | 53 GB | No | Passive | No | No | 2 nodes (primary + replica), 99.9% SLA |
| **Premium** | 1.2 TB | Yes (10 shards) | Passive | RDB + AOF | No | 99.9% SLA, VNet injection, zones |
| **Enterprise** | 14 TB | Yes (OSS) | Active-Active | RDB + AOF | Yes (RediSearch, etc.) | 99.99% SLA, Redis Software stack |
| **Enterprise Flash** | 1.5 TB RAM + NVMe | Yes | Active-Active | RDB + AOF | Yes | Cost-optimised for large datasets |

---

## Create & Manage

### Basic / Standard / Premium

```bash
# Create Standard C2 (6 GB) cache
az redis create \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --location "eastus" \
  --sku Standard \
  --vm-size C2 \
  --minimum-tls-version "1.2" \
  --enable-non-ssl-port false

# Create Premium P1 (6 GB) with zones
az redis create \
  --name "redis-premium-prod" \
  --resource-group "rg-cache" \
  --location "eastus" \
  --sku Premium \
  --vm-size P1 \
  --zones 1 2 3 \
  --minimum-tls-version "1.2" \
  --enable-non-ssl-port false \
  --redis-configuration '{"maxmemory-policy":"volatile-lru","maxmemory-reserved":"50","maxfragmentationmemory-reserved":"50","notify-keyspace-events":""}'

# List caches
az redis list --resource-group "rg-cache" \
  --query "[].{name:name, sku:sku.name, size:sku.family, gb:sku.capacity, state:provisioningState}" \
  -o table

# Get connection details
az redis show \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --query "{host:hostName, sslPort:sslPort, enableNonSsl:enableNonSslPort}"

# Get access keys
az redis list-keys \
  --name "redis-prod-001" \
  --resource-group "rg-cache"

# Regenerate primary key
az redis regenerate-keys \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --key-type Primary

# Update redis configuration
az redis update \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --set "redisConfiguration.maxmemory-policy=allkeys-lru"

# Delete cache
az redis delete \
  --name "redis-prod-001" \
  --resource-group "rg-cache" --yes
```

### SKU Sizes

| SKU | Family | Capacity | Memory |
|-----|--------|---------|--------|
| Basic/Standard | C | 0–6 | 250 MB – 53 GB |
| Premium | P | 1–5 | 6 GB – 120 GB |
| Enterprise | - | varies | up to 14 TB |

```bash
# Scale up (Standard C1 → C2)
az redis update \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --sku Standard \
  --vm-size C2

# Scale to Premium (non-reversible from Premium back to Standard)
az redis update \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --sku Premium \
  --vm-size P1
```

---

## Clustering (Premium)

```bash
# Create Premium with 3 shards (each shard = primary + replica)
az redis create \
  --name "redis-cluster-prod" \
  --resource-group "rg-cache" \
  --location "eastus" \
  --sku Premium \
  --vm-size P3 \
  --shard-count 3 \
  --minimum-tls-version "1.2"

# Update shard count (scale out — only increase supported)
az redis update \
  --name "redis-cluster-prod" \
  --resource-group "rg-cache" \
  --shard-count 5
```

**Clustering client note:** Use StackExchange.Redis with cluster-aware config:

```csharp
// C# — StackExchange.Redis cluster connection
var config = new ConfigurationOptions {
    EndPoints = { "redis-cluster-prod.redis.cache.windows.net:6380" },
    Password = "<primary-key>",
    Ssl = true,
    SslProtocols = SslProtocols.Tls12,
    AbortOnConnectFail = false,
    ConnectTimeout = 5000,
    SyncTimeout = 5000,
};
// StackExchange.Redis handles cluster topology automatically
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect(config);
```

---

## Geo-Replication

### Passive Geo-Replication (Standard / Premium)

```bash
# Create primary (Premium, eastus)
az redis create \
  --name "redis-geo-primary" \
  --resource-group "rg-cache-east" \
  --location "eastus" \
  --sku Premium --vm-size P1

# Create secondary (same SKU/size, different region)
az redis create \
  --name "redis-geo-secondary" \
  --resource-group "rg-cache-west" \
  --location "westus2" \
  --sku Premium --vm-size P1

# Link as geo-replication pair
az redis geo-replication link create \
  --name "redis-geo-primary" \
  --resource-group "rg-cache-east" \
  --server-to-link "/subscriptions/SUB_ID/resourceGroups/rg-cache-west/providers/Microsoft.Cache/Redis/redis-geo-secondary" \
  --secondary-name "redis-geo-secondary"

# Check replication status
az redis geo-replication link list \
  --name "redis-geo-primary" \
  --resource-group "rg-cache-east"

# Unlink (before deleting secondary)
az redis geo-replication link delete \
  --name "redis-geo-primary" \
  --resource-group "rg-cache-east" \
  --linked-server-name "redis-geo-secondary"
```

### Active-Active Geo-Replication (Enterprise tier)

Enterprise active-active uses Redis CRDTs — all nodes accept writes and converge.

```bash
# Create Enterprise E10 in eastus
az redisenterprise create \
  --cluster-name "redis-ent-east" \
  --resource-group "rg-cache-east" \
  --location "eastus" \
  --sku "Enterprise_E10" \
  --minimum-tls-version "1.2"

# Create database with geo-replication
az redisenterprise database create \
  --cluster-name "redis-ent-east" \
  --resource-group "rg-cache-east" \
  --name "default" \
  --geo-replication '{
    "groupNickname": "geo-group-prod",
    "linkedDatabases": [
      {
        "id": "/subscriptions/SUB_ID/resourceGroups/rg-cache-east/providers/Microsoft.Cache/redisEnterprise/redis-ent-east/databases/default"
      },
      {
        "id": "/subscriptions/SUB_ID/resourceGroups/rg-cache-west/providers/Microsoft.Cache/redisEnterprise/redis-ent-west/databases/default"
      }
    ]
  }' \
  --clustering-policy EnterpriseCluster \
  --eviction-policy NoEviction \
  --modules '[{"name":"RediSearch"},{"name":"RedisJSON"}]'

# List Enterprise clusters
az redisenterprise list --resource-group "rg-cache-east" -o table
```

---

## Data Persistence (Premium)

```bash
# Enable RDB persistence (snapshot every 60 minutes)
az redis update \
  --name "redis-premium-prod" \
  --resource-group "rg-cache" \
  --set "redisConfiguration.rdb-backup-enabled=true" \
  --set "redisConfiguration.rdb-backup-frequency=60" \
  --set "redisConfiguration.rdb-backup-max-snapshot-count=1" \
  --set "redisConfiguration.rdb-storage-connection-string=DefaultEndpointsProtocol=https;AccountName=stredisbackup;AccountKey=<key>;EndpointSuffix=core.windows.net"

# Enable AOF persistence (appendfsync = everysec)
az redis update \
  --name "redis-premium-prod" \
  --resource-group "rg-cache" \
  --set "redisConfiguration.aof-backup-enabled=true" \
  --set "redisConfiguration.aof-storage-connection-string-0=<storage-conn-str>" \
  --set "redisConfiguration.aof-storage-connection-string-1=<storage-conn-str-replica>"

# Import RDB backup (restore from snapshot)
az redis import \
  --name "redis-premium-prod" \
  --resource-group "rg-cache" \
  --files "https://stredisbackup.blob.core.windows.net/backups/dump.rdb"

# Export current dataset to blob
az redis export \
  --name "redis-premium-prod" \
  --resource-group "rg-cache" \
  --prefix "backup-$(date +%Y%m%d)" \
  --container "https://stredisbackup.blob.core.windows.net/backups" \
  --storage-account-key "<key>" \
  --file-format RDB
```

---

## Redis Modules (Enterprise Tier)

| Module | Use Case | Key Commands |
|--------|----------|-------------|
| **RediSearch** | Full-text search + secondary indexes | `FT.CREATE`, `FT.SEARCH`, `FT.AGGREGATE` |
| **RedisJSON** | Native JSON document store | `JSON.SET`, `JSON.GET`, `JSON.ARRAPPEND` |
| **RedisTimeSeries** | Time series data at scale | `TS.CREATE`, `TS.ADD`, `TS.RANGE`, `TS.MRANGE` |
| **RedisBloom** | Probabilistic (Bloom, Cuckoo, TopK, HLL) | `BF.ADD`, `BF.EXISTS`, `CF.ADD` |

```bash
# RediSearch — create index and search (redis-cli examples)
FT.CREATE idx:products ON JSON PREFIX 1 product: SCHEMA $.name AS name TEXT $.price AS price NUMERIC SORTABLE $.category AS category TAG
FT.SEARCH idx:products "@category:{electronics} @price:[100 500]" LIMIT 0 10 SORTBY price ASC

# RedisJSON
JSON.SET user:1001 $ '{"name":"Alice","email":"alice@company.com","role":"admin"}'
JSON.GET user:1001 $.name
JSON.ARRAPPEND user:1001 $.roles '"viewer"'

# RedisTimeSeries
TS.CREATE sensor:temp:room1 RETENTION 86400000 LABELS location room1 unit celsius
TS.ADD sensor:temp:room1 * 22.5
TS.RANGE sensor:temp:room1 - + AGGREGATION avg 60000  # 1-min averages

# RedisBloom
BF.RESERVE visited:urls 0.01 1000000  # 1% error rate, 1M capacity
BF.ADD visited:urls "https://example.com/page"
BF.EXISTS visited:urls "https://example.com/page"  # Returns 1 (probably yes)
```

---

## Caching Patterns

### Cache-Aside (Lazy Loading)

```python
import redis, json, hashlib

r = redis.Redis(
    host="redis-prod-001.redis.cache.windows.net",
    port=6380, ssl=True,
    password="<primary-key>",
    decode_responses=True
)

def get_user(user_id: str) -> dict:
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)                   # Cache HIT

    user = db.query("SELECT * FROM users WHERE id=?", user_id)  # Cache MISS
    r.setex(cache_key, 300, json.dumps(user))       # Cache for 5 minutes
    return user
```

### Write-Through

```python
def update_user(user_id: str, data: dict):
    db.execute("UPDATE users SET ... WHERE id=?", user_id, data)   # Write DB first
    cache_key = f"user:{user_id}"
    r.setex(cache_key, 300, json.dumps(data))                       # Then update cache
```

### Write-Behind (Write-Back)

```python
# Write to cache immediately; async worker flushes to DB
def update_user_writeback(user_id: str, data: dict):
    cache_key = f"user:{user_id}"
    dirty_key = f"dirty:users"
    pipeline = r.pipeline()
    pipeline.setex(cache_key, 3600, json.dumps(data))
    pipeline.sadd(dirty_key, user_id)               # Mark as dirty
    pipeline.execute()

# Background worker
def flush_dirty_users():
    dirty_ids = r.smembers("dirty:users")
    for uid in dirty_ids:
        data = r.get(f"user:{uid}")
        if data:
            db.execute("UPDATE users SET ... WHERE id=?", uid, json.loads(data))
            r.srem("dirty:users", uid)
```

### Distributed Lock (Redlock pattern)

```python
import uuid, time

def acquire_lock(lock_name: str, ttl_ms: int = 5000) -> str | None:
    token = str(uuid.uuid4())
    acquired = r.set(f"lock:{lock_name}", token, px=ttl_ms, nx=True)
    return token if acquired else None

def release_lock(lock_name: str, token: str) -> bool:
    # Lua script ensures atomic check-and-delete
    script = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
    else return 0 end
    """
    return bool(r.eval(script, 1, f"lock:{lock_name}", token))

# Usage
token = acquire_lock("order-processor", ttl_ms=10000)
if token:
    try:
        process_order()
    finally:
        release_lock("order-processor", token)
```

### Session State Store

```python
# Flask session with Redis
from flask import Flask
from flask_session import Session

app = Flask(__name__)
app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_REDIS"] = r
app.config["SESSION_PERMANENT"] = False
app.config["SESSION_USE_SIGNER"] = True
app.config["SECRET_KEY"] = "super-secret"
Session(app)
```

### Pub/Sub

```python
# Publisher
def publish_event(channel: str, event: dict):
    r.publish(channel, json.dumps(event))

# Subscriber (blocking)
def subscribe_events(channel: str):
    pubsub = r.pubsub()
    pubsub.subscribe(channel)
    for message in pubsub.listen():
        if message["type"] == "message":
            event = json.loads(message["data"])
            handle_event(event)
```

---

## Security

### Private Endpoint (Premium+)

```bash
# Disable public access
az redis update \
  --name "redis-premium-prod" \
  --resource-group "rg-cache" \
  --set "publicNetworkAccess=Disabled"

# Create private endpoint
az network private-endpoint create \
  --name "pe-redis-prod" \
  --resource-group "rg-cache" \
  --vnet-name "vnet-prod" \
  --subnet "subnet-private-endpoints" \
  --private-connection-resource-id "/subscriptions/SUB_ID/resourceGroups/rg-cache/providers/Microsoft.Cache/Redis/redis-premium-prod" \
  --connection-name "redis-pe-conn" \
  --group-ids "redisCache"

# DNS zone
az network private-dns zone create -g "rg-cache" -n "privatelink.redis.cache.windows.net"
az network private-dns link vnet create \
  -g "rg-cache" \
  -z "privatelink.redis.cache.windows.net" \
  -n "redis-dns-link" \
  --virtual-network "vnet-prod" \
  --registration-enabled false
az network private-endpoint dns-zone-group create \
  --endpoint-name "pe-redis-prod" \
  -g "rg-cache" \
  -n "redis-dns-zone-group" \
  --zone-name "privatelink.redis.cache.windows.net" \
  --private-dns-zone "privatelink.redis.cache.windows.net"
```

### Firewall Rules

```bash
# Allow specific IP (when public access is on)
az redis firewall-rules create \
  --name "redis-prod-001" \
  --resource-group "rg-cache" \
  --rule-name "allow-office" \
  --start-ip "203.0.113.10" \
  --end-ip "203.0.113.10"

# List rules
az redis firewall-rules list \
  --name "redis-prod-001" \
  --resource-group "rg-cache"
```

### Entra ID Authentication (Preview — Enterprise)

```bash
# Enterprise tier: enable Entra ID auth
az redisenterprise database update \
  --cluster-name "redis-ent-east" \
  --resource-group "rg-cache-east" \
  --name "default" \
  --access-keys-authentication Disabled

# Assign role to managed identity
az role assignment create \
  --role "Redis Cache Contributor" \
  --assignee-object-id "<managed-identity-oid>" \
  --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/SUB_ID/resourceGroups/rg-cache-east/providers/Microsoft.Cache/redisEnterprise/redis-ent-east"
```

---

## Monitoring

```bash
# Key metrics to alert on:
# - CacheHitRate < 80% → insufficient cache population
# - UsedMemoryPercentage > 80% → scale up or adjust eviction policy
# - ConnectedClients spike → connection leak
# - ServerLoad > 80% → scale up CPU

# Create alert — cache hit rate < 80%
az monitor metrics alert create \
  --name "redis-low-hit-rate" \
  --resource-group "rg-cache" \
  --scopes "/subscriptions/SUB_ID/resourceGroups/rg-cache/providers/Microsoft.Cache/Redis/redis-prod-001" \
  --condition "avg CacheHits < 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action "/subscriptions/SUB_ID/resourceGroups/rg-monitoring/providers/microsoft.insights/actionGroups/ops-team"

# Create alert — memory pressure
az monitor metrics alert create \
  --name "redis-high-memory" \
  --resource-group "rg-cache" \
  --scopes "/subscriptions/SUB_ID/resourceGroups/rg-cache/providers/Microsoft.Cache/Redis/redis-prod-001" \
  --condition "avg UsedMemoryPercentage > 85" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1

# Enable diagnostic logs
az monitor diagnostic-settings create \
  --name "redis-diag" \
  --resource "/subscriptions/SUB_ID/resourceGroups/rg-cache/providers/Microsoft.Cache/Redis/redis-prod-001" \
  --logs '[{"category":"ConnectedClientList","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --workspace "/subscriptions/SUB_ID/resourceGroups/rg-mon/providers/Microsoft.OperationalInsights/workspaces/law-prod"
```

### Maxmemory Eviction Policies

| Policy | Behavior |
|--------|----------|
| `noeviction` | Return error when memory full (default) |
| `allkeys-lru` | Evict any key using LRU — **best for general caching** |
| `volatile-lru` | Evict keys with TTL using LRU |
| `allkeys-lfu` | Evict any key using LFU (Redis 4.0+) |
| `volatile-lfu` | Evict keys with TTL using LFU |
| `allkeys-random` | Evict random keys |
| `volatile-random` | Evict random keys with TTL |
| `volatile-ttl` | Evict keys with shortest TTL first |

---

## Bicep — Redis Cache

```bicep
param location string = 'eastus'
param redisCacheName string = 'redis-prod-001'

resource redisCache 'Microsoft.Cache/redis@2023-08-01' = {
  name: redisCacheName
  location: location
  properties: {
    sku: {
      name: 'Premium'
      family: 'P'
      capacity: 1       // P1 = 6 GB
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'
    redisConfiguration: {
      'maxmemory-policy': 'allkeys-lru'
      'maxmemory-reserved': '50'
      'maxfragmentationmemory-reserved': '50'
      'rdb-backup-enabled': 'true'
      'rdb-backup-frequency': '60'
      'rdb-backup-max-snapshot-count': '1'
      'rdb-storage-connection-string': rdbStorageConnectionString
    }
    shardCount: 2
    replicasPerMaster: 1
    zones: ['1', '2', '3']
  }
}

// Output connection string as Key Vault secret
resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'kv-prod'
  scope: resourceGroup('rg-security')
}

resource redisSecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: kv
  name: 'redis-connection-string'
  properties: {
    value: '${redisCache.properties.hostName}:${redisCache.properties.sslPort},password=${redisCache.listKeys().primaryKey},ssl=True,abortConnect=False'
  }
}
```

---

## Connection String Patterns

```bash
# StackExchange.Redis (C#)
# "redis-prod-001.redis.cache.windows.net:6380,password=<key>,ssl=True,abortConnect=False"

# Python (redis-py)
r = redis.Redis(host="redis-prod-001.redis.cache.windows.net", port=6380, password="<key>", ssl=True)

# Node.js (ioredis)
# const client = new Redis({ host: "redis-prod-001.redis.cache.windows.net", port: 6380, password: "<key>", tls: {} })

# Java (Jedis)
# JedisPool pool = new JedisPool(new JedisPoolConfig(), "redis-prod-001.redis.cache.windows.net", 6380, 2000, "<key>", true)

# Get connection string via CLI
az redis show --name "redis-prod-001" -g "rg-cache" --query "hostName" -o tsv
PRIMARY_KEY=$(az redis list-keys --name "redis-prod-001" -g "rg-cache" --query primaryKey -o tsv)
echo "redis-prod-001.redis.cache.windows.net:6380,password=${PRIMARY_KEY},ssl=True,abortConnect=False"
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Create Standard C2 | `az redis create --sku Standard --vm-size C2` |
| Create Premium P1 | `az redis create --sku Premium --vm-size P1` |
| Get access keys | `az redis list-keys` |
| Rotate key | `az redis regenerate-keys --key-type Primary` |
| Enable clustering | `--shard-count N` on create or `az redis update --shard-count N` |
| Passive geo-link | `az redis geo-replication link create` |
| Enable RDB persistence | `az redis update --set redisConfiguration.rdb-backup-enabled=true` |
| Create firewall rule | `az redis firewall-rules create` |
| Disable public access | `az redis update --set publicNetworkAccess=Disabled` |
| Scale up | `az redis update --sku Premium --vm-size P2` |
| Export data | `az redis export` |
| Import data | `az redis import --files <blob-url>` |
