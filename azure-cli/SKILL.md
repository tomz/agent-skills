---
name: azure-cli
description: Azure CLI (az) patterns, authentication, resource management, output formats, JMESPath queries, and productivity tips
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

# Azure CLI (az) Skills

## Installation & Setup

```bash
# Install (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Update
az upgrade

# Check version
az version

# Install a specific extension
az extension add --name containerapp
az extension add --name aks-preview

# List installed extensions
az extension list --output table

# Update all extensions
az extension update --all
```

## Authentication

### Interactive Login
```bash
# Browser-based login (default)
az login

# Device code flow (headless/SSH environments)
az login --use-device-code

# Login to a specific tenant
az login --tenant <tenant-id>

# Verify who you are
az account show
az ad signed-in-user show
```

### Service Principal Authentication
```bash
# Create a service principal with Contributor on a subscription
az ad sp create-for-rbac \
  --name "my-sp-name" \
  --role Contributor \
  --scopes /subscriptions/<sub-id> \
  --sdk-auth   # outputs JSON for GitHub Actions / CI

# Login as a service principal
az login \
  --service-principal \
  --username <app-id> \
  --password <client-secret> \
  --tenant <tenant-id>

# Login with a certificate
az login \
  --service-principal \
  --username <app-id> \
  --certificate /path/to/cert.pem \
  --tenant <tenant-id>
```

### Managed Identity (within Azure resources)
```bash
# Login using system-assigned managed identity
az login --identity

# Login using user-assigned managed identity
az login --identity --username <client-id-or-resource-id>
```

### Multi-Subscription Management
```bash
# List all subscriptions
az account list --output table

# Set active subscription
az account set --subscription "My Subscription Name"
az account set --subscription <subscription-id>

# Show current subscription
az account show --query "{name:name, id:id, tenantId:tenantId}"
```

## Resource Groups

```bash
# Create
az group create --name myRG --location eastus

# List all resource groups
az group list --output table

# List with a filter
az group list --query "[?location=='eastus']" --output table

# Show details
az group show --name myRG

# Delete (with confirmation prompt)
az group delete --name myRG

# Delete without prompt (careful!)
az group delete --name myRG --yes --no-wait

# Lock a resource group (prevent deletion)
az lock create \
  --lock-type CanNotDelete \
  --name protect-prod \
  --resource-group myRG

# Export all resources as ARM template
az group export --name myRG > exported-template.json
```

## Output Formats

```bash
# Table — human readable (default for list commands)
az vm list --output table

# JSON — full response, good for scripting
az vm show --name myVM --resource-group myRG --output json

# TSV — tab-separated, ideal for bash variable assignment
az vm show --name myVM --resource-group myRG \
  --query "id" --output tsv

# YAML — readable structured output
az vm show --name myVM --resource-group myRG --output yaml

# jsonc — JSON with comments (useful for templates)
az deployment group show --name myDeploy --resource-group myRG --output jsonc

# Set default output format (persists across sessions)
az config set core.output=table
```

## JMESPath Queries (`--query`)

JMESPath is Azure CLI's built-in query language. Always use with `--output tsv` or `--output json`.

```bash
# Get a single field
az vm show -g myRG -n myVM --query "id" -o tsv

# Multiple fields as object
az vm show -g myRG -n myVM \
  --query "{name:name, size:hardwareProfile.vmSize, os:storageProfile.osDisk.osType}" \
  -o json

# Filter a list
az vm list -g myRG \
  --query "[?powerState=='VM running']" -o table

# Filter and project
az vm list -g myRG \
  --query "[].{Name:name, Location:location, Size:hardwareProfile.vmSize}" \
  -o table

# Contains filter
az resource list \
  --query "[?contains(type, 'Microsoft.Network')]" \
  -o table

# Get first item
az vm list -g myRG --query "[0].name" -o tsv

# Length
az vm list -g myRG --query "length(@)" -o tsv

# Nested array flattening
az aks list --query "[].agentPoolProfiles[].{name:name, count:count}" -o table

# Sort
az vm list --query "sort_by(@, &name)[].name" -o tsv
```

## Common Resource Commands

### Virtual Machines
```bash
az vm create -g myRG -n myVM \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --no-wait

az vm start -g myRG -n myVM
az vm stop -g myRG -n myVM       # deallocates (stops billing for compute)
az vm deallocate -g myRG -n myVM
az vm list-sizes --location eastus --output table
az vm get-instance-view -g myRG -n myVM --query "instanceView.statuses[1]"
```

### Storage
```bash
az storage account create \
  -n mystorageacct \
  -g myRG \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot

# Get connection string
az storage account show-connection-string -n mystorageacct -g myRG -o tsv

# Upload a blob
az storage blob upload \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myblob.txt \
  --file ./localfile.txt \
  --auth-mode login
```

### Key Vault
```bash
az keyvault create -n myKV -g myRG --location eastus
az keyvault secret set --vault-name myKV --name MySecret --value "s3cr3t"
az keyvault secret show --vault-name myKV --name MySecret --query "value" -o tsv
```

## Scripting Patterns

### Wait for async operations
```bash
# --no-wait starts operation asynchronously; poll with:
az vm wait -g myRG -n myVM --created
az vm wait -g myRG -n myVM --running
az vm wait -g myRG -n myVM --deleted
```

### Loop over resources
```bash
# Delete all VMs in a resource group
for vm in $(az vm list -g myRG --query "[].name" -o tsv); do
  echo "Deleting $vm..."
  az vm delete -g myRG -n "$vm" --yes --no-wait
done
```

### Capture IDs for later use
```bash
SUBNET_ID=$(az network vnet subnet show \
  -g myRG --vnet-name myVNet --name mySubnet \
  --query id -o tsv)

az aks create -g myRG -n myAKS \
  --vnet-subnet-id "$SUBNET_ID" \
  ...
```

### Check if resource exists before creating
```bash
if az group show -n myRG &>/dev/null; then
  echo "RG exists, skipping..."
else
  az group create -n myRG -l eastus
fi
```

## Configuration & Defaults

```bash
# Set persistent defaults (avoid repeating -g and -l)
az config set defaults.group=myRG
az config set defaults.location=eastus

# View current config
az config get

# Clear a default
az config unset defaults.group

# Enable/disable telemetry
az config set core.collect_telemetry=false
```

## Tags

```bash
# Add tags at creation
az group create -n myRG -l eastus \
  --tags Environment=Production CostCenter=12345

# Update tags on existing resource (replaces all tags)
az resource tag \
  --ids /subscriptions/<sub>/resourceGroups/myRG \
  --tags Environment=Production Owner=TeamA

# Append a tag (merge pattern)
EXISTING=$(az resource show --ids <resource-id> --query "tags" -o json)
# Then use jq to merge and re-apply
```

## Gotchas & Tips

- **`--no-wait` is not always safe**: some resources have dependencies — ensure downstream steps wait properly.
- **JMESPath is case-sensitive**: `powerState` not `powerstate`.
- **TSV strips quotes**: ideal for shell variable assignment; never pipe JSON fields to TSV without `--query`.
- **`az login` caches tokens in `~/.azure/`**: on shared machines, use `az logout` when done.
- **Resource names are global or regional**: Storage account names must be globally unique; Key Vault names must be globally unique.
- **ARM throttling**: The Azure Resource Manager rate-limits at ~1200 reads/5min per subscription. Scripts that list thousands of resources can hit this.
- **Use `--verbose` or `--debug`** to inspect HTTP calls when troubleshooting API errors.
- **Cloud environments**: Use `az cloud set --name AzureUSGovernment` for Gov cloud; `az cloud list` to see all.
- **Always pin `az` version in CI**: Use Docker image `mcr.microsoft.com/azure-cli:2.x.x` for reproducible builds.
