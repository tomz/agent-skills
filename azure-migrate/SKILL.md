---
name: azure-migrate
description: Azure Migrate — discovery, assessment, migration of servers, databases, web apps, VDI; Azure Site Recovery, Database Migration Service
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Migrate Skill

Azure Migrate is the hub for discovering, assessing, and migrating on-premises
workloads to Azure. It integrates Site Recovery, Database Migration Service, and
partner ISV tools into a single project-based experience.

---

## Architecture Overview

```
Azure Migrate Project (resource type: Microsoft.Migrate/migrateProjects)
├── Discovery & Assessment
│   ├── Azure Migrate Appliance         # Lightweight VM deployed on-prem
│   │   ├── Server Discovery            # VMware (vCenter), Hyper-V, Physical
│   │   ├── SQL Discovery               # SQL Server instances + databases
│   │   └── Web App Discovery           # ASP.NET apps on IIS
│   ├── Server Assessment               # Azure VM sizing + TCO
│   ├── SQL Assessment                  # Azure SQL DB / MI / SQL VM target
│   └── Web App Assessment              # Azure App Service target
├── Migration Tools
│   ├── Server Migration (ASR-based)    # Agentless (VMware) or Agent-based
│   ├── Database Migration Service      # Homogeneous + heterogeneous DB migrations
│   └── Web App Migration               # App Service Migration Assistant
└── Business Case                       # Full TCO/ROI analysis
```

---

## Prerequisites

```bash
# Register providers
az provider register --namespace Microsoft.Migrate
az provider register --namespace Microsoft.OffAzure
az provider register --namespace Microsoft.DataMigration

# Install CLI extension
az extension add --name azure-migrate   # limited; portal/REST used for most operations

# Create Migrate project
az group create --name "rg-migrate" --location "eastus"

# Projects are primarily portal-driven; use REST API for automation
MIGRATE_API="https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Migrate/migrateProjects"
TOKEN=$(az account get-access-token --query accessToken -o tsv)

curl -X PUT "$MIGRATE_API/migrate-proj-prod?api-version=2020-05-01" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "location": "eastus",
    "properties": { "publicNetworkAccess": "Enabled" }
  }'
```

---

## Discovery Appliance

The Azure Migrate appliance is a lightweight VM (OVA for VMware, VHD for Hyper-V,
ZIP installer for physical/AWS/GCP) that performs agentless discovery.

### VMware Appliance Deployment

```bash
# Step 1: Download OVA from Azure portal → Azure Migrate → Servers → Discover
# Step 2: Deploy OVA in vCenter (32 GB RAM, 8 vCPU, 80 GB disk recommended)
# Step 3: Configure appliance via browser at https://<appliance-ip>:44368

# Appliance requires outbound access to:
# *.portal.azure.com, *.servicebus.windows.net, *.vault.azure.net
# login.microsoftonline.com, management.azure.com

# For proxy environments:
# Set proxy in appliance config manager UI during setup

# Verify appliance registration (REST)
curl "$MIGRATE_API/migrate-proj-prod/solutions/Servers-Discovery-ServerDiscovery?api-version=2020-05-01" \
  -H "Authorization: Bearer $TOKEN" | jq '.properties.status'
```

### Physical / AWS / GCP Server Discovery

```powershell
# On each Windows server to discover — install discovery agent
# Download installer from portal, run:
.\AzureMigrateInstaller.ps1 -scenario Physical -cloud Public -proxy $null

# On Linux servers:
# wget https://raw.githubusercontent.com/Azure/azure-migrate/master/linux/AzureMigrateInstaller.sh
# chmod +x AzureMigrateInstaller.sh && sudo ./AzureMigrateInstaller.sh
```

---

## Server Assessment

### Create & Run Assessment

```bash
# Assessments are managed via REST API
ASSESS_API="https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Migrate/assessmentProjects/migrate-proj-prod"

# Create assessment group (list of discovered machines)
curl -X PUT "$ASSESS_API/groups/servers-wave1?api-version=2019-10-01" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "machines": [
        "/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Migrate/assessmentProjects/migrate-proj-prod/machines/server-01-uuid",
        "/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Migrate/assessmentProjects/migrate-proj-prod/machines/server-02-uuid"
      ]
    }
  }'

# Create VM assessment
curl -X PUT "$ASSESS_API/groups/servers-wave1/assessments/assess-wave1-eastus?api-version=2019-10-01" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "azureLocation": "EastUs",
      "azureOfferCode": "MSAZR0003P",
      "currency": "USD",
      "discountPercentage": 0,
      "percentile": "Percentile95",
      "scalingFactor": 1.3,
      "sizingCriterion": "PerformanceBased",
      "timeRange": "Month",
      "azureVmFamilies": ["Dsv3_series","Dsv4_series","Esv3_series"],
      "reservedInstance": "RI3Year",
      "azurePricingTier": "Standard",
      "vmUptime": { "daysPerMonth": 31, "hoursPerDay": 24 }
    }
  }'

# Get assessment results
curl "$ASSESS_API/groups/servers-wave1/assessments/assess-wave1-eastus?api-version=2019-10-01" \
  -H "Authorization: Bearer $TOKEN" | jq '.properties | {status, numberOfMachines, monthlyCost}'

# Get per-machine recommendations
curl "$ASSESS_API/groups/servers-wave1/assessments/assess-wave1-eastus/assessedMachines?api-version=2019-10-01" \
  -H "Authorization: Bearer $TOKEN" | jq '.value[] | {name: .properties.displayName, size: .properties.recommendedSize, suitability: .properties.suitability}'
```

### Dependency Analysis

```bash
# Agentless dependency analysis (VMware only — uses vCenter APIs)
# Enable in Azure Migrate portal → Discover → Start dependency analysis

# Agent-based dependency analysis (all sources)
# Install MMA (Log Analytics Agent) + Dependency Agent on each VM:
# Windows:
msiexec.exe /i MMASetup-AMD64.exe /qn NOAPM=1 ADD_OPINSIGHTS_WORKSPACE=1 \
  OPINSIGHTS_WORKSPACE_ID="<WorkspaceID>" OPINSIGHTS_WORKSPACE_KEY="<Key>" AcceptEndUserLicenseAgreement=1

# Linux:
wget https://raw.githubusercontent.com/Microsoft/OMS-Agent-for-Linux/master/installer/scripts/onboard_agent.sh
sh onboard_agent.sh -w <WorkspaceID> -s <Key> -d opinsights.azure.com

# View dependency map in portal → Azure Migrate → Servers → Dependency analysis
```

---

## Server Migration

### Agentless VMware Migration

```bash
# Step 1: Enable replication (agentless — uses vCenter snapshot APIs)
# Step 2: In portal: Azure Migrate → Servers → Migrate → Replicate
# Step 3: Monitor via REST

MIGRATION_API="https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.RecoveryServices"

# List replicating items
az resource list \
  --resource-type "Microsoft.RecoveryServices/vaults" \
  --resource-group "rg-migrate" -o table

# Check replication health via Site Recovery vault
az site-recovery protected-item list \
  --resource-group "rg-migrate" \
  --vault-name "migrate-vault-prod"
```

### Agent-Based Migration (Physical/AWS/GCP/Hyper-V)

```bash
# Install Mobility Service on source server
# Download from: portal → Azure Migrate → Replicate → Prepare infrastructure

# Windows (run as admin):
# MobSvcWindows\install.bat /Role MS /InstallLocation "C:\Program Files (x86)\Microsoft Azure Site Recovery" \
#   /Platform VmWare /Silent

# Linux (run as root):
# ./install -t both -a host -R Agent -l 10.0.0.100 -p 443 -s <process-server-passphrase>

# Enable replication via REST (agent-based)
curl -X PUT "https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.RecoveryServices/vaults/migrate-vault/replicationProtectedItems/server-01-uuid?api-version=2022-10-01" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "policyId": "/subscriptions/SUB_ID/.../replicationPolicies/migrate-policy",
      "providerSpecificDetails": {
        "instanceType": "InMageAzureV2",
        "targetResourceGroupId": "/subscriptions/SUB_ID/resourceGroups/rg-migrated-prod",
        "targetAzureNetworkId": "/subscriptions/SUB_ID/resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-prod",
        "targetAzureSubnetId": "subnet-app",
        "targetAzureVmName": "vm-prod-server01",
        "targetAzureVmSize": "Standard_D4s_v5",
        "targetAzureStorageAccountId": "/subscriptions/SUB_ID/.../storageAccounts/stagingsa"
      }
    }
  }'

# Monitor replication state
az site-recovery protected-item show \
  --resource-group "rg-migrate" \
  --vault-name "migrate-vault" \
  --name "server-01-uuid" \
  --fabric-name "onprem-fabric" \
  --protection-container "onprem-container" \
  --query "properties.replicationHealth"
```

### Test Failover & Final Migration

```bash
# Test migration (creates isolated test VM — doesn't impact replication)
curl -X POST "https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.RecoveryServices/vaults/migrate-vault/replicationProtectedItems/server-01-uuid/testFailover?api-version=2022-10-01" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "networkId": "/subscriptions/SUB_ID/.../virtualNetworks/vnet-test",
      "failoverDirection": "PrimaryToRecovery"
    }
  }'

# Clean up test migration
curl -X POST ".../server-01-uuid/testFailoverCleanup?api-version=2022-10-01" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "properties": { "comments": "Test passed - ready for production migration" } }'

# Initiate actual migration (final)
curl -X POST ".../server-01-uuid/plannedFailover?api-version=2022-10-01" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "properties": { "failoverDirection": "PrimaryToRecovery", "sourceSiteOperations": "NotRequired" } }'
```

---

## Database Migration

### Azure Database Migration Service (DMS)

```bash
# Create DMS instance (Premium for online migration, Standard for offline)
az dms create \
  --name "dms-prod" \
  --resource-group "rg-migrate" \
  --location "eastus" \
  --sku-name "Premium_4vCores" \
  --subnet "/subscriptions/SUB_ID/resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/subnet-dms" \
  --no-wait

# Create migration project
az dms project create \
  --service-name "dms-prod" \
  --resource-group "rg-migrate" \
  --name "proj-sql-to-azuresql" \
  --source-platform SQL \
  --target-platform SQLDB \
  --location "eastus"

# Create migration task (SQL Server → Azure SQL Database)
az dms project task create \
  --service-name "dms-prod" \
  --resource-group "rg-migrate" \
  --project-name "proj-sql-to-azuresql" \
  --name "task-migrate-adventureworks" \
  --task-type MigrateSqlServerSqlDb \
  --source-connection-json '{
    "dataSource": "onprem-sql-server.company.local",
    "authentication": "SqlAuthentication",
    "encryptConnection": true,
    "trustServerCertificate": true,
    "userName": "sa",
    "password": "P@ssw0rd!"
  }' \
  --target-connection-json '{
    "dataSource": "target-sql.database.windows.net",
    "authentication": "SqlAuthentication",
    "encryptConnection": true,
    "trustServerCertificate": false,
    "userName": "sqladmin",
    "password": "AzureP@ss1!"
  }' \
  --database-options-json '[{
    "name": "AdventureWorks",
    "targetDatabaseName": "AdventureWorks",
    "makeSourceDbReadOnly": false,
    "tableMap": {}
  }]'

# Check task status
az dms project task show \
  --service-name "dms-prod" \
  --resource-group "rg-migrate" \
  --project-name "proj-sql-to-azuresql" \
  --name "task-migrate-adventureworks" \
  --expand output

# SQL Server → Azure SQL Managed Instance (online migration)
az dms project task create \
  --service-name "dms-prod" \
  --resource-group "rg-migrate" \
  --project-name "proj-sql-to-mi" \
  --name "task-online-mi" \
  --task-type MigrateSqlServerSqlDbMiSync \
  --source-connection-json '{ "dataSource": "onprem-sql.company.local", ... }' \
  --target-connection-json '{ "dataSource": "mi-prod.xxxx.database.windows.net", ... }' \
  --database-options-json '[{
    "name": "AdventureWorks",
    "restoreWithNoRecovery": false,
    "backupFileShare": { "path": "\\\\onprem-sql\\SQLBackups", "userName": "domain\\svc-backup", "password": "pwd" }
  }]'
```

### Azure SQL Migration Extension (SSMS / ADS)

The Azure SQL Migration extension for Azure Data Studio provides guided online/offline migrations:

```bash
# Install Azure Data Studio
# In ADS → Extensions → Install "Azure SQL Migration"

# Or use az datamigration CLI for SQL MI migrations:
az extension add --name datamigration

az datamigration sql-managed-instance create \
  --resource-group "rg-migrate" \
  --managed-instance-name "mi-prod" \
  --target-db-name "AdventureWorks" \
  --migration-service "/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.DataMigration/SqlMigrationServices/dms-prod" \
  --scope "/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Sql/managedInstances/mi-prod" \
  --source-database-name "AdventureWorks" \
  --source-sql-connection '{"dataSource":"onprem-sql.company.local","authentication":"SqlAuthentication","userName":"sa","password":"P@ss!"}' \
  --backup-configuration '{"sourceLocation":{"fileShare":{"path":"\\\\onprem-sql\\SQLBackups","username":"domain\\svc","password":"pwd"}},"targetLocation":{"storageAccountResourceId":"/subscriptions/SUB_ID/resourceGroups/rg/providers/Microsoft.Storage/storageAccounts/stmigrate","accountKey":"key"}}'
```

---

## Azure Site Recovery (ASR) — DR & Migration

```bash
# Create Recovery Services Vault
az backup vault create \
  --name "rsv-dr-prod" \
  --resource-group "rg-dr" \
  --location "westus2"

# Set replication policy (RPO, app-consistent, crash-consistent)
az site-recovery policy create \
  --resource-group "rg-dr" \
  --vault-name "rsv-dr-prod" \
  --name "replication-policy-prod" \
  --recovery-point-history 72 \      # hours
  --app-consistent-frequency 60 \   # minutes between app-consistent snapshots
  --crash-consistent-frequency 5    # minutes between crash-consistent

# Failover (for DR event)
az site-recovery protected-item failover-commit \
  --resource-group "rg-dr" \
  --vault-name "rsv-dr-prod" \
  --name "server-01-uuid" \
  --fabric-name "primary-fabric" \
  --protection-container "primary-container"

# Failback (return to on-prem after DR)
az site-recovery protected-item reprotect \
  --resource-group "rg-dr" \
  --vault-name "rsv-dr-prod" \
  --name "server-01-uuid" \
  --fabric-name "primary-fabric" \
  --protection-container "primary-container"
```

---

## Web App Migration

```bash
# Use App Service Migration Assistant for ASP.NET discovery + migration
# Download from: https://azure.microsoft.com/en-us/products/app-service/migration-tools/

# CLI-assisted migration — create target App Service
az appservice plan create \
  --name "asp-migrated-prod" \
  --resource-group "rg-migrate-apps" \
  --location "eastus" \
  --sku P2V3 \
  --is-linux false

az webapp create \
  --name "myapp-migrated" \
  --resource-group "rg-migrate-apps" \
  --plan "asp-migrated-prod"

# Deploy migrated package
az webapp deploy \
  --name "myapp-migrated" \
  --resource-group "rg-migrate-apps" \
  --src-path "./migrated-app.zip" \
  --type zip
```

---

## Business Case & TCO

```bash
# Business case is generated in Azure Migrate portal
# REST API to retrieve business case report:
curl "https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Migrate/assessmentProjects/migrate-proj-prod/businessCases/businesscase1?api-version=2023-03-15" \
  -H "Authorization: Bearer $TOKEN" | jq '.properties | {status, onPremisesAnnualCost, azureAnnualCost}'

# TCO Calculator (web-based, export to CSV)
# https://azure.microsoft.com/en-us/pricing/tco/calculator/
```

---

## Migration Wave Planning (Shell Script)

```bash
#!/bin/bash
# Export assessed servers to CSV for wave planning
ASSESS_API="https://management.azure.com/subscriptions/SUB_ID/resourceGroups/rg-migrate/providers/Microsoft.Migrate/assessmentProjects/migrate-proj-prod"
TOKEN=$(az account get-access-token --query accessToken -o tsv)

# Get all assessed machines with recommended sizes and monthly cost
curl -s "$ASSESS_API/groups/all-servers/assessments/assess-prod/assessedMachines?api-version=2019-10-01" \
  -H "Authorization: Bearer $TOKEN" | \
  jq -r '.value[] | [
    .properties.displayName,
    .properties.recommendedSize,
    .properties.suitability,
    (.properties.monthlyComputeCostEstimate | tostring),
    (.properties.monthlyStorageCostEstimate | tostring),
    .properties.operatingSystem
  ] | @csv' > assessed-servers.csv

echo "name,recommended_size,suitability,monthly_compute,monthly_storage,os" | \
  cat - assessed-servers.csv > migration-plan.csv
echo "Exported $(wc -l < migration-plan.csv) servers to migration-plan.csv"
```

---

## Quick Reference

| Task | Tool |
|------|------|
| Discover VMware VMs | Azure Migrate OVA appliance + vCenter credentials |
| Discover physical/AWS/GCP | Azure Migrate installer script |
| Assess VMs | Azure Migrate portal → Assessment → Create |
| Dependency mapping | Agentless (VMware) or MMA+DependencyAgent |
| Migrate VMware VMs (agentless) | Azure Migrate → Migrate → Replicate |
| Migrate physical servers | Agent-based: Mobility Service + ASR |
| Migrate SQL databases | `az dms` or Azure SQL Migration extension |
| Migrate web apps | App Service Migration Assistant |
| DR with ASR | Recovery Services Vault → Site Recovery |
| Business case | Azure Migrate portal → Business case |
| Check replication | `az site-recovery protected-item list` |
