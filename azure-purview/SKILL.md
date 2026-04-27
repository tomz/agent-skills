---
name: azure-purview
description: Microsoft Purview — data governance, data catalog, data map, lineage, classification, sensitivity labels, compliance, Data Loss Prevention
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Microsoft Purview Skill

Microsoft Purview is a unified data governance and compliance platform covering
data discovery, classification, lineage, cataloging, and compliance (formerly
Azure Purview + Microsoft 365 Compliance Center).

---

## Architecture Overview

```
Microsoft Purview Account
├── Data Map                    # Metadata store — scanned assets, lineage
│   ├── Collections             # Hierarchical namespace for assets & access
│   ├── Data Sources            # Registered Azure/on-prem/multi-cloud sources
│   ├── Scan Rule Sets          # Which file types / tables to scan
│   └── Integration Runtimes    # Self-hosted IR for on-prem sources
├── Data Catalog                # Search, browse, curate
│   ├── Business Glossary       # Terms, owners, stewards
│   ├── Asset Management        # Schemas, classifications, sensitivity
│   └── Insights                # Data estate analytics
└── Compliance (M365 Purview)
    ├── Information Protection  # Sensitivity labels, label policies
    ├── Data Loss Prevention    # DLP policies (Exchange, SharePoint, Teams, Endpoint)
    ├── eDiscovery              # Content search, holds, review sets
    ├── Audit                   # Unified audit log
    ├── Retention               # Policies + labels
    └── Insider Risk Management
```

---

## Prerequisites & Setup

```bash
# Install Purview CLI extension
az extension add --name purview

# Register provider
az provider register --namespace Microsoft.Purview

# Create a Purview account
az purview account create \
  --name "purview-prod-001" \
  --resource-group "rg-governance" \
  --location "eastus" \
  --managed-group-name "purview-managed-rg"

# Get account endpoint
az purview account show \
  --name "purview-prod-001" \
  --resource-group "rg-governance" \
  --query "properties.endpoints"
```

---

## Data Map — Sources & Scans

### Register a Data Source

```bash
# REST API — register Azure Data Lake Storage Gen2
PURVIEW_ENDPOINT="https://purview-prod-001.purview.azure.com"
TOKEN=$(az account get-access-token --resource "https://purview.azure.com" --query accessToken -o tsv)

curl -X PUT "$PURVIEW_ENDPOINT/scan/datasources/adls-prod" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "AdlsGen2",
    "properties": {
      "endpoint": "https://prodsa.dfs.core.windows.net/",
      "collection": { "referenceName": "root-collection", "type": "CollectionReference" }
    }
  }'
```

### Common Source Types

| Kind | Description |
|------|-------------|
| `AdlsGen2` | Azure Data Lake Storage Gen2 |
| `AzureSqlDatabase` | Azure SQL DB |
| `AzureSynapseWorkspace` | Synapse Analytics |
| `AzureSqlDataWarehouse` | Dedicated SQL Pool |
| `AzureCosmosDb` | Cosmos DB |
| `AzureDataExplorer` | Azure Data Explorer |
| `PowerBI` | Power BI tenant |
| `SqlServerDatabase` | On-prem SQL (Self-hosted IR) |
| `AmazonS3` | AWS S3 (multi-cloud) |

### Create & Run a Scan

```bash
# Create scan for ADLS source
curl -X PUT "$PURVIEW_ENDPOINT/scan/datasources/adls-prod/scans/scan-weekly" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "AdlsGen2Msi",
    "properties": {
      "scanRulesetName": "AdlsGen2",
      "scanRulesetType": "System",
      "collection": { "referenceName": "data-team", "type": "CollectionReference" }
    }
  }'

# Trigger scan run
curl -X PUT "$PURVIEW_ENDPOINT/scan/datasources/adls-prod/scans/scan-weekly/runs/run-$(date +%Y%m%d)" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "scanLevel": "Incremental" }'

# Check scan run status
curl "$PURVIEW_ENDPOINT/scan/datasources/adls-prod/scans/scan-weekly/runs" \
  -H "Authorization: Bearer $TOKEN" | jq '.value[] | {status, startTime, endTime}'
```

### Scan Rule Sets

```bash
# List system scan rule sets
curl "$PURVIEW_ENDPOINT/scan/systemScanRulesets" \
  -H "Authorization: Bearer $TOKEN" | jq '.value[].name'

# Create custom scan rule set (include only .parquet and .delta files)
curl -X PUT "$PURVIEW_ENDPOINT/scan/scanrulesets/custom-parquet-only" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "AdlsGen2",
    "properties": {
      "scanningRule": {
        "fileExtensions": ["Parquet", "OrcFile"],
        "customFileExtensions": [{ "customFileExtension": ".delta" }]
      },
      "classificationRules": [
        { "name": "MICROSOFT.FINANCIAL.CREDIT_CARD_NUMBER", "kind": "System" },
        { "name": "MICROSOFT.PERSONAL.EMAIL", "kind": "System" }
      ],
      "includedPatterns": ["prod/*"],
      "excludedPatterns": ["archive/*", "tmp/*"]
    }
  }'
```

### Self-Hosted Integration Runtime (on-prem sources)

```bash
# Create self-hosted IR
curl -X PUT "$PURVIEW_ENDPOINT/scan/integrationruntimes/onprem-ir" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "kind": "SelfHosted", "properties": { "description": "On-prem SQL IR" } }'

# Get auth keys to install IR agent on Windows server
curl "$PURVIEW_ENDPOINT/scan/integrationruntimes/onprem-ir/listAuthKeys" \
  -X POST -H "Authorization: Bearer $TOKEN" | jq '.authKey1'
# Install Microsoft Integration Runtime on Windows, paste the key
```

---

## Collections & Access Control

```bash
# Create a child collection
curl -X PUT "$PURVIEW_ENDPOINT/account/collections/finance-team" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "friendlyName": "Finance Team",
    "parentCollection": { "referenceName": "root-collection" },
    "description": "Finance data assets"
  }'

# Assign Collection Admin role
# Roles: collectionadmin, datareaderwith metadata, datacurator, datasource administrator
curl -X PUT "$PURVIEW_ENDPOINT/account/collections/finance-team/metadataPolicies/policy-001" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "attributeRules": [{
        "kind": "attributerule",
        "id": "rule-datacurator",
        "name": "datacurator",
        "dnfCondition": [[{
          "attributeName": "principal.microsoft.groups",
          "attributeValueIncludes": "00000000-group-guid-here"
        }]]
      }],
      "collection": { "referenceName": "finance-team" },
      "decisionRules": [{
        "kind": "decisionrule",
        "effect": "Permit",
        "dnfCondition": [[{ "attributeName": "derived.purview.role", "attributeValueIncludes": "datacurators" }]]
      }]
    }
  }'
```

---

## Data Catalog — Search & Assets

```bash
# Full-text search across catalog
curl -X POST "$PURVIEW_ENDPOINT/catalog/api/search/query" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "keywords": "customer sales",
    "limit": 10,
    "filter": {
      "and": [
        { "entityType": "azure_datalake_gen2_path" },
        { "classification": "MICROSOFT.PERSONAL.EMAIL" }
      ]
    },
    "facets": [
      { "facet": "entityType", "count": 10, "sort": { "count": "desc" } },
      { "facet": "classification", "count": 10 }
    ]
  }' | jq '.value[] | {name: .name, qualifiedName: .qualifiedName}'

# Get asset details by GUID
ASSET_GUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
curl "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/entity/guid/$ASSET_GUID" \
  -H "Authorization: Bearer $TOKEN" | jq '.entity | {typeName, attributes}'

# Update asset — add description and owners
curl -X PUT "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/entity/guid/$ASSET_GUID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entity": {
      "guid": "'$ASSET_GUID'",
      "typeName": "azure_datalake_gen2_path",
      "attributes": {
        "userDescription": "Daily sales transactions from POS system",
        "contacts": {
          "owner": [{ "id": "user-object-id", "info": "Data owner" }],
          "expert": [{ "id": "steward-object-id", "info": "Data steward" }]
        }
      }
    }
  }'
```

### Business Glossary

```bash
# Create glossary term
curl -X POST "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/glossary/term" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Customer Lifetime Value",
    "shortDescription": "Total revenue from a customer over their relationship",
    "longDescription": "CLV = (Average Purchase Value) × (Purchase Frequency) × (Customer Lifespan)",
    "abbreviation": "CLV",
    "status": "Approved",
    "anchor": { "glossaryGuid": "glossary-guid-here" },
    "contacts": {
      "owner": [{ "id": "data-owner-object-id", "info": "Business owner" }]
    }
  }'

# Assign term to asset
curl -X POST "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/entity/guid/$ASSET_GUID/classifications" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{ "typeName": "AtlasGlossaryTerm", "entityGuid": "'$ASSET_GUID'" }]'
```

---

## Classification

### Built-in Classifications (sample)

| Category | Examples |
|----------|----------|
| Financial | `MICROSOFT.FINANCIAL.CREDIT_CARD_NUMBER`, `MICROSOFT.FINANCIAL.ROUTING_NUMBER` |
| Personal | `MICROSOFT.PERSONAL.EMAIL`, `MICROSOFT.PERSONAL.IPADDRESS`, `MICROSOFT.PERSONAL.NAME` |
| Government | `MICROSOFT.GOVERNMENT.US.SOCIAL_SECURITY_NUMBER`, `MICROSOFT.GOVERNMENT.US.PASSPORT_NUMBER` |
| Health | `MICROSOFT.HEALTH.DISEASE`, `MICROSOFT.HEALTH.DRUG` |

### Custom Classification Rules

```bash
# Create regex-based custom classification
curl -X POST "$PURVIEW_ENDPOINT/scan/classificationrules/EmployeeID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "Custom",
    "properties": {
      "description": "Internal employee ID format EMP-XXXXXX",
      "classificationName": "Custom.HR.EmployeeID",
      "ruleStatus": "Enabled",
      "minimumPercentageMatch": 60,
      "classificationAction": "Label",
      "dataPatterns": [{
        "kind": "Regex",
        "pattern": "EMP-\\d{6}",
        "minimumDocumentPercentage": 10
      }],
      "columnPatterns": [{
        "kind": "Regex",
        "pattern": "emp(loyee)?[_-]?id",
        "minimumDocumentPercentage": 0
      }]
    }
  }'

# Manually apply classification to an asset
curl -X POST "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/entity/guid/$ASSET_GUID/classifications" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{ "typeName": "Custom.HR.EmployeeID" }]'
```

---

## Lineage Tracking

```bash
# Get upstream lineage (what feeds this asset)
curl "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/lineage/$ASSET_GUID?direction=INPUT&depth=3" \
  -H "Authorization: Bearer $TOKEN" | jq '.relations[] | {fromEntityId, toEntityId}'

# Get downstream lineage (what this asset feeds)
curl "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/lineage/$ASSET_GUID?direction=OUTPUT&depth=3" \
  -H "Authorization: Bearer $TOKEN"

# Push custom lineage via Atlas API
curl -X POST "$PURVIEW_ENDPOINT/catalog/api/atlas/v2/entity/bulk" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entities": [
      {
        "typeName": "Process",
        "attributes": {
          "qualifiedName": "custom://etl/sales-transform@prod",
          "name": "Sales ETL Transform",
          "inputs": [{ "guid": "source-asset-guid", "typeName": "DataSet" }],
          "outputs": [{ "guid": "target-asset-guid", "typeName": "DataSet" }]
        }
      }
    ]
  }'
```

---

## Sensitivity Labels & Information Protection

Sensitivity labels are configured in the Microsoft Purview compliance portal (M365).

```powershell
# PowerShell — Security & Compliance module
Install-Module ExchangeOnlineManagement
Connect-IPPSSession

# List sensitivity labels
Get-Label | Select-Object DisplayName, Priority, IsActive

# Create a label
New-Label -DisplayName "Confidential - Internal" `
  -Name "Confidential_Internal" `
  -Tooltip "Sensitive internal data not for external sharing" `
  -EncryptionEnabled $false `
  -ContentType "File, Email"

# Publish label policy
New-LabelPolicy -Name "Corp-Label-Policy" `
  -Labels "Confidential_Internal","Public","Highly Confidential" `
  -ExchangeLocation All `
  -SharePointLocation All
```

---

## Data Loss Prevention (DLP)

```powershell
# Connect to compliance
Connect-IPPSSession

# Create DLP policy — block external sharing of credit cards
New-DlpCompliancePolicy -Name "Block-CCN-External" `
  -ExchangeLocation All `
  -SharePointLocation All `
  -TeamsLocation All `
  -Mode Enable

New-DlpComplianceRule -Name "Block-CCN-External-Rule" `
  -Policy "Block-CCN-External" `
  -ContentContainsSensitiveInformation @{
    Name = "Credit Card Number"; minCount = 1; minConfidence = 85
  } `
  -BlockAccess $true `
  -AccessScope NotInOrganization `
  -NotifyUser Owner `
  -NotifyPolicyTipCustomText "This file contains credit card numbers and cannot be shared externally."

# Get DLP policy match reports
Get-DlpDetectionsReport -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) |
  Group-Object PolicyName | Select-Object Name, Count
```

---

## eDiscovery & Audit

```powershell
# Create eDiscovery case
New-ComplianceCase -Name "Legal-Hold-2026-Q1"

# Add hold — preserve all email from a custodian
New-CaseHoldPolicy -Name "Custodian-Hold" `
  -Case "Legal-Hold-2026-Q1" `
  -ExchangeLocation "john.doe@company.com"

New-CaseHoldRule -Name "Custodian-Hold-Rule" `
  -Policy "Custodian-Hold"

# Search for content
New-ComplianceSearch -Name "Search-Q1-Contracts" `
  -Case "Legal-Hold-2026-Q1" `
  -ExchangeLocation All `
  -ContentMatchQuery 'kind:email AND "contract" AND (received>=2026-01-01 AND received<=2026-03-31)'

Start-ComplianceSearch -Identity "Search-Q1-Contracts"
Get-ComplianceSearch -Identity "Search-Q1-Contracts" | Select-Object Status, Items, Size

# Unified audit log search
Search-UnifiedAuditLog -StartDate "2026-04-01" -EndDate "2026-04-24" `
  -RecordType SharePointFileOperation `
  -Operations FileDownloaded, FileCopied `
  -ResultSize 1000 |
  Select-Object CreationDate, UserIds, Operations, AuditData
```

---

## Retention Policies

```powershell
# Create 7-year retention policy for financial records
New-RetentionCompliancePolicy -Name "Financial-7yr-Retain" `
  -SharePointLocation "https://company.sharepoint.com/sites/Finance" `
  -OneDriveLocation All

New-RetentionComplianceRule -Name "Financial-7yr-Rule" `
  -Policy "Financial-7yr-Retain" `
  -RetentionDuration 2555 `       # 7 years in days
  -RetentionComplianceAction Keep `
  -ExpirationDateOption CreationAgeInDays

# Publish retention label for manual application
New-RetentionCompliancePolicy -Name "HR-Records-Label-Policy" `
  -IsSimulationPolicy $false `
  -PublishComplianceTag "HR-7yr"
```

---

## Insider Risk Management

Configure in Microsoft Purview compliance portal (no PowerShell API for policy creation).

Key policy templates:
- **Data theft by departing users** — trigger: HR resignation date; indicators: bulk downloads, USB copy
- **Data leaks** — trigger: DLP match; indicators: cloud upload, email external
- **Security policy violations** — trigger: Microsoft Defender alerts
- **Risky browser usage** — trigger: browsing to restricted categories

```powershell
# View insider risk alerts
Get-InsiderRiskPolicy | Select-Object Name, Status, PolicyTemplate

# Triage alert (requires Insider Risk Management Analysts role)
# Use the compliance portal UI: https://compliance.microsoft.com/insiderriskmgmt
```

---

## Monitoring & Insights

```bash
# Purview Insights — asset statistics via REST
curl "$PURVIEW_ENDPOINT/mapanddiscover/reports/assetDistribution/allRegisteredSources" \
  -H "Authorization: Bearer $TOKEN" | jq '.value[] | {sourceName, assetCount}'

# Data stewardship report — assets with/without owners
curl "$PURVIEW_ENDPOINT/mapanddiscover/reports/stewardship" \
  -H "Authorization: Bearer $TOKEN"

# Azure Monitor — Purview diagnostic logs
az monitor diagnostic-settings create \
  --name "purview-diag" \
  --resource "/subscriptions/SUB_ID/resourceGroups/rg-governance/providers/Microsoft.Purview/accounts/purview-prod-001" \
  --logs '[{"category":"ScanStatusLogEvent","enabled":true},{"category":"DataSensitivityLogEvent","enabled":true}]' \
  --workspace "/subscriptions/SUB_ID/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-prod"
```

---

## Bicep — Purview Account

```bicep
resource purviewAccount 'Microsoft.Purview/accounts@2021-07-01' = {
  name: 'purview-prod-001'
  location: 'eastus'
  identity: { type: 'SystemAssigned' }
  properties: {
    publicNetworkAccess: 'Disabled'      // use Private Endpoints
    managedResourceGroupName: 'purview-managed-rg'
  }
}

// Assign Purview MSI as Storage Blob Data Reader on ADLS
resource rbac 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, purviewAccount.identity.principalId, 'StorageBlobDataReader')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')
    principalId: purviewAccount.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

---

## Quick Reference

| Task | Method |
|------|--------|
| Onboard data source | REST API `PUT /scan/datasources/{name}` |
| Run a scan | REST API `PUT /scan/datasources/{src}/scans/{scan}/runs/{run}` |
| Search catalog | REST API `POST /catalog/api/search/query` |
| Create sensitivity label | PowerShell `New-Label` |
| Create DLP policy | PowerShell `New-DlpCompliancePolicy` + `New-DlpComplianceRule` |
| eDiscovery hold | PowerShell `New-CaseHoldPolicy` |
| View audit logs | PowerShell `Search-UnifiedAuditLog` |
| Retention | PowerShell `New-RetentionCompliancePolicy` |
