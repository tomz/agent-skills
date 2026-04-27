---
name: do-data
description: DigitalOcean managed databases (PostgreSQL, MySQL, Redis, MongoDB, Kafka) and Spaces object storage
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

# DigitalOcean Data Services Skill

## Managed Databases

DO manages patching, replication, backups, and failover. You just connect and query.
Supported engines: **PostgreSQL**, **MySQL**, **Redis**, **MongoDB**, **Kafka**.

---

### Creating a Database Cluster

```bash
# PostgreSQL (most common)
doctl databases create pg-prod \
  --engine pg \
  --version 16 \
  --size db-s-1vcpu-1gb \
  --region nyc3 \
  --num-nodes 1

# High-availability (2+ nodes = primary + standby with auto-failover)
doctl databases create pg-prod-ha \
  --engine pg \
  --version 16 \
  --size db-s-2vcpu-4gb \
  --region nyc3 \
  --num-nodes 2

# MySQL
doctl databases create mysql-app \
  --engine mysql \
  --version 8 \
  --size db-s-1vcpu-1gb \
  --region nyc3 \
  --num-nodes 1

# Redis (cache/queue)
doctl databases create redis-cache \
  --engine redis \
  --version 7 \
  --size db-s-1vcpu-1gb \
  --region nyc3 \
  --num-nodes 1

# MongoDB
doctl databases create mongo-app \
  --engine mongodb \
  --version 6 \
  --size db-s-1vcpu-1gb \
  --region nyc3 \
  --num-nodes 1
```

### Database size slugs

```bash
doctl databases options slugs --engine pg  # list valid sizes

# Common:
# db-s-1vcpu-1gb   — $15/mo (dev/test)
# db-s-1vcpu-2gb   — $30/mo
# db-s-2vcpu-4gb   — $60/mo
# db-s-4vcpu-8gb   — $120/mo
```

---

### Getting Connection Info

```bash
# Full connection details (host, port, user, password, database)
doctl databases connection $DB_ID

# Get just the connection URI (paste directly into apps)
doctl databases connection $DB_ID --output json | jq -r '.connection.uri'

# Private network URI (if app and DB are in same VPC — use this for production)
doctl databases connection $DB_ID --output json | jq -r '.connection.private_uri'

# List clusters
doctl databases list
doctl databases get $DB_ID
```

---

### Users and Databases

```bash
# Create a database within the cluster
doctl databases db create $DB_ID myapp_production

# Create a user
doctl databases user create $DB_ID myapp_user

# Get user credentials
doctl databases user get $DB_ID myapp_user

# List users
doctl databases user list $DB_ID

# Reset user password
doctl databases user reset-auth $DB_ID myapp_user

# Delete user
doctl databases user delete $DB_ID myapp_user
```

---

### Trusted Sources (Firewall)

By default, managed databases accept connections from anywhere (protected by credentials).
**Always restrict access** to specific IPs or your VPC.

```bash
# Restrict to specific IPs
doctl databases firewalls append $DB_ID \
  --rule "ip_addr:203.0.113.10" \
  --rule "ip_addr:10.0.0.0/8"

# Restrict to a specific droplet
doctl databases firewalls append $DB_ID \
  --rule "droplet:$DROPLET_ID"

# Restrict to all droplets with a tag
doctl databases firewalls append $DB_ID \
  --rule "tag:role:backend"

# Restrict to a Kubernetes cluster
doctl databases firewalls append $DB_ID \
  --rule "k8s:$CLUSTER_ID"

# View current rules
doctl databases firewalls list $DB_ID

# Replace all rules at once
doctl databases firewalls replace $DB_ID \
  --rule "tag:env:production"
```

---

### Connection Pools (PostgreSQL)

Connection pooling (PgBouncer) sits between your app and Postgres, reducing connection overhead.
Essential for apps with many concurrent connections (Rails, Django with gunicorn, etc.)

```bash
# Create a pool
doctl databases pool create $DB_ID \
  --name app-pool \
  --db myapp_production \
  --user myapp_user \
  --size 10 \
  --mode transaction   # transaction | session | statement

# Get pool connection string
doctl databases pool get $DB_ID app-pool --output json | jq -r '.connection.uri'

# List pools
doctl databases pool list $DB_ID

# Delete pool
doctl databases pool delete $DB_ID app-pool
```

Pool modes:
- `transaction` — connection released after each transaction (recommended for most apps)
- `session` — connection held for entire client session (use with SET LOCAL, prepared statements)
- `statement` — released after each statement (no multi-statement transactions)

---

### Read Replicas

Add read replicas to scale read-heavy workloads. They're separate clusters with replication.

```bash
# Create a replica of an existing cluster
doctl databases replicas create $DB_ID replica-01 \
  --region sfo3 \
  --size db-s-1vcpu-1gb

# Get replica connection info
doctl databases replicas connection $DB_ID replica-01

# Promote replica to standalone (disaster recovery)
doctl databases replicas promote $DB_ID replica-01
```

---

### Maintenance Windows

```bash
# Set maintenance window (updates, minor version upgrades happen here)
doctl databases maintenance-window update $DB_ID \
  --day sunday \
  --hour 03:00  # UTC

# View current window
doctl databases get $DB_ID --output json | jq '.maintenance_window'
```

---

### Backups and Restores

DO takes daily backups automatically (7-day retention on basic plans).

```bash
# List backups
doctl databases backups list $DB_ID

# Restore to a new cluster from backup
doctl databases create pg-restored \
  --engine pg \
  --version 16 \
  --size db-s-1vcpu-1gb \
  --region nyc3 \
  --restore-from-cluster-name pg-prod \
  --restore-from-timestamp "2024-01-15T02:00:00Z"
```

---

## Spaces (Object Storage)

Spaces is DO's S3-compatible object storage. Works with `s3cmd`, `aws cli`, `boto3`, any S3 SDK.

### Key concepts

- **Bucket** = Space (globally unique name within a region)
- **Endpoint**: `https://<region>.digitaloceanspaces.com`
- **CDN endpoint**: `https://<space-name>.<region>.cdn.digitaloceanspaces.com`
- **Access keys**: separate from API tokens — created at Settings → Spaces Keys

### Setup with AWS CLI (most compatible)

```bash
# Configure AWS CLI for Spaces
aws configure set aws_access_key_id $SPACES_KEY
aws configure set aws_secret_access_key $SPACES_SECRET
aws configure set region nyc3

# Use --endpoint-url for every command
aws s3 ls --endpoint-url https://nyc3.digitaloceanspaces.com
```

### Setup with s3cmd

```bash
s3cmd --configure  # enter Spaces key/secret, set endpoint to nyc3.digitaloceanspaces.com

# Or write config directly
cat > ~/.s3cfg << EOF
access_key = $SPACES_KEY
secret_key = $SPACES_SECRET
host_base = nyc3.digitaloceanspaces.com
host_bucket = %(bucket)s.nyc3.digitaloceanspaces.com
EOF
```

### Common Spaces operations

```bash
# Create a Space
doctl spaces create my-bucket --region nyc3

# List Spaces
doctl spaces list

# Upload file
doctl spaces object put my-bucket local-file.zip --remote-path uploads/file.zip

# Upload directory (recursive)
aws s3 sync ./dist/ s3://my-bucket/static/ \
  --endpoint-url https://nyc3.digitaloceanspaces.com \
  --acl public-read

# Download
aws s3 cp s3://my-bucket/file.zip ./file.zip \
  --endpoint-url https://nyc3.digitaloceanspaces.com

# List objects
aws s3 ls s3://my-bucket/ --endpoint-url https://nyc3.digitaloceanspaces.com

# Delete object
aws s3 rm s3://my-bucket/old-file.zip \
  --endpoint-url https://nyc3.digitaloceanspaces.com

# Presigned URL (time-limited download link)
aws s3 presign s3://my-bucket/report.pdf \
  --endpoint-url https://nyc3.digitaloceanspaces.com \
  --expires-in 3600
```

### CDN

Enable CDN on a Space to serve files from edge locations:

```bash
# Enable CDN for a Space
doctl compute cdn create \
  --origin my-bucket.nyc3.digitaloceanspaces.com \
  --ttl 3600

# Files served at: https://my-bucket.nyc3.cdn.digitaloceanspaces.com/path/to/file

# List CDN endpoints
doctl compute cdn list

# Flush CDN cache (after updating files)
doctl compute cdn flush $CDN_ID --files "static/*"
```

### Lifecycle policies (auto-delete old objects)

Use the S3 API via AWS CLI:

```bash
# Create lifecycle policy JSON
cat > lifecycle.json << 'EOF'
{
  "Rules": [{
    "ID": "delete-old-logs",
    "Filter": {"Prefix": "logs/"},
    "Status": "Enabled",
    "Expiration": {"Days": 30}
  }]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json \
  --endpoint-url https://nyc3.digitaloceanspaces.com
```

---

## Guardrails

- **Use private URIs** when app and DB are in the same VPC — never expose DB publicly
- **Set firewall rules** immediately after creating a database — default is open
- **Connection pools** are essential for PostgreSQL with >20 app instances
- **Spaces keys ≠ API tokens** — generate at Settings → Spaces Keys
- **Bucket names are globally unique per region** — use project-specific prefixes
- **CDN caches aggressively** — always flush after deployments if serving assets
- **Read replicas add latency** — measure replication lag before routing queries
- **`--num-nodes 2`** for production — enables auto-failover; single node has no HA
