---
name: azure-backup
description: Azure Backup — Recovery Services vault, Backup vault, VM backup, SQL/SAP HANA backup, MARS agent, Azure Files backup, disk backup, blob backup, policies, instant restore, cross-region restore
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Backup — Comprehensive Reference

## Overview

Azure Backup is a cloud-native backup service that protects on-premises and Azure workloads.
It stores backups in two vault types and supports a wide range of data sources.

Key capabilities:
- Zero-infrastructure, serverless backup management
- Application-consistent backups for VMs, SQL, SAP HANA
- Built-in security: soft delete, immutability, MUA, encryption at rest
- Backup center for unified monitoring across subscriptions and vaults

---

## 1. Vault Types

### Recovery Services Vault (RSV)
- **Legacy primary vault** — supports all classic workloads
- Supported datasources: Azure VMs, SQL in VM, SAP HANA in VM, Azure Files, MARS agent, MABS/DPM, AFS
- Supports geo-redundant (GRS), locally redundant (LRS), zone-redundant (ZRS) storage
- Required for cross-region restore (CRR)

```bash
az backup vault create \
  --name myRSV \
  --resource-group myRG \
  --location eastus \
  --storage-redundancy GeoRedundant

# Enable cross-region restore
az backup vault backup-properties set \
  --name myRSV \
  --resource-group myRG \
  --cross-region-restore-flag true
```

### Backup Vault (BV) — New
- **Newer vault type** introduced for modern datasources
- Supported datasources: Azure Disk, Azure Blob (vaulted), Azure Database for PostgreSQL, AKS
- Uses Azure Resource Manager (ARM) — better RBAC, locks, Bicep/Terraform support
- Does NOT support Azure VMs or MARS agent

```bash
az dataprotection backup-vault create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --location eastus \
  --storage-settings datastore-type="VaultStore" type="GeoRedundant"
```

### RSV vs BV Comparison

| Feature                  | Recovery Services Vault | Backup Vault       |
|--------------------------|-------------------------|--------------------|
| Azure VMs                | Yes                     | No                 |
| SQL in VM                | Yes                     | No                 |
| SAP HANA in VM           | Yes                     | No                 |
| Azure Files              | Yes                     | No                 |
| MARS / MABS              | Yes                     | No                 |
| Azure Disk               | No                      | Yes                |
| Azure Blob (vaulted)     | No                      | Yes                |
| PostgreSQL (Azure DB)    | No                      | Yes                |
| AKS backup               | No                      | Yes                |
| Cross-region restore     | Yes (GRS vaults)        | No (yet)           |
| Immutable vault          | Yes                     | Yes                |
| Soft delete              | Yes                     | Yes                |

---

## 2. Backup Policies

### Standard vs Enhanced Policy (VM)

| Feature                  | Standard Policy     | Enhanced Policy          |
|--------------------------|---------------------|--------------------------|
| Snapshot tier            | Instant RP (1–5 d)  | Operational tier (1–30 d)|
| Vault tier               | Yes                 | Yes                      |
| Backup frequency         | Daily               | Hourly (1,2,4,6,8,12 h)  |
| Trusted Azure VM         | Not required        | Required for hourly       |
| Archive tier support     | Yes                 | Yes                      |

### Creating a Custom VM Backup Policy (JSON)

```json
{
  "name": "MyEnhancedPolicy",
  "properties": {
    "backupManagementType": "AzureIaasVM",
    "schedulePolicy": {
      "schedulePolicyType": "SimpleSchedulePolicyV2",
      "scheduleRunFrequency": "Hourly",
      "hourlySchedule": {
        "interval": 4,
        "scheduleWindowStartTime": "2024-01-01T06:00:00Z",
        "scheduleWindowDuration": 16
      }
    },
    "retentionPolicy": {
      "retentionPolicyType": "LongTermRetentionPolicy",
      "dailySchedule": {
        "retentionTimes": ["2024-01-01T06:00:00Z"],
        "retentionDuration": { "count": 30, "durationType": "Days" }
      },
      "weeklySchedule": {
        "daysOfTheWeek": ["Sunday"],
        "retentionTimes": ["2024-01-01T06:00:00Z"],
        "retentionDuration": { "count": 12, "durationType": "Weeks" }
      },
      "monthlySchedule": {
        "retentionScheduleFormatType": "Weekly",
        "retentionScheduleWeekly": {
          "daysOfTheWeek": ["Sunday"],
          "weeksOfTheMonth": ["First"]
        },
        "retentionTimes": ["2024-01-01T06:00:00Z"],
        "retentionDuration": { "count": 24, "durationType": "Months" }
      },
      "yearlySchedule": {
        "retentionScheduleFormatType": "Weekly",
        "monthsOfYear": ["January"],
        "retentionScheduleWeekly": {
          "daysOfTheWeek": ["Sunday"],
          "weeksOfTheMonth": ["First"]
        },
        "retentionTimes": ["2024-01-01T06:00:00Z"],
        "retentionDuration": { "count": 7, "durationType": "Years" }
      }
    },
    "instantRpRetentionRangeInDays": 5,
    "tieringPolicy": {
      "ArchivedRP": {
        "tieringMode": "TierAfter",
        "duration": 3,
        "durationType": "Months"
      }
    }
  }
}
```

```bash
# Apply policy from JSON
az backup policy create \
  --vault-name myRSV \
  --resource-group myRG \
  --name MyEnhancedPolicy \
  --backup-management-type AzureIaasVM \
  --policy @policy.json
```

---

## 3. Azure VM Backup

### Enable VM Backup

```bash
# Enable backup for a VM
az backup protection enable-for-vm \
  --vault-name myRSV \
  --resource-group myRG \
  --vm myVM \
  --policy-name DefaultPolicy

# Trigger on-demand backup
az backup protection backup-now \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --backup-management-type AzureIaasVM \
  --retain-until 30-09-2025

# List backup jobs
az backup job list \
  --vault-name myRSV \
  --resource-group myRG \
  --output table

# Monitor a specific job
az backup job show \
  --vault-name myRSV \
  --resource-group myRG \
  --name <job-id>
```

### Instant Restore
Snapshots stored in the customer's resource group (not in vault) for 1–5 days (Standard)
or 1–30 days (Enhanced policy). Enables fast restore without waiting for vault tier retrieval.

```bash
# List restore points (recovery points)
az backup recoverypoint list \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --backup-management-type AzureIaasVM \
  --output table
```

### VM Restore Options

```bash
# Restore to new VM
az backup restore restore-disks \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --rp-name <recovery-point-name> \
  --storage-account myStorageAccount \
  --target-resource-group myTargetRG

# Restore to original location (replace)
az backup restore restore-disks \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --rp-name <recovery-point-name> \
  --storage-account myStorageAccount \
  --restore-to-staging-storage-account false
```

### Cross-Region Restore (CRR)

```bash
# List recovery points in secondary region
az backup recoverypoint list \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --backup-management-type AzureIaasVM \
  --use-secondary-region \
  --output table

# Restore from secondary region
az backup restore restore-disks \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --rp-name <recovery-point-name> \
  --storage-account myStorageAccount \
  --target-resource-group myTargetRG \
  --use-secondary-region
```

### Selective Disk Backup

```bash
# Backup only OS disk (exclude data disks)
az backup protection enable-for-vm \
  --vault-name myRSV \
  --resource-group myRG \
  --vm myVM \
  --policy-name DefaultPolicy \
  --disk-list-setting include \
  --diskslist 0
```

---

## 4. Azure Files Backup (Snapshot-Based)

Azure Files backup uses share-level snapshots stored in the storage account.

```bash
# Enable backup for an Azure file share
az backup protection enable-for-azurefileshare \
  --vault-name myRSV \
  --resource-group myRG \
  --storage-account myStorageAccount \
  --azure-file-share myFileShare \
  --policy-name AzureFileSharePolicy

# Trigger on-demand backup
az backup protection backup-now \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name "StorageContainer;Storage;myRG;myStorageAccount" \
  --item-name "AzureFileShare;myFileShare" \
  --backup-management-type AzureStorage \
  --retain-until 30-09-2025

# Item-level recovery — restore individual file
az backup restore restore-azurefiles \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name "StorageContainer;Storage;myRG;myStorageAccount" \
  --item-name "AzureFileShare;myFileShare" \
  --rp-name <recovery-point-name> \
  --resolve-conflict Overwrite \
  --restore-type ItemLevelRestore \
  --source-file-type File \
  --source-file-path "folder1/file.txt" \
  --target-storage-account myTargetStorageAccount \
  --target-file-share myTargetShare \
  --target-folder restoredFiles
```

---

## 5. SQL Server in Azure VM Backup

### Enable Auto-Protection and Backup

```bash
# Register SQL VM with vault
az backup container register \
  --vault-name myRSV \
  --resource-group myRG \
  --workload-type SQLDataBase \
  --backup-management-type AzureWorkload \
  --resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/mySQLVM

# List discoverable SQL databases
az backup protectable-item list \
  --vault-name myRSV \
  --resource-group myRG \
  --workload-type SQLDataBase \
  --output table

# Enable auto-protection on SQL instance (protects all existing + future DBs)
az backup protection auto-enable-for-azurewl \
  --vault-name myRSV \
  --resource-group myRG \
  --policy-name SQLPolicy \
  --protectable-item-name "SQLInstance;MSSQLServer" \
  --protectable-item-type SQLInstance \
  --server-name mySQLVM \
  --workload-type SQLDataBase

# Enable backup for specific database
az backup protection enable-for-azurewl \
  --vault-name myRSV \
  --resource-group myRG \
  --policy-name SQLPolicy \
  --protectable-item-name "SQLDataBase;MSSQLSERVER;AdventureWorks" \
  --protectable-item-type SQLDataBase \
  --server-name mySQLVM \
  --workload-type SQLDataBase

# Trigger full backup on-demand
az backup protection backup-now \
  --vault-name myRSV \
  --resource-group myRG \
  --item-name "SQLDataBase;MSSQLSERVER;AdventureWorks" \
  --container-name "VMAppContainer;Compute;myRG;mySQLVM" \
  --backup-management-type AzureWorkload \
  --backup-type Full \
  --retain-until 31-12-2025

# Restore SQL DB to point-in-time
az backup restore restore-azurewl \
  --vault-name myRSV \
  --resource-group myRG \
  --restore-config @sql-restore-config.json
```

### SQL Backup Policy JSON

```json
{
  "backupManagementType": "AzureWorkload",
  "workLoadType": "SQLDataBase",
  "settings": { "isCompression": true, "issqlcompression": true, "timeZone": "UTC" },
  "subProtectionPolicy": [
    {
      "policyType": "Full",
      "schedulePolicy": {
        "schedulePolicyType": "SimpleSchedulePolicy",
        "scheduleRunFrequency": "Weekly",
        "scheduleRunDays": ["Sunday"],
        "scheduleRunTimes": ["2024-01-01T22:00:00Z"]
      },
      "retentionPolicy": {
        "retentionPolicyType": "LongTermRetentionPolicy",
        "weeklySchedule": {
          "daysOfTheWeek": ["Sunday"],
          "retentionTimes": ["2024-01-01T22:00:00Z"],
          "retentionDuration": { "count": 104, "durationType": "Weeks" }
        }
      }
    },
    {
      "policyType": "Differential",
      "schedulePolicy": {
        "schedulePolicyType": "SimpleSchedulePolicy",
        "scheduleRunFrequency": "Weekly",
        "scheduleRunDays": ["Wednesday"],
        "scheduleRunTimes": ["2024-01-01T22:00:00Z"]
      },
      "retentionPolicy": {
        "retentionPolicyType": "SimpleRetentionPolicy",
        "retentionDuration": { "count": 30, "durationType": "Days" }
      }
    },
    {
      "policyType": "Log",
      "schedulePolicy": {
        "schedulePolicyType": "LogSchedulePolicy",
        "scheduleFrequencyInMins": 60
      },
      "retentionPolicy": {
        "retentionPolicyType": "SimpleRetentionPolicy",
        "retentionDuration": { "count": 15, "durationType": "Days" }
      }
    }
  ]
}
```

---

## 6. SAP HANA Backup

```bash
# Register SAP HANA container
az backup container register \
  --vault-name myRSV \
  --resource-group myRG \
  --workload-type SAPHanaDatabase \
  --backup-management-type AzureWorkload \
  --resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myHANAVM

# Run pre-registration script on HANA VM first (installs plugin, sets permissions)
# Download from: aka.ms/scriptforpermsonhana

# List HANA databases
az backup protectable-item list \
  --vault-name myRSV \
  --resource-group myRG \
  --workload-type SAPHanaDatabase \
  --output table

# Enable HANA backup
az backup protection enable-for-azurewl \
  --vault-name myRSV \
  --resource-group myRG \
  --policy-name HANAPolicy \
  --protectable-item-name "SAPHanaDatabase;HDB;SYSTEMDB" \
  --protectable-item-type SAPHanaDatabase \
  --server-name myHANAVM \
  --workload-type SAPHanaDatabase

# Trigger on-demand full backup
az backup protection backup-now \
  --vault-name myRSV \
  --resource-group myRG \
  --item-name "SAPHanaDatabase;HDB;SYSTEMDB" \
  --container-name "VMAppContainer;Compute;myRG;myHANAVM" \
  --backup-management-type AzureWorkload \
  --backup-type Full
```

---

## 7. Azure Disk Backup

Uses Backup vault. Incremental snapshots stored in a snapshot resource group.

```bash
# Create a backup policy for disk
az dataprotection backup-policy create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --name DiskBackupPolicy \
  --policy @disk-policy.json

# Enable disk backup
az dataprotection backup-instance create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance @disk-backup-instance.json

# Trigger on-demand backup
az dataprotection backup-instance adhoc-backup \
  --vault-name myBackupVault \
  --resource-group myRG \
  --name myDiskInstance \
  --rule-name BackupHourly

# List restore points
az dataprotection recovery-point list \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance-name myDiskInstance \
  --output table

# Restore disk
az dataprotection backup-instance restore trigger \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance-name myDiskInstance \
  --restore-request-object @disk-restore-request.json
```

### Disk Backup Policy JSON

```json
{
  "datasourceTypes": ["Microsoft.Compute/disks"],
  "name": "DiskBackupPolicy",
  "objectType": "BackupPolicy",
  "policyRules": [
    {
      "backupParameters": { "backupType": "Incremental", "objectType": "AzureBackupParams" },
      "dataStore": { "dataStoreType": "OperationalStore", "objectType": "DataStoreInfoBase" },
      "name": "BackupHourly",
      "objectType": "AzureBackupRule",
      "trigger": {
        "objectType": "ScheduleBasedTriggerContext",
        "schedule": {
          "repeatingTimeIntervals": ["R/2024-01-01T06:00:00+00:00/PT4H"],
          "timeZone": "UTC"
        },
        "taggingCriteria": [{ "isDefault": true, "tagInfo": { "id": "Default_", "tagName": "Default" }, "taggingPriority": 99 }]
      }
    },
    {
      "isDefault": true,
      "lifecycles": [{ "deleteAfter": { "duration": "P7D", "objectType": "AbsoluteDeleteOption" }, "sourceDataStore": { "dataStoreType": "OperationalStore", "objectType": "DataStoreInfoBase" } }],
      "name": "Default",
      "objectType": "AzureRetentionRule"
    }
  ]
}
```

---

## 8. Azure Blob Backup

### Operational Backup (within storage account — no vault needed for snapshots)
Point-in-time restore using continuous journaling. Retained in same storage account.

```bash
# Enable operational backup (uses Recovery Services Vault)
az backup protection enable-for-azureblob \
  --vault-name myRSV \
  --resource-group myRG \
  --storage-account myStorageAccount \
  --policy-name BlobBackupPolicy
```

### Vaulted Blob Backup (Backup Vault — copies to vault storage)

```bash
# Create blob backup instance in Backup Vault
az dataprotection backup-instance create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance @blob-backup-instance.json

# Restore blobs to point-in-time
az dataprotection backup-instance restore trigger \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance-name myBlobInstance \
  --restore-request-object @blob-restore-request.json
```

---

## 9. MARS Agent (On-Premises Files/Folders)

Microsoft Azure Recovery Services (MARS) agent backs up files, folders, and system state
from Windows on-premises machines or Azure VMs directly to a Recovery Services vault.

### Setup Steps

```powershell
# Download and install MARS agent from Azure portal or:
# https://aka.ms/azurebackup_agent

# Register with vault using vault credentials (.VaultCredentials file downloaded from portal)
# Done via MARS agent UI or:
Start-OBRegistration -VaultCredentials "C:\Vault\myVault.VaultCredentials" -Confirm:$false

# Create backup schedule
$policy = New-OBPolicy
$schedule = New-OBSchedule -DaysOfWeek Sunday,Wednesday -TimesOfDay 22:00
Set-OBSchedule -Policy $policy -Schedule $schedule

# Set retention
$retention = New-OBRetentionPolicy -RetentionDays 30
Set-OBRetentionPolicy -Policy $policy -RetentionPolicy $retention

# Add paths to backup
$spec = New-OBFileSpec -FileSpec "C:\Data"
Add-OBFileSpec -Policy $policy -FileSpec $spec

# Set policy and trigger backup
Set-OBPolicy -Policy $policy
Start-OBBackup -Policy $policy
```

```bash
# Monitor MARS backups from CLI (on Azure side)
az backup item list \
  --vault-name myRSV \
  --resource-group myRG \
  --backup-management-type MAB \
  --output table
```

---

## 10. Azure Backup Server (MABS / DPM)

MABS (Microsoft Azure Backup Server) is a free server application for protecting workloads:
Hyper-V VMs, VMware VMs, SQL, SharePoint, Exchange, system state.

- Based on DPM (Data Protection Manager) engine
- Stores short-term backups locally (disk), then moves to Azure vault
- Supports application-consistent backup without agents on each VM

```bash
# Register MABS/DPM server with vault (done from MABS console)
# Monitor from az CLI:
az backup container list \
  --vault-name myRSV \
  --resource-group myRG \
  --backup-management-type AzureBackupServer \
  --output table
```

---

## 11. Azure Database for PostgreSQL Backup

Uses Backup vault. Long-term retention up to 10 years.

```bash
# Create PostgreSQL backup policy
az dataprotection backup-policy create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --name PGFlexPolicy \
  --policy @pg-backup-policy.json

# Enable backup
az dataprotection backup-instance create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance @pg-backup-instance.json
```

---

## 12. AKS Backup

Backs up Kubernetes workloads (cluster resources + persistent volumes) using Backup vault.
Requires backup extension installed in AKS cluster.

```bash
# Install backup extension in AKS
az k8s-extension create \
  --name azure-aks-backup \
  --extension-type microsoft.dataprotection.kubernetes \
  --scope cluster \
  --cluster-type managedClusters \
  --cluster-name myAKSCluster \
  --resource-group myRG \
  --release-train stable \
  --configuration-settings \
    blobContainer=myBackupContainer \
    storageAccount=myStorageAccount \
    storageAccountResourceGroup=myRG \
    storageAccountSubscriptionId=<sub-id>

# Create AKS backup instance
az dataprotection backup-instance create \
  --vault-name myBackupVault \
  --resource-group myRG \
  --backup-instance @aks-backup-instance.json

# Trigger on-demand AKS backup
az dataprotection backup-instance adhoc-backup \
  --vault-name myBackupVault \
  --resource-group myRG \
  --name myAKSInstance \
  --rule-name BackupDaily
```

---

## 13. Security Features

### Soft Delete

Protects deleted backup data from permanent loss for 14 days (VMs) by default.

```bash
# Enable soft delete (on by default, but can be disabled)
az backup vault backup-properties set \
  --name myRSV \
  --resource-group myRG \
  --soft-delete-feature-state Enable

# List soft-deleted items
az backup item list \
  --vault-name myRSV \
  --resource-group myRG \
  --backup-management-type AzureIaasVM \
  --output table

# Undelete a soft-deleted item
az backup protection undelete \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --backup-management-type AzureIaasVM \
  --workload-type VM
```

### Immutable Vault

Prevents changes to backup policies that would reduce retention. Two modes:
- **Unlocked immutability** — can be disabled later
- **Locked immutability** — cannot be disabled (irreversible)

```bash
# Enable unlocked immutability
az backup vault update \
  --name myRSV \
  --resource-group myRG \
  --immutability-state Unlocked

# Lock immutability (IRREVERSIBLE)
az backup vault update \
  --name myRSV \
  --resource-group myRG \
  --immutability-state Locked
```

### Multi-User Authorization (MUA)

Requires a second authorized user (Resource Guard owner) to approve critical backup operations.

```bash
# Create Resource Guard (in a different subscription/tenant for security)
az dataprotection resource-guard create \
  --resource-group guardRG \
  --resource-guard-name myResourceGuard \
  --location eastus

# Associate Resource Guard with vault
az backup vault resource-guard-mapping create \
  --vault-name myRSV \
  --resource-group myRG \
  --resource-guard-id /subscriptions/<guard-sub>/resourceGroups/guardRG/providers/Microsoft.DataProtection/resourceGuards/myResourceGuard
```

### Encryption

```bash
# Enable CMK (Customer-Managed Key) encryption on vault
az backup vault encryption update \
  --vault-name myRSV \
  --resource-group myRG \
  --encryption-key-id https://myKeyVault.vault.azure.net/keys/myKey/version \
  --mi-system-assigned \
  --infrastructure-encryption-state Enable
```

### Private Endpoints

```bash
# Create private endpoint for vault
az network private-endpoint create \
  --name myVaultPE \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.RecoveryServices/vaults/myRSV \
  --group-id AzureBackup \
  --connection-name myVaultConnection \
  --location eastus
```

### RBAC Roles

| Role | Description |
|------|-------------|
| Backup Contributor | Full backup management (no vault deletion) |
| Backup Operator | Enable/disable protection, trigger backup/restore |
| Backup Reader | Read-only view of all backup information |
| Backup Restore Operator | Trigger restore operations |

```bash
# Assign Backup Operator role
az role assignment create \
  --assignee user@domain.com \
  --role "Backup Operator" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.RecoveryServices/vaults/myRSV
```

---

## 14. Monitoring

### Backup Center

Unified monitoring across all vaults and subscriptions.

```bash
# List all backup instances across subscriptions
az dataprotection backup-instance list-from-resourcegraph \
  --subscriptions <sub1> <sub2> \
  --datasource-type AzureVM \
  --output table

# Check protection health
az backup job list \
  --vault-name myRSV \
  --resource-group myRG \
  --status Failed \
  --output table
```

### Backup Reports (Log Analytics)

```bash
# Enable diagnostic settings for vault to Log Analytics
az monitor diagnostic-settings create \
  --name BackupDiagnostics \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.RecoveryServices/vaults/myRSV \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace \
  --logs '[{"category":"AzureBackupReport","enabled":true},{"category":"CoreAzureBackup","enabled":true},{"category":"AddonAzureBackupJobs","enabled":true},{"category":"AddonAzureBackupAlerts","enabled":true},{"category":"AddonAzureBackupPolicy","enabled":true},{"category":"AddonAzureBackupStorage","enabled":true},{"category":"AddonAzureBackupProtectedInstance","enabled":true}]'
```

### KQL Query Examples (Log Analytics)

```kusto
// Failed backup jobs in last 7 days
AddonAzureBackupJobs
| where TimeGenerated > ago(7d)
| where JobStatus == "Failed"
| project TimeGenerated, BackupItemUniqueId, JobOperation, JobFailureCode
| order by TimeGenerated desc

// Backup storage usage per vault
CoreAzureBackup
| where TimeGenerated > ago(1d)
| where OperationName == "BackupItem"
| summarize TotalStorageGB = sum(BackupItemFrontEndSize) / 1073741824 by VaultUniqueId
```

### Azure Monitor Alerts

```bash
# Create alert for backup job failures
az monitor alert create \
  --name BackupFailureAlert \
  --resource-group myRG \
  --target /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.RecoveryServices/vaults/myRSV \
  --condition "count backup_health_event > 0" \
  --action-group /subscriptions/<sub>/resourceGroups/myRG/providers/microsoft.insights/actionGroups/myActionGroup
```

---

## 15. Infrastructure as Code

### Bicep — Recovery Services Vault + VM Backup Policy

```bicep
param location string = resourceGroup().location
param vaultName string = 'myRSV'

resource recoveryVault 'Microsoft.RecoveryServices/vaults@2023-04-01' = {
  name: vaultName
  location: location
  sku: { name: 'RS0', tier: 'Standard' }
  properties: {
    publicNetworkAccess: 'Disabled'
    securitySettings: {
      softDeleteSettings: {
        softDeleteState: 'AlwaysON'
        softDeleteRetentionPeriodInDays: 14
      }
      immutabilitySettings: {
        state: 'Unlocked'
      }
    }
  }
}

resource backupPolicy 'Microsoft.RecoveryServices/vaults/backupPolicies@2023-04-01' = {
  name: 'EnhancedVMPolicy'
  parent: recoveryVault
  properties: {
    backupManagementType: 'AzureIaasVM'
    policyType: 'V2'
    instantRpRetentionRangeInDays: 5
    schedulePolicy: {
      schedulePolicyType: 'SimpleSchedulePolicyV2'
      scheduleRunFrequency: 'Hourly'
      hourlySchedule: {
        interval: 4
        scheduleWindowStartTime: '2024-01-01T06:00:00Z'
        scheduleWindowDuration: 16
      }
    }
    retentionPolicy: {
      retentionPolicyType: 'LongTermRetentionPolicy'
      dailySchedule: {
        retentionTimes: ['2024-01-01T06:00:00Z']
        retentionDuration: { count: 30, durationType: 'Days' }
      }
      weeklySchedule: {
        daysOfTheWeek: ['Sunday']
        retentionTimes: ['2024-01-01T06:00:00Z']
        retentionDuration: { count: 12, durationType: 'Weeks' }
      }
      monthlySchedule: {
        retentionScheduleFormatType: 'Weekly'
        retentionScheduleWeekly: { daysOfTheWeek: ['Sunday'], weeksOfTheMonth: ['First'] }
        retentionTimes: ['2024-01-01T06:00:00Z']
        retentionDuration: { count: 24, durationType: 'Months' }
      }
      yearlySchedule: {
        retentionScheduleFormatType: 'Weekly'
        monthsOfYear: ['January']
        retentionScheduleWeekly: { daysOfTheWeek: ['Sunday'], weeksOfTheMonth: ['First'] }
        retentionTimes: ['2024-01-01T06:00:00Z']
        retentionDuration: { count: 7, durationType: 'Years' }
      }
    }
  }
}
```

### Terraform — Backup Vault + Disk Backup

```hcl
resource "azurerm_data_protection_backup_vault" "bv" {
  name                = "myBackupVault"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  datastore_type      = "VaultStore"
  redundancy          = "GeoRedundant"

  identity { type = "SystemAssigned" }

  soft_delete       = "On"
  retention_duration_in_days = 14
}

resource "azurerm_data_protection_backup_policy_disk" "disk_policy" {
  name     = "DiskBackupPolicy"
  vault_id = azurerm_data_protection_backup_vault.bv.id

  backup_repeating_time_intervals = ["R/2024-01-01T06:00:00+00:00/PT4H"]
  default_retention_duration      = "P7D"
  time_zone                       = "UTC"

  retention_rule {
    name     = "Weekly"
    duration = "P4W"
    priority = 25
    criteria {
      absolute_criteria = "FirstOfWeek"
    }
  }
}

resource "azurerm_data_protection_backup_instance_disk" "disk_instance" {
  name                         = "myDiskBackupInstance"
  vault_id                     = azurerm_data_protection_backup_vault.bv.id
  location                     = azurerm_resource_group.rg.location
  disk_id                      = azurerm_managed_disk.disk.id
  snapshot_resource_group_name = azurerm_resource_group.snapshot_rg.name
  backup_policy_id             = azurerm_data_protection_backup_policy_disk.disk_policy.id
}
```

---

## 16. Cost Optimization

### Archive Tier

Move recovery points older than 3 months to archive tier (~75% cost reduction vs vault-standard).
Supported for: Azure VMs, SQL in VM, SAP HANA in VM.

```bash
# Move eligible recovery points to archive tier
az backup recoverypoint move \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --backup-management-type AzureIaasVM \
  --rp-name <recovery-point-name> \
  --source-tier VaultStandard \
  --destination-tier VaultArchive

# List recovery points by tier
az backup recoverypoint list \
  --vault-name myRSV \
  --resource-group myRG \
  --container-name myVM \
  --item-name myVM \
  --backup-management-type AzureIaasVM \
  --tier VaultStandard \
  --output table
```

### Storage Redundancy

| Redundancy | Copies | Cost | Use Case |
|------------|--------|------|----------|
| LRS | 3 (same DC) | Lowest | Dev/test, non-critical |
| ZRS | 3 (different zones) | Medium | Zone-redundant HA |
| GRS | 6 (paired region) | Higher | CRR, production |

### Reserved Capacity

Azure Backup reserved capacity: pre-purchase storage in 1-year or 3-year terms for ~20% savings.

---

## 17. Disaster Recovery Patterns

### Pattern 1: Cross-Region Restore (Active-Passive)

```
Primary Region (East US)               Secondary Region (West US)
┌─────────────────────┐                ┌─────────────────────────┐
│  VM / SQL / HANA    │                │  Restored VM (on demand) │
│         │           │                │                          │
│    RSV (GRS)  ──── CRR ──────────▶  │  RSV (secondary copy)   │
└─────────────────────┘                └─────────────────────────┘
```

- Enable GRS on RSV + enable CRR feature
- RTO: hours (restore from snapshot + boot)
- RPO: up to 24h (last vault snapshot)

### Pattern 2: Snapshot-Based Low RTO

- Enhanced policy with hourly backups + 5-day instant restore
- Restore from local snapshot (no vault retrieval) = minutes
- Best for VMs requiring <2h RTO

### Pattern 3: On-Premises to Azure (MARS + MABS)

```
On-Premises                            Azure
┌──────────────────┐                   ┌──────────────────┐
│  File Servers    │── MARS Agent ──▶  │  Recovery        │
│  Hyper-V VMs     │── MABS      ──▶  │  Services Vault  │
│  SQL Server      │                   └──────────────────┘
└──────────────────┘
```

### Pattern 4: Multi-Vault Immutable Strategy

```bash
# Production vault — immutable locked
az backup vault update --name prodRSV --resource-group myRG --immutability-state Locked

# DR vault — MUA protected, GRS
az backup vault update --name drRSV --resource-group drRG --immutability-state Unlocked
```

---

## 18. Quick Reference

```bash
# Show vault details
az backup vault show --name myRSV --resource-group myRG

# List all protected items
az backup item list --vault-name myRSV --resource-group myRG --output table

# List recent jobs (last 24h)
az backup job list --vault-name myRSV --resource-group myRG \
  --start-date $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) --output table

# Stop backup protection (retain data)
az backup protection disable \
  --vault-name myRSV --resource-group myRG \
  --container-name myVM --item-name myVM \
  --backup-management-type AzureIaasVM --yes

# Delete backup data permanently
az backup protection disable \
  --vault-name myRSV --resource-group myRG \
  --container-name myVM --item-name myVM \
  --backup-management-type AzureIaasVM \
  --delete-backup-data true --yes

# List policies
az backup policy list --vault-name myRSV --resource-group myRG --output table

# Update policy on existing protected item
az backup item set-policy \
  --vault-name myRSV --resource-group myRG \
  --container-name myVM --item-name myVM \
  --backup-management-type AzureIaasVM \
  --policy-name NewPolicy
```
