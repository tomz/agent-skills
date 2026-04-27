---
name: azure-iac
description: Infrastructure as Code for Azure — Bicep (preferred), ARM templates, Terraform azurerm, deployment modes, what-if, modules, parameter files, deployment stacks
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

# Azure Infrastructure as Code (IaC) Skills

## Bicep (Preferred over ARM)

Bicep is a DSL that transpiles to ARM JSON. It is the recommended IaC language for Azure-native deployments.

### Getting Started
```bash
# Install Bicep CLI (standalone)
az bicep install
az bicep upgrade
az bicep version

# VS Code extension: ms-azuretools.vscode-bicep
# Provides intellisense, linting, and inline ARM type docs
```

### Basic Bicep File Structure
```bicep
// main.bicep
targetScope = 'resourceGroup'   // default; also: subscription, managementGroup, tenant

@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('Location for all resources')
param location string = resourceGroup().location

@minLength(3)
@maxLength(24)
param storageAccountName string

var storageSkuName = environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSkuName
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
  tags: {
    environment: environment
  }
}

output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
```

### Bicep Modules
```bicep
// modules/storage.bicep — reusable module
param name string
param location string
param sku string = 'Standard_LRS'

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: name
  location: location
  sku: { name: sku }
  kind: 'StorageV2'
}

output id string = sa.id

// main.bicep — consuming the module
module storage './modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    name: 'myuniquestorage001'
    location: location
    sku: 'Standard_GRS'
  }
}

// Cross-resource-group module deployment
module remoteStorage './modules/storage.bicep' = {
  name: 'remoteStorageDeployment'
  scope: resourceGroup('other-rg')
  params: {
    name: 'crossrgstorage001'
    location: location
  }
}
```

### Bicep — Loops & Conditions
```bicep
// Loop over array
param subnetConfigs array = [
  { name: 'web', prefix: '10.0.1.0/24' }
  { name: 'app', prefix: '10.0.2.0/24' }
  { name: 'db',  prefix: '10.0.3.0/24' }
]

resource subnets 'Microsoft.Network/virtualNetworks/subnets@2023-09-01' = [for subnet in subnetConfigs: {
  name: subnet.name
  parent: vnet
  properties: {
    addressPrefix: subnet.prefix
  }
}]

// Conditional resource
resource diagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = if (environment == 'prod') {
  name: 'prod-diag'
  scope: storageAccount
  properties: {
    workspaceId: logAnalyticsWorkspaceId
    logs: []
    metrics: [{ category: 'Transaction'; enabled: true }]
  }
}
```

### Bicep Parameter Files
```json
// main.bicepparam (preferred — Bicep native syntax)
using './main.bicep'

param environment = 'prod'
param storageAccountName = 'myprodstore001'
param location = 'eastus2'
```

```json
// main.parameters.json (ARM-compatible format, also valid)
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": { "value": "prod" },
    "storageAccountName": { "value": "myprodstore001" }
  }
}
```

## Deploying Bicep / ARM Templates

### Resource Group Scope
```bash
# Deploy Bicep
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters environment=prod storageAccountName=mystore001

# Deploy with parameter file
az deployment group create \
  -g myRG \
  --template-file main.bicep \
  --parameters @main.parameters.json

# Deploy with .bicepparam file
az deployment group create \
  -g myRG \
  --parameters main.bicepparam
```

### Subscription Scope
```bash
az deployment sub create \
  --location eastus \
  --template-file subscription-level.bicep \
  --parameters @params.json
```

### Management Group Scope
```bash
az deployment mg create \
  --management-group-id myMG \
  --location eastus \
  --template-file mg-policies.bicep
```

## Deployment Modes

```bash
# Incremental (default) — adds/updates resources, does NOT delete resources not in template
az deployment group create -g myRG --template-file main.bicep --mode Incremental

# Complete — deletes resources in RG not in template (DANGEROUS — use with caution)
az deployment group create -g myRG --template-file main.bicep --mode Complete
```

> **Warning**: Complete mode will delete any resource in the resource group not defined
> in the template. Always run `--what-if` first when using Complete mode.

## What-If (Dry Run)

```bash
# Preview changes before deploying
az deployment group what-if \
  -g myRG \
  --template-file main.bicep \
  --parameters @params.json

# What-if for subscription scope
az deployment sub what-if \
  --location eastus \
  --template-file main.bicep

# Output as JSON for scripting
az deployment group what-if \
  -g myRG \
  --template-file main.bicep \
  --result-format FullResourcePayloads \
  --output json
```

## Deployment Stacks (Preview → GA)

Deployment stacks manage a set of resources as a single atomic unit with deny-assignment support.

```bash
# Create/update a deployment stack at resource group scope
az stack group create \
  --name myStack \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @params.json \
  --deny-settings-mode None \
  --action-on-unmanage deleteResources

# With deny assignments (prevents manual changes)
az stack group create \
  --name myStack \
  --resource-group myRG \
  --template-file main.bicep \
  --deny-settings-mode DenyWriteAndDelete \
  --deny-settings-apply-to-child-scopes

# Show stack
az stack group show --name myStack --resource-group myRG

# Delete stack (removes all managed resources)
az stack group delete --name myStack --resource-group myRG
```

## ARM Templates (JSON)

Bicep is preferred, but ARM JSON is still used in some contexts.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "metadata": { "description": "Storage account name" }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "variables": {
    "storageSku": "Standard_LRS"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": { "name": "[variables('storageSku')]" },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
  "outputs": {
    "storageId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    }
  }
}
```

### ARM Template Functions (useful to know)
```
resourceId(...)           → /subscriptions/.../resourceGroups/.../providers/...
reference(resourceId)     → runtime resource properties
concat(a, b, c)           → string concat
format('{0}-{1}', a, b)   → string format
uniqueString(base)        → deterministic 13-char hash (great for globally unique names)
toLower(str)              → lowercase
substring(str, 0, 8)      → substring
resourceGroup().location  → location of current RG
subscription().id         → current subscription ID
```

## Terraform (azurerm Provider)

```hcl
# versions.tf
terraform {
  required_version = ">= 1.7"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.110"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateaccount001"
    container_name       = "tfstate"
    key                  = "prod/main.tfstate"
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
  subscription_id = var.subscription_id
}

# main.tf
resource "azurerm_resource_group" "main" {
  name     = "my-rg-${var.environment}"
  location = var.location
  tags     = local.common_tags
}

resource "azurerm_storage_account" "main" {
  name                     = "mystg${var.environment}001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"
  allow_nested_items_to_be_public = false
}

# locals.tf
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    CostCenter  = var.cost_center
  }
}
```

### Terraform Workflow
```bash
terraform init                         # initialize backend + providers
terraform fmt -recursive               # format all .tf files
terraform validate                     # syntax + schema validation
terraform plan -out=tfplan             # create plan file
terraform show -json tfplan            # inspect plan as JSON
terraform apply tfplan                 # apply saved plan (no prompt)
terraform apply -auto-approve          # apply without plan file (CI use)
terraform destroy -auto-approve        # destroy all (careful!)
terraform state list                   # list managed resources
terraform state show azurerm_storage_account.main
terraform import azurerm_resource_group.main /subscriptions/<sub>/resourceGroups/my-rg
terraform output -json                 # show outputs
```

## Bicep Best Practices

1. **Use modules** for reusable components (networking, storage, identity).
2. **Always specify API versions explicitly** — prevents silent breaking changes.
3. **Use `@description` decorators** on all params and outputs.
4. **Use `@secure()` for secrets** — prevents logging in deployment history.
5. **Use `uniqueString(resourceGroup().id)` for globally unique names** (storage, KV, ACR).
6. **Never hardcode subscription IDs or tenant IDs** — use `subscription().id`.
7. **Use `.bicepparam` files** (not `.parameters.json`) for new projects.
8. **Run `az bicep build`** in CI to catch lint errors before deployment.
9. **Store state/history in deployment names** — use `utcNow()` for unique deployment names in pipelines.
10. **Prefer `existing` resource references** over hardcoded IDs:
    ```bicep
    resource existingKV 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
      name: keyVaultName
    }
    // Then reference: existingKV.id
    ```

## Gotchas

- **`dependsOn` is usually inferred** in Bicep from resource property references — only add explicit `dependsOn` when there's no property reference to infer from.
- **Complete mode deletes unlisted resources** — this includes resources created manually or by other processes.
- **ARM template `reference()` at deploy time** can fail if the resource doesn't exist yet; use `dependsOn`.
- **Bicep modules cannot output secrets directly** — use Key Vault references instead.
- **Terraform `azurerm` provider `features {}` block is mandatory** even if empty.
- **Terraform state locking**: AzureRM backend uses blob leases. If a plan fails mid-apply, you may need to break the lease manually.
- **`uniqueString()` is NOT random** — it's deterministic based on input. Same inputs always produce same output.
