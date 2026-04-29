---
name: azure-hdinsight-migration-esp-to-non-esp
description: End-to-end migration playbook for moving Azure HDInsight Enterprise Security Package (ESP) clusters to non-ESP HDInsight before ESP retirement (end of July 2026), with compensating security controls — VNet isolation, Private Endpoints, Managed Identity for storage (no keys), per-cluster local auth, Azure RBAC + ABAC for ADLS Gen2, Azure Policy guardrails, Key Vault secrets, Log Analytics + Sentinel auditing, Bastion-only SSH, NSG/ASG firewalling, OneLake/Purview data-plane protections, and Ranger replacement strategies plus Microsoft Purview for data lineage (Atlas is not part of HDI ESP).
license: MIT
version: 1.0.0
updated: 2026-04-29
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
triggers: hdinsight esp, esp retirement, esp migration, non-esp hdinsight, aad ds, azure ad domain services, ranger hdi, kerberos hdinsight, hdi security, hdi private endpoint, hdi managed identity, hdi vnet, ldaps hdi
requires: azure-hdinsight
---
# Migrating HDInsight ESP → Non-ESP (with Compensating Security Controls)

A field guide for migrating production **HDInsight Enterprise Security Package
(ESP)** clusters — Kerberos via Azure AD Domain Services (AAD DS) + Apache Ranger
— to **non-ESP** HDInsight clusters **before ESP retirement on
31 July 2026**, while preserving (or improving) the security posture using
Azure-native controls instead of the Hadoop-ecosystem ones.

> **Why this migration is mandatory.**
> Microsoft has announced the **retirement of HDInsight ESP at end of July 2026**.
> After that date, ESP-enabled clusters will not be supported and AAD DS integration
> plus Ranger will not be available on HDI. **All ESP clusters must be either
> migrated to Fabric/Databricks/Synapse, or re-deployed as non-ESP HDInsight clusters
> with compensating Azure-native controls — before the deadline.**
>
> This skill focuses on **path B** (stay on HDI, drop ESP). For full re-platforms, see:
> - `azure-hdinsight-migration-spark-to-fabric`
> - `azure-hdinsight-migration-interactive-query-to-fabric-spark-or-dw`
> - `azure-hdinsight-migration-kafka-to-fabric-rti`
> - `azure-hdinsight-migration-hbase-to-fabric-cosmosdb`

> **Note on Apache Atlas**: Atlas is **not** a standard HDInsight component and is
> **not** bundled in ESP. The ESP feature set is **AAD DS + Kerberos + Ranger** (plus
> the audit DB Ranger writes to). If your team self-installed Atlas via custom
> script actions, treat it as custom infrastructure outside this migration's scope.
> **Microsoft Purview** is the Azure-native replacement for data lineage and
> classification regardless of whether you ran Atlas on HDI or not.

---

## 1. Decision Matrix — Stay on HDI Non-ESP vs Re-platform

| Driver | Stay on HDI Non-ESP | Re-platform (Fabric / Databricks / Synapse) |
|--------|---------------------|---------------------------------------------|
| Hard deadline pressure (months left) | ✅ Faster path | ⚠ Re-platforms often run >6 months |
| Existing scripts/JARs deeply Hadoop-coupled | ✅ Minimal code change | ⚠ Significant rewrite |
| Workload uses HBase / Storm / Pig | ✅ HDI still supports these | ❌ No equivalents in Fabric |
| Multi-user interactive analytics | ⚠ Lose Ranger fine-grained ACLs | ✅ Workspace RBAC + SQL endpoint RLS |
| Cost-sensitive, idle clusters | ❌ Always-on head nodes | ✅ Serverless capacity (Fabric / Databricks SQL) |
| Strong audit / compliance requirements | ✅ With Log Analytics + Sentinel + Defender | ✅ Native Fabric/Databricks audit logs |
| Strategic direction = Microsoft Fabric | ⚠ Tactical only — schedule re-platform later | ✅ Strategic |

**Decision shortcut**: If you have **<6 months** to ESP retirement and the workload
isn't in scope for a Fabric/Databricks migration this fiscal year, **drop ESP, stay on
HDI**, harden with Azure-native controls. Schedule the strategic re-platform as a
separate Q+1 / Q+2 initiative.

---

## 2. What ESP Provides Today — and the Gap to Fill

| ESP Feature | Mechanism | Compensating Control on Non-ESP |
|-------------|-----------|----------------------------------|
| **Domain join** to AAD DS | Kerberos realm | Local cluster admin + AAD-backed tools (Bastion / Storage / Key Vault) |
| **SSH/Ambari login as AAD user** | LDAP via AAD DS | SSH-key-only access via **Azure Bastion**; AAD authentication for the storage and management planes |
| **Hive/HBase/Kafka Kerberos auth** | GSSAPI / SPN | **Network isolation** (private cluster, no public ingress); workload identity via **Managed Identity for storage** |
| **Apache Ranger fine-grained ACL** (db/table/column/row) | Ranger admin + plugins | **ADLS Gen2 ABAC + RBAC** for data-plane; **per-table physical separation** (one storage container per security tier); Ranger no longer available |
| **Apache Atlas data lineage** *(not bundled in ESP — only present if self-installed via custom script action)* | Atlas catalog | **Microsoft Purview** (managed catalog) — connects to ADLS, Synapse, Databricks, Fabric |
| **Service-account (kinit) per-job auth** | Keytab files | **Managed Identity** for cluster's compute → ADLS Gen2; per-job identity isolation via **MSI per cluster** |
| **Audit logs** (Ranger audit DB) | DB-backed | **Diagnostic Settings → Log Analytics → Sentinel**; Storage account diagnostic logs for data-plane; Defender for Storage / Key Vault |

> **Critical reality**: There is **no fine-grained-table-ACL replacement** on non-ESP
> HDI. If Ranger column/row-level policies are load-bearing for compliance, you have
> two real options:
> 1. **Re-platform** to Fabric SQL endpoint / Databricks Unity Catalog (where Spark SQL
>    + RLS/CLS exist).
> 2. **Physically separate** data into multiple storage containers/accounts, each gated
>    by its own RBAC/ABAC, and run separate clusters per tenant/role.

---

## 3. Assessment Phase

### 3.1 Inventory the ESP Cluster

Run on a head node (SSH, AAD-domain-joined login).

```bash
ssh someone@MYDOMAIN.COM@<cluster>-ssh.azurehdinsight.net   # ESP-style login

# Cluster + ESP metadata
curl -s -u admin:$PASS \
  "https://<cluster>.azurehdinsight.net/api/v1/clusters/<cluster>" \
  | python3 -m json.tool > cluster_meta.json

# Confirm ESP is enabled
az hdinsight show -g $RG -n <cluster> --query "properties.securityProfile" -o json \
  > security_profile.json

# AAD DS / domain join details
realm list 2>/dev/null > realms.txt
sudo cat /etc/krb5.conf > krb5.conf.backup
sudo cat /etc/security/keytabs/*.keytab 2>/dev/null | head   # don't dump full
ls -la /etc/security/keytabs/ > keytabs_list.txt

# Cluster-level config touching Kerberos / LDAP
sudo grep -rE 'principal|keytab|kerberos|ldap' \
  /etc/hadoop/conf /etc/hive/conf /etc/spark*/conf /etc/hbase/conf /etc/kafka*/conf \
  2>/dev/null > kerberos_configs.txt
```

### 3.2 Inventory Ranger Policies

```bash
# Ranger admin REST — pull every policy in every service
RANGER_URL="https://<cluster>.azurehdinsight.net/ranger"
RANGER_AUTH="-u admin:$RANGER_PASS"

# Services (hive, hbase, kafka, hdfs) — note: Atlas is NOT part of standard ESP
curl -s $RANGER_AUTH "$RANGER_URL/service/public/v2/api/service" \
  | python3 -m json.tool > ranger_services.json

# All policies across all services
curl -s $RANGER_AUTH "$RANGER_URL/service/public/v2/api/policy?pageSize=10000" \
  | python3 -m json.tool > ranger_policies.json

# Audit logs (last 90 days) — pull for compliance archive
curl -s $RANGER_AUTH \
  "$RANGER_URL/service/assets/accessAudit?startDate=$(date -u -d '90 days ago' +%F)&pageSize=100000" \
  > ranger_audits_last90d.json
```

```bash
# Categorize policies by access type — this drives the compensating-control plan
python3 << 'EOF'
import json
from collections import Counter
with open("ranger_policies.json") as f: pol = json.load(f)
buckets = Counter()
for p in pol.get("policies", []):
    svc  = p["serviceType"]
    res  = ", ".join(p.get("resources", {}).keys())
    accs = sorted({a["accesses"][0]["type"] for a in p.get("policyItems", []) if a.get("accesses")})
    has_col = "column" in res
    has_row = bool(p.get("rowFilterPolicyItems"))
    has_mask= bool(p.get("dataMaskPolicyItems"))
    bucket = (svc,
              "col-level" if has_col else "table-level",
              "row-filter" if has_row else "no-row-filter",
              "masking" if has_mask else "no-mask")
    buckets[bucket] += 1
for k, n in sorted(buckets.items(), key=lambda x: -x[1]):
    print(f'{n:5d}  {k}')
EOF
```

| Policy Bucket | Compensating Control Path |
|---------------|---------------------------|
| Service-level / table-level READ for an AAD group | **ADLS Gen2 RBAC** (Storage Blob Data Reader on container scope) |
| Service-level / table-level WRITE for an AAD group | **ADLS Gen2 RBAC** (Storage Blob Data Contributor on container scope) |
| Path-prefix grant (e.g. `/data/sales/*`) | **ADLS Gen2 ABAC condition** on `@Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs:path]` |
| Column-level grant | **No 1:1 control** — physical separation (one container per sensitivity tier) OR re-platform |
| Row-filter (`region = '${user.region}'`) | **No 1:1 control** — re-platform to Fabric/Synapse/Databricks for SQL RLS |
| Column mask (e.g. mask SSN) | **No 1:1 control** — re-platform OR pre-redact in ETL stage |
| Tag-based policy (Atlas tag → Ranger, only if Atlas was self-installed) | **Purview** classifications + custom policy via Purview-driven ETL gates |

### 3.3 Inventory Apache Atlas (Only If Self-Installed)

> **Skip this section unless your team explicitly installed Atlas via custom script
> action.** Atlas is **not** part of HDInsight ESP and is **not** present on a
> standard ESP cluster — `https://<cluster>.azurehdinsight.net/atlas` will return 404
> on default deployments. ESP ships AAD DS + Kerberos + Ranger only.

If you DID self-install Atlas (look for `*atlas*.sh` script actions in the cluster's
script action history), export entities and classifications before decommissioning so
Purview can ingest the equivalents:

```bash
# Only run if Atlas was self-installed; otherwise skip
ATLAS_URL="https://<cluster>.azurehdinsight.net/atlas"
ATLAS_AUTH="-u admin:$ATLAS_PASS"

# Confirm Atlas is actually reachable
if curl -fs -o /dev/null $ATLAS_AUTH "$ATLAS_URL/api/atlas/admin/version"; then
  curl -s $ATLAS_AUTH "$ATLAS_URL/api/atlas/v2/types/typedefs?type=ENTITY" \
    | python3 -m json.tool > atlas_entity_types.json
  curl -s $ATLAS_AUTH "$ATLAS_URL/api/atlas/v2/types/typedefs?type=CLASSIFICATION" \
    > atlas_classifications.json
  for type in hive_table hive_db hive_column hbase_table kafka_topic; do
    curl -s $ATLAS_AUTH \
      "$ATLAS_URL/api/atlas/v2/search/basic?typeName=$type&limit=10000" \
      > "atlas_export_${type}.json"
  done
else
  echo "Atlas not present on this cluster (expected on standard ESP). Skipping."
fi
```

### 3.4 Inventory the Client Codebase

```bash
cd /path/to/repos

# 1) Kerberos auth code
grep -RInE 'kinit|krb5|UserGroupInformation|loginUserFromKeytab|KerberosPrincipal' \
  --include='*.py' --include='*.java' --include='*.scala' --include='*.sh'

# 2) Keytab references
grep -RIn 'keytab' --include='*.py' --include='*.java' --include='*.scala' \
  --include='*.sh' --include='*.properties' --include='*.yaml' --include='*.yml'

# 3) JAAS configurations
find . -name 'jaas*.conf' -o -name 'gssapi*.conf'
grep -RIn 'KafkaClient\|HiveClient\|com.sun.security.auth.module.Krb5LoginModule' \
  --include='*.conf' --include='*.properties'

# 4) AAD-DS-style login (someone@MYDOMAIN.COM)
grep -RInE '@[A-Z][A-Z0-9.-]+\.COM' --include='*.sh' --include='*.py'

# 5) Ranger API consumers (and Atlas, if you self-installed it)
grep -RInE 'ranger.*service|atlas.*api' \
  --include='*.py' --include='*.java' --include='*.scala' --include='*.json'
```

### 3.5 Compensating-Control Mapping Matrix

Build per-cluster:

| Asset | Today (ESP) | After Migration (Non-ESP) | Owner |
|-------|-------------|--------------------------|-------|
| User SSH login | AAD-domain SSH | SSH-key-only via **Azure Bastion**; AAD users have IAM role to Bastion only | Platform team |
| Cluster admin (Ambari) | AAD user via LDAP | Local Ambari user; password in **Key Vault**; Ambari behind Private Endpoint + NSG IP allowlist | Platform team |
| Hive server access | Kerberos GSSAPI | Beeline over **HTTP+TLS** (HiveServer2 binary mode disabled); cluster on private VNet; ADLS data-plane RBAC | Data team |
| Job storage auth | kinit + keytab | **User-Assigned Managed Identity** on cluster → Storage Blob Data Contributor | Platform team |
| Cross-cluster trust | AAD DS forest | Each cluster has its own MSI; cross-cluster access via VNet peering + RBAC | Platform team |
| Audit logs | Ranger audit DB | Storage diagnostic logs + Spark/Hive logs → **Log Analytics + Sentinel** | Security |
| Lineage *(if self-installed Atlas)* | Atlas | **Microsoft Purview** | Data governance |
| Column / row policies | Ranger | **Physical container separation** (HDI) OR re-platform | Data + Security |

---

## 4. Compensating Control Reference Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│  Hub VNet                                                                   │
│  ┌──────────────┐    ┌─────────────────┐    ┌──────────────────────────┐  │
│  │ Azure Bastion│    │ DNS Private Zone│    │ Azure Firewall (egress)  │  │
│  └──────────────┘    └─────────────────┘    └──────────────────────────┘  │
│         │                     │                          │                 │
│         │                     │ peer                     │ FQDN allowlist  │
│  ┌──────────────────────── Spoke VNet (HDI subnet) ──────────────────────┐│
│  │                                                                        ││
│  │   ┌─────────────────┐    ┌────────────────────────────┐               ││
│  │   │ HDI Non-ESP     │    │ Private Endpoint subnet    │               ││
│  │   │ (Spark/Hive/    │────│  • ADLS Gen2 (data)        │               ││
│  │   │  HBase/Kafka)   │    │  • Key Vault               │               ││
│  │   │  + UAMI         │    │  • SQL DB (Hive metastore) │               ││
│  │   └─────────────────┘    │  • Storage (logs)          │               ││
│  │       NSG: deny-all      └────────────────────────────┘               ││
│  │       except Bastion                                                  ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                             │
│  Diagnostic Settings → Log Analytics Workspace → Sentinel                   │
│  Defender for Storage / Defender for Key Vault                              │
│  Purview catalog scan via Managed Identity                                  │
│  Azure Policy (deny-list ESP, deny public ingress, require diag settings)   │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Network Isolation (Highest Priority)

```bash
# Create isolated VNet and subnets
az network vnet create -g $RG -n hdi-vnet --address-prefix 10.10.0.0/16
az network vnet subnet create -g $RG --vnet-name hdi-vnet -n hdi-subnet --address-prefixes 10.10.1.0/24
az network vnet subnet create -g $RG --vnet-name hdi-vnet -n pe-subnet  --address-prefixes 10.10.2.0/24 \
  --disable-private-endpoint-network-policies true
az network vnet subnet create -g $RG --vnet-name hdi-vnet -n bastion-subnet \
  --address-prefixes 10.10.3.0/26    # AzureBastionSubnet exact name required
# Note: actual Bastion subnet must be named "AzureBastionSubnet"

# NSG: allow only Bastion → cluster, allow VNet-internal Spark
az network nsg create -g $RG -n hdi-nsg
az network nsg rule create -g $RG --nsg-name hdi-nsg -n allow-bastion \
  --priority 100 --direction Inbound --protocol Tcp \
  --source-address-prefixes 10.10.3.0/26 --destination-port-ranges 22 \
  --access Allow
az network nsg rule create -g $RG --nsg-name hdi-nsg -n deny-all-internet \
  --priority 4096 --direction Inbound --protocol '*' \
  --source-address-prefixes Internet --access Deny
az network vnet subnet update -g $RG --vnet-name hdi-vnet -n hdi-subnet \
  --network-security-group hdi-nsg

# Resource Provider connection (HDI control plane needs reach-back)
# Use "Outbound and Private Link" mode (no public inbound from HDI RP)
```

### 4.2 Private Cluster + Private Endpoints

```bash
# Create non-ESP HDI Spark cluster with private cluster mode
az hdinsight create \
  -g $RG -n my-secure-cluster \
  --version 5.1 --type Spark --component-version Spark=3.5 \
  --workernode-count 4 --workernode-size Standard_D8ds_v5 \
  --headnode-size Standard_D8ds_v5 \
  --vnet-name hdi-vnet --subnet hdi-subnet \
  --resource-provider-connection Outbound \
  --enable-private-link true \
  --storage-account "$STORAGE_ACCT" \
  --storage-account-managed-identity "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$UAMI" \
  --storage-filesystem mycontainer \
  --ssh-user sshuser --ssh-public-key "$(cat ~/.ssh/id_rsa.pub)" \
  --http-user admin --http-password "$LOCAL_ADMIN_PASS" \
  --no-validation-timeout

# Create Private Endpoint for the storage account (data plane)
az network private-endpoint create -g $RG -n pe-adls-data \
  --vnet-name hdi-vnet --subnet pe-subnet \
  --private-connection-resource-id "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT" \
  --group-id dfs --connection-name adls-data-conn

# Same pattern for Key Vault, Hive Metastore Azure SQL DB, log storage
```

### 4.3 Managed Identity for Storage (Replaces Keytab)

```bash
# Create User-Assigned Managed Identity (one per cluster — least privilege)
az identity create -g $RG -n my-cluster-uami
UAMI_OBJECT_ID=$(az identity show -g $RG -n my-cluster-uami --query principalId -o tsv)

# Grant Storage Blob Data Contributor on the cluster's data container
az role assignment create \
  --assignee "$UAMI_OBJECT_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT/blobServices/default/containers/mycontainer"

# (Then pass the UAMI to az hdinsight create as shown above)
```

> **Result**: Spark/Hive/HBase processes on the cluster authenticate to ADLS via UAMI
> token — **no keytabs, no kinit, no Kerberos tickets to rotate**. Use the same UAMI
> for Key Vault data-plane access (`get` secret).

### 4.4 ADLS Gen2 Path-Level Authorization (RBAC + ABAC)

For coarser, prefix-based authorization (replacing Ranger HDFS path policies):

```bash
# Standard RBAC at container/path scope
az role assignment create \
  --assignee "$ANALYST_GROUP_OID" \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT/blobServices/default/containers/sales-bronze"

# ABAC condition (path-prefix-based; replaces /sales/* style Ranger rules)
cat > /tmp/abac-condition.json << 'EOF'
(
  (
    !(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'})
  )
  OR
  (
    @Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs:path]
      StringStartsWith 'sales/region-east/'
  )
)
EOF

az role assignment create \
  --assignee "$EAST_REGION_USER_OID" \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT" \
  --condition "$(cat /tmp/abac-condition.json)" \
  --condition-version "2.0"
```

### 4.5 Replacing Column / Row-Level Policies (Hardest Gap)

Non-ESP HDI **cannot enforce column / row policies inside Hive/HBase** the way Ranger
did. Three viable patterns:

1. **Physical separation** — one container per sensitivity tier:
   - `data-public/`   → broad RBAC
   - `data-internal/` → narrow RBAC
   - `data-pii/`      → very narrow RBAC + JIT access via PIM
   The ETL pipeline writes redacted/projected versions to lower-sensitivity containers.

2. **Per-tenant clusters** — for multi-tenant SaaS, give each tenant its own cluster
   + container + UAMI. Row filters become tenant-physical-separation.

3. **Re-platform the policy-bearing piece only** — push the SQL/analytical surface to
   **Fabric SQL endpoint** (with RLS/CLS) or **Databricks SQL Warehouse** (Unity
   Catalog ACLs). HDI continues to host pure compute (Spark batch ETL).

### 4.6 Key Vault for All Secrets

```bash
az keyvault create -g $RG -n my-kv --enable-rbac-authorization true \
  --enable-purge-protection true --retention-days 90

# Grant the cluster UAMI Key Vault Secrets User
az role assignment create \
  --assignee "$UAMI_OBJECT_ID" \
  --role "Key Vault Secrets User" \
  --scope "$(az keyvault show -n my-kv --query id -o tsv)"

# Store and retrieve secrets in Spark
az keyvault secret set --vault-name my-kv --name "my-jdbc-pass" --value "$JDBC_PASS"
```

```python
# Spark / PySpark on cluster — fetch secret via UAMI token
from azure.identity import ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient

cred = ManagedIdentityCredential(client_id="<UAMI client id>")
kv = SecretClient(vault_url="https://my-kv.vault.azure.net/", credential=cred)
jdbc_pass = kv.get_secret("my-jdbc-pass").value
```

### 4.7 SSH via Azure Bastion (No Public 22)

```bash
# Bastion subnet MUST be named exactly "AzureBastionSubnet" with /27 or larger
az network public-ip create -g $RG -n bastion-pip --sku Standard --allocation-method Static
az network bastion create -g $RG -n my-bastion --vnet-name hdi-vnet \
  --public-ip-address bastion-pip --sku Standard

# Connect via Azure Portal → Bastion → SSH (uses your AAD identity to authorize Bastion)
# Inside Bastion: SSH key (username sshuser, key created at cluster create time)
```

### 4.8 Audit Logging — Diagnostic Settings + Sentinel

```bash
# Cluster diagnostic settings (Spark/HDI logs → LAW)
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n my-law --query id -o tsv)

az monitor diagnostic-settings create \
  --name hdi-diag \
  --resource $(az hdinsight show -g $RG -n my-secure-cluster --query id -o tsv) \
  --workspace "$LAW_ID" \
  --logs '[
    {"category":"GatewayAuditLogs","enabled":true},
    {"category":"SparkApplicationsRunningEvents","enabled":true},
    {"category":"SparkApplicationsFinishedEvents","enabled":true},
    {"category":"HiveQueryLogs","enabled":true}
  ]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Storage account (data-plane audit — REPLACES Ranger audit for ADLS access)
az monitor diagnostic-settings create \
  --name adls-diag \
  --resource "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT/blobServices/default" \
  --workspace "$LAW_ID" \
  --logs '[
    {"category":"StorageRead","enabled":true},
    {"category":"StorageWrite","enabled":true},
    {"category":"StorageDelete","enabled":true}
  ]'

# Key Vault (data-plane: secret access)
az monitor diagnostic-settings create \
  --name kv-diag \
  --resource "$(az keyvault show -n my-kv --query id -o tsv)" \
  --workspace "$LAW_ID" \
  --logs '[{"category":"AuditEvent","enabled":true}]'

# Connect to Sentinel for higher-value detections
az sentinel workspace-manager assignment create \
  --workspace-name my-law -g $RG --name hdi-sentinel
```

```kusto
// Sentinel KQL: replaces Ranger "denied access" report
StorageBlobLogs
| where StatusCode in (401, 403)
| where AccountName == "$STORAGE_ACCT"
| project TimeGenerated, RequesterObjectId, OperationName, ObjectKey, StatusText, CallerIpAddress
| summarize attempts=count() by RequesterObjectId, OperationName, bin(TimeGenerated, 1h)
| order by attempts desc
```

### 4.9 Defender for Cloud Plans

```bash
az security pricing create -n StorageAccounts --tier Standard
az security pricing create -n KeyVaults       --tier Standard
az security pricing create -n Arm             --tier Standard
az security pricing create -n CloudPosture    --tier Standard
```

### 4.10 Azure Policy Guardrails

```bash
# Built-in: deny ESP from ever being re-introduced
az policy assignment create \
  --name deny-hdi-esp --display-name "Deny HDInsight with ESP" \
  --scope "/subscriptions/$SUB" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/<deny-esp-policy-id>"

# Custom: require Private Link on HDI clusters
cat > /tmp/policy.json << 'EOF'
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {"field": "type", "equals": "Microsoft.HDInsight/clusters"},
        {"field": "Microsoft.HDInsight/clusters/networkProperties.publicNetworkAccess", "notEquals": "OutboundOnly"}
      ]
    },
    "then": {"effect": "deny"}
  }
}
EOF
az policy definition create --name require-hdi-private-link --rules /tmp/policy.json --mode All
az policy assignment create --name require-hdi-private-link --scope "/subscriptions/$SUB" \
  --policy require-hdi-private-link
```

Add policies for: deny public storage account network access, require diagnostic
settings on HDI/Storage/KeyVault, require UAMI (no storage account keys), require
TLS 1.2+ on storage.

### 4.11 Microsoft Purview (Azure-Native Lineage / Catalog)

```bash
# Provision Purview account
az purview account create -g $RG -n my-purview --location eastus

# Grant Purview MI Storage Blob Data Reader on the data lake
PV_OID=$(az purview account show -g $RG -n my-purview --query identity.principalId -o tsv)
az role assignment create --assignee "$PV_OID" \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT"

# Register and scan via Purview Studio:
#   Data Map → Register → ADLS Gen2 → schedule weekly scan
#   Add HDInsight Hive metastore source for hive_table lineage
```

### 4.12 Hive Metastore on Azure SQL DB (Best Practice)

If your ESP cluster used the cluster-internal Hive Metastore, migrate to **external
Azure SQL DB** so multiple non-ESP clusters can share schema (and so cluster recreates
don't lose metadata):

```bash
# Provision Azure SQL DB with Private Endpoint (no public access)
az sql server create -g $RG -n hdi-hms-server -l eastus \
  --admin-user sqladmin --admin-password "$SQL_ADMIN_PASS" \
  --enable-public-network false
az sql db create -g $RG -s hdi-hms-server -n hivemetastore --service-objective S2

# Private Endpoint
az network private-endpoint create -g $RG -n pe-hms \
  --vnet-name hdi-vnet --subnet pe-subnet \
  --private-connection-resource-id "$(az sql server show -g $RG -n hdi-hms-server --query id -o tsv)" \
  --group-id sqlServer --connection-name hms-conn

# Pass to az hdinsight create:
#   --metastore-type hive --metastore-server-name hdi-hms-server.database.windows.net \
#   --metastore-database-name hivemetastore --metastore-user sqladmin \
#   --metastore-password "$SQL_ADMIN_PASS"
```

---

## 5. Migration Sequence

### 5.1 Phase Plan

```
T-6m  : Inventory + risk assessment + decide stay-on-HDI vs re-platform
T-5m  : Provision hub/spoke VNet, Bastion, Key Vault, Log Analytics, Purview
T-4m  : Provision Hive Metastore Azure SQL DB (PE) and migrate metadata
T-3m  : Provision FIRST non-ESP cluster (parallel to ESP); validate end-to-end
T-2m  : Migrate workloads cluster by cluster; remove Kerberos auth from clients
T-1m  : Decommission ESP clusters
2026-07-31: ESP retirement deadline — must be done
```

### 5.2 Per-Cluster Cutover Steps

1. **Snapshot Ranger policies** (and Atlas exports if Atlas was self-installed) to
   Git as compliance evidence and future re-platform input.
2. **Provision parallel non-ESP cluster** with the reference architecture above.
3. **Migrate Hive Metastore** to shared Azure SQL DB (or reuse existing if you already
   externalized it).
4. **Bulk-copy storage** (or repoint, since data lives in ADLS, not in the cluster).
5. **Update client code** — strip Kerberos:
   ```python
   # OLD (ESP)
   import subprocess
   subprocess.run(["kinit", "-kt", "/etc/security/keytabs/etl.keytab",
                   "etl@MYDOMAIN.COM"])
   # then HDFS / Hive operations
   ```
   ```python
   # NEW (non-ESP)
   # Storage auth via UAMI (no client code change needed for spark.read on ADLS via abfss://)
   # For other Azure services: use DefaultAzureCredential
   from azure.identity import DefaultAzureCredential
   cred = DefaultAzureCredential()  # picks up UAMI on the cluster automatically
   ```
6. **Reconfigure Beeline / JDBC** — drop SASL/GSSAPI, use HTTP+TLS:
   ```bash
   # OLD (ESP)
   beeline -u "jdbc:hive2://hn0-...:10001/;principal=hive/_HOST@MYDOMAIN.COM;transportMode=http;httpPath=cliservice"
   # NEW (non-ESP)
   beeline -u "jdbc:hive2://hn0-...:10001/;transportMode=http;httpPath=cliservice;ssl=true" -n admin -p "$LOCAL_PASS"
   ```
7. **Re-point pipelines** (ADF, Synapse, Spark Job Definitions) at the new cluster.
8. **Validate** using the dual-read pattern (compare query outputs between old and new
   clusters for ≥7 days).
9. **Remove read traffic from ESP cluster**, watch for regressions.
10. **Decommission ESP cluster** (delete; data persists in ADLS).

---

## 6. Validation & Acceptance

```yaml
migration_pass_criteria:
  esp_removal:
    - all_clusters_esp_disabled: true
    - aad_ds_unused_for_hdi: true
    - no_keytabs_on_clusters: true
    - no_kinit_in_client_code: true
  network_isolation:
    - hdi_clusters_private_link_only: true
    - storage_accounts_public_access: Disabled
    - keyvault_public_access: Disabled
    - sql_server_public_access: Disabled
    - no_public_ip_on_cluster: true
  identity:
    - all_clusters_use_uami_for_storage: true
    - storage_account_keys_unused_for_data_plane: true
    - kv_uses_rbac_authorization: true
  audit:
    - all_clusters_diag_settings_to_law: true
    - all_storage_diag_settings_to_law: true
    - sentinel_workspace_attached: true
    - defender_for_storage_enabled: true
    - defender_for_keyvault_enabled: true
  policy:
    - deny-esp-policy assignment: present at sub scope
    - require-private-link assignment: present at sub scope
    - require-uami assignment: present at sub scope
  governance:
    - purview_scans_running_for_adls: true
    - hive_metastore_external_azure_sql: true
    - critical_ranger_policies_mapped_to_compensating_controls: 100%
  compliance_evidence:
    - ranger_policies_archived_in_git: true
    - atlas_export_archived_if_self_installed: true
    - last_90d_ranger_audit_archived: true
```

---

## 7. Cost Re-Modeling

| ESP Cost Driver | After Migration | Net Change |
|-----------------|-----------------|------------|
| **AAD DS (~$110/mo)** | None | **-$110/mo** |
| ESP cluster surcharge | None | **savings** (varies) |
| Ranger admin DB (and Atlas storage if self-installed) | None | savings |
| HDI head/worker nodes (unchanged) | Same | $0 |
| New: Azure Bastion (~$140/mo Standard) | New | +$140/mo (shared across many resources) |
| New: Log Analytics (~$2.30/GB ingest, ~$0.10/GB retention) | New | varies — typical $50-300/mo per cluster |
| New: Purview (~$0.21/scan-hour minimum) | New | $50-200/mo for moderate scan cadence |
| New: Defender for Storage / Key Vault (~$10/mo per resource) | New | +$30-60/mo |
| New: Private Endpoints ($7.30/mo each + $0.01/GB) | New | +$50-150/mo |

**Net impact**: Roughly break-even or slight increase per workspace. Bastion + Sentinel
are typically shared across many resources, amortizing their cost. The big architectural
win is **operational simplicity** (no more AAD DS to maintain, no Kerberos ticket
issues, no Ranger policy sync drift).

---

## 8. Pitfalls & Lessons Learned

1. **Ranger column/row policies have NO direct replacement** on non-ESP HDI. If they
   are load-bearing for compliance, **physical separation** + **re-platform** the
   policy-bearing surface to Fabric/Databricks SQL is the only honest answer.

2. **AAD DS is shared infrastructure** — don't decommission it until ALL ESP clusters
   in the tenant are gone. Other services (e.g., Windows VM domain join) may also use it.

3. **Cluster admin (Ambari) goes back to local-only auth** post-ESP. Store the local
   admin password in **Key Vault**, rotate quarterly, and gate Ambari behind Bastion
   + NSG. Don't expose it on a public IP.

4. **"Deny public access" on storage breaks unprepared clients**. Make sure all
   pipelines using the storage are routed via Private Endpoint or VNet service
   endpoints BEFORE flipping `publicNetworkAccess = Disabled`.

5. **Hive Metastore migration order matters**. Migrate (or externalize) the metastore
   FIRST — re-deploying clusters against the same metastore preserves all DDL. If you
   recreate clusters with cluster-local metastore, you lose all Hive table definitions.

6. **Keytab dependencies in JARs**. Some custom Spark JARs initialize Kerberos at
   class-load time. Audit your JAR set: any `UserGroupInformation.loginUserFromKeytab`
   call must be removed (or guarded with config flags).

7. **Cluster MSI is NOT cluster-user identity**. Per-user attribution is lost — every
   job appears to ADLS as "the cluster MSI". For per-user audit, either rebuild
   attribution from Spark logs, or run **per-team clusters** so MSI ≈ team identity.

8. **ABAC on ADLS is preview / GA-recent**. Test ABAC conditions extensively before
   relying on them for compliance — wildcards, escape characters, and cross-resource
   conditions have edge cases.

9. **Bastion subnet name is rigid**. Must be exactly `AzureBastionSubnet`, /27 or
   larger. Cluster will fail to deploy if the subnet is mis-named.

10. **HDI Resource Provider connection mode**. With `--resource-provider-connection
    Outbound` (required for full private link), the cluster reaches the HDI control
    plane via outbound NAT. If your firewall/NSG drops outbound 443 to HDI RP IPs,
    the cluster will fail to provision. Allowlist HDI RP service tags.

11. **`enableHiveSupport()` and HMS connection strings**. Spark code that hardcoded
    Kerberos principal in the metastore JDBC URL must be updated to plain JDBC user/
    password (preferably read from Key Vault, not env vars).

12. **Diagnostic settings ingest cost can balloon**. If you enable **all** Spark log
    categories on a busy cluster, Log Analytics ingest can hit $1000+/mo. Tune to
    relevant categories only and use **Basic Logs** tier where appropriate.

13. **Don't forget Storm/HBase clients** if those services are on the cluster — they
    have their own Kerberos client paths to remove (`hbase-site.xml`,
    `storm-jaas.conf`).

14. **Purview is the Azure-native lineage tool, not a 1:1 Atlas replacement**. If
    you self-installed Atlas on HDI, Purview classification semantics differ from
    Atlas classifications, and lineage capture works differently. Don't expect a
    clean auto-import — plan for manual re-curation. (Reminder: Atlas is not part of
    standard HDI ESP, so most teams will be standing up Purview fresh, not migrating.)

15. **Local Ambari admin password rotation breaks running jobs**. Ambari talks to
    daemons using stored creds; rotating the admin password may require a daemon
    restart cycle. Schedule rotations in maintenance windows.

16. **PIM for JIT access to data**. For sensitive containers, use **Privileged Identity
    Management** to grant `Storage Blob Data Reader` JIT (e.g., 4-hour windows) instead
    of permanent role assignments.

17. **Service principal vs Managed Identity**. Prefer **User-Assigned Managed Identity**
    over Service Principal. UAMI has no secret to rotate, can be assigned to multiple
    resources, and survives cluster recreation.

18. **Trial-mode validation is critical**. Build the FIRST non-ESP cluster as a "pilot"
    with **all** the compensating controls enabled. Run a representative workload for
    2-4 weeks BEFORE migrating other clusters.

---

## 9. Quick-Reference Migration Checklist

```
[ ] Inventory all ESP clusters, AAD DS users/groups, Ranger policies (and Atlas entities ONLY if self-installed)
[ ] Categorize Ranger policies (table-level → RBAC; col/row-level → re-platform/separate)
[ ] Decide stay-on-HDI vs re-platform per cluster
[ ] Provision hub VNet + Bastion + Log Analytics + Sentinel + Purview
[ ] Provision spoke VNet + NSG + DNS Private Zones
[ ] Provision external Hive Metastore (Azure SQL DB + PE)
[ ] Create User-Assigned Managed Identity per cluster
[ ] Grant UAMI: Storage Blob Data Contributor on its data container, KV Secrets User
[ ] Provision FIRST non-ESP cluster with --enable-private-link, UAMI, external HMS
[ ] Configure storage account: TLS 1.2 min, public access Disabled, PE only
[ ] Configure Key Vault: RBAC auth, purge protection, public access Disabled, PE only
[ ] Enable diagnostic settings on cluster, storage, KV, SQL, vault → LAW
[ ] Enable Defender for Storage, Defender for Key Vault
[ ] Apply Azure Policies: deny-ESP, require-PE, require-UAMI, require-diag-settings
[ ] Run pilot workload 2-4 weeks; validate Sentinel detections fire correctly
[ ] Strip Kerberos from client code (kinit, keytabs, JAAS, Krb5LoginModule)
[ ] Update Beeline/JDBC URLs to drop principal=, add ssl=true
[ ] Migrate workloads cluster-by-cluster (parallel run + dual-read validation)
[ ] Archive Ranger policies + last 90d Ranger audits to Git/Blob (also Atlas exports if self-installed)
[ ] Provision Purview scans for ADLS + Hive metastore
[ ] Decommission ESP clusters
[ ] Decommission AAD DS (LAST step — ensure no other tenant resources depend on it)
[ ] Set calendar reminder: 31 July 2026 — final ESP retirement deadline
```

---

## 10. Useful Commands & Snippets

```bash
# Find all ESP-enabled HDI clusters in subscription
az hdinsight list --query "[?properties.securityProfile != null].{name:name, rg:resourceGroup, type:properties.clusterDefinition.kind}" -o table

# Show one cluster's security profile
az hdinsight show -g $RG -n <cluster> --query "properties.securityProfile" -o json

# List all role assignments scoped to the data lake (audit who can read)
az role assignment list \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT" \
  --include-inherited -o table

# Verify storage account is locked down
az storage account show -n $STORAGE_ACCT -g $RG \
  --query "{publicAccess:publicNetworkAccess, tlsMin:minimumTlsVersion, allowKey:allowSharedKeyAccess, allowBlobAnon:allowBlobPublicAccess}"

# Verify KV is locked down
az keyvault show -n my-kv \
  --query "properties.{publicAccess:publicNetworkAccess, rbac:enableRbacAuthorization, purge:enablePurgeProtection}"

# Sentinel KQL: any storage access from outside expected VNet
StorageBlobLogs
| where AccountName == "$STORAGE_ACCT"
| where CallerIpAddress !startswith "10.10."
| summarize n=count() by RequesterObjectId, CallerIpAddress
| order by n desc

# Sentinel KQL: cluster MSI activity baseline
AzureActivity
| where Caller == "$UAMI_OBJECT_ID"
| summarize count() by OperationNameValue, ResourceGroup, bin(TimeGenerated, 1d)
```

---

## Rules

1. **Treat 31 July 2026 as a hard deadline** — schedule the migration as P0 work, not background.
2. **Network-first hardening** — Private Endpoints + Bastion + NSG before anything else.
3. **One UAMI per cluster** — never reuse identities; least privilege at container scope.
4. **No keys, no keytabs** — disable storage account-key data-plane auth; remove all keytabs.
5. **Diagnostic Settings on EVERYTHING** — cluster, storage, KV, SQL, vaults — all to one LAW.
6. **Externalize the Hive Metastore** — Azure SQL DB with Private Endpoint; survives cluster recreation.
7. **Be honest about col/row policies** — if Ranger fine-grained ACLs were load-bearing, you must re-platform that surface, not pretend RBAC covers it.
8. **Azure Policy as enforcement** — `deny-ESP`, `require-private-link`, `require-UAMI`, `require-diag-settings` at subscription scope.
9. **Pilot for 2-4 weeks** before bulk-migrating clusters.
10. **Archive Ranger policies + audits** to Git/Blob as compliance evidence — they are the audit trail of what your old policies were. (Add Atlas exports only if Atlas was self-installed — it is not part of standard HDI ESP.)
11. **Decommission AAD DS LAST** — verify no other tenant workloads depend on it first.
