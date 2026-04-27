---
name: azure-arc
description: Azure Arc — hybrid and multi-cloud management, Arc-enabled servers, Kubernetes, SQL, data services, GitOps, extensions
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Arc Skill

Azure Arc extends Azure management plane (RBAC, Policy, Defender, Monitor,
Update Manager) to any infrastructure — on-prem servers, edge, and other clouds.

---

## Architecture Overview

```
Azure Arc Control Plane (Azure Resource Manager)
├── Arc-enabled Servers              # Windows/Linux machines → Microsoft.HybridCompute/machines
│   ├── Azure Monitor Agent          # Extension: metrics & logs
│   ├── Defender for Servers         # Extension: MDE onboarding
│   ├── Machine Configuration        # DSC-like policy (formerly Guest Config)
│   └── SSH Access                   # Just-in-time SSH via AAD
├── Arc-enabled Kubernetes           # Any CNCF-conformant cluster
│   ├── GitOps (Flux v2)             # Declarative app deployment
│   ├── Azure Monitor Container Insights
│   ├── Defender for Containers
│   └── Open Service Mesh (extension)
├── Arc-enabled Data Services        # Run Azure PaaS on your K8s
│   ├── SQL Managed Instance         # Always Current or Evergreen
│   └── PostgreSQL (preview)
├── Arc-enabled SQL Server           # On-prem SQL Server → inventory + Defender
└── Custom Locations                 # Target for Arc data services deployment
```

---

## Prerequisites

```bash
# Required providers
az provider register --namespace Microsoft.HybridCompute
az provider register --namespace Microsoft.GuestConfiguration
az provider register --namespace Microsoft.HybridConnectivity
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ExtendedLocation
az provider register --namespace Microsoft.AzureArcData

# Install CLI extensions
az extension add --name connectedmachine
az extension add --name k8s-configuration
az extension add --name k8s-extension
az extension add --name customlocation
az extension add --name arcdata
```

---

## Arc-enabled Servers

### Onboarding at Scale — Service Principal Method

```bash
# Create service principal for onboarding
SP=$(az ad sp create-for-rbac \
  --name "arc-onboard-sp" \
  --role "Azure Connected Machine Onboarding" \
  --scopes "/subscriptions/$(az account show --query id -o tsv)" \
  --years 1)

SP_ID=$(echo $SP | jq -r '.appId')
SP_SECRET=$(echo $SP | jq -r '.password')
TENANT=$(echo $SP | jq -r '.tenant')

# Generate onboarding script (Windows PS1)
az connectedmachine generate-install-script \
  --subscription-id "$(az account show --query id -o tsv)" \
  --resource-group "rg-arc-servers" \
  --location "eastus" \
  --service-principal-id "$SP_ID" \
  --service-principal-secret "$SP_SECRET" \
  --cloud "AzureCloud" \
  --output-type "windows" > onboard-windows.ps1

# Or Linux bash script
az connectedmachine generate-install-script \
  --resource-group "rg-arc-servers" \
  --location "eastus" \
  --service-principal-id "$SP_ID" \
  --service-principal-secret "$SP_SECRET" \
  --output-type "bash" > onboard-linux.sh

# Run on target Linux machine:
# sudo bash onboard-linux.sh
```

### Manage Connected Machines

```bash
# List all Arc servers
az connectedmachine list --resource-group "rg-arc-servers" \
  --query "[].{name:name, status:status, os:properties.osSku, location:location}" \
  -o table

# Show single machine details
az connectedmachine show \
  --name "srv-onprem-01" \
  --resource-group "rg-arc-servers"

# Delete (does not uninstall the agent)
az connectedmachine delete \
  --name "srv-onprem-01" \
  --resource-group "rg-arc-servers" --yes

# Uninstall agent on Linux (run on the machine):
# sudo azcmagent disconnect --service-principal-id $SP_ID --service-principal-secret $SP_SECRET
# sudo apt remove azcmagent  # or yum remove
```

### Extensions

```bash
# List available extensions
az connectedmachine extension list \
  --machine-name "srv-onprem-01" \
  --resource-group "rg-arc-servers"

# Install Azure Monitor Agent
az connectedmachine extension create \
  --name "AzureMonitorLinuxAgent" \
  --machine-name "srv-onprem-01" \
  --resource-group "rg-arc-servers" \
  --location "eastus" \
  --type "AzureMonitorLinuxAgent" \
  --publisher "Microsoft.Azure.Monitor" \
  --type-handler-version "1.0" \
  --enable-auto-upgrade true

# Install Dependency Agent (for VM insights)
az connectedmachine extension create \
  --name "DependencyAgentLinux" \
  --machine-name "srv-onprem-01" \
  --resource-group "rg-arc-servers" \
  --type "DependencyAgentLinux" \
  --publisher "Microsoft.Azure.Monitoring.DependencyAgent" \
  --type-handler-version "9.0" \
  --enable-auto-upgrade true

# Install Microsoft Defender (MDE)
az connectedmachine extension create \
  --name "MDE.Linux" \
  --machine-name "srv-onprem-01" \
  --resource-group "rg-arc-servers" \
  --type "MDE.Linux" \
  --publisher "Microsoft.Azure.AzureDefenderForServers" \
  --type-handler-version "1.0"

# Custom Script Extension
az connectedmachine extension create \
  --name "CustomScriptExtension" \
  --machine-name "srv-onprem-01" \
  --resource-group "rg-arc-servers" \
  --type "CustomScript" \
  --publisher "Microsoft.Compute" \
  --type-handler-version "1.10" \
  --settings '{"commandToExecute":"apt-get update && apt-get install -y nginx"}'
```

### Machine Configuration (DSC-like Policy)

```bash
# Assign built-in guest config policy — ensure password policy
az policy assignment create \
  --name "require-password-complexity" \
  --policy "azureLinuxBaseline" \
  --scope "/subscriptions/SUB_ID/resourceGroups/rg-arc-servers"

# Check compliance
az policy state list \
  --resource "/subscriptions/SUB_ID/resourceGroups/rg-arc-servers/providers/Microsoft.HybridCompute/machines/srv-onprem-01" \
  --query "[].{policy:policyDefinitionName, compliance:complianceState}" \
  -o table
```

### SSH Access to Arc Servers

```bash
# Enable SSH connectivity
az connectedmachine extension create \
  --name "SSH" \
  --machine-name "srv-onprem-01" \
  --resource-group "rg-arc-servers" \
  --type "SSH" \
  --publisher "Microsoft.Azure.OpenSSH" \
  --type-handler-version "1.0"

# SSH via Azure (no public IP required)
az ssh arc \
  --resource-group "rg-arc-servers" \
  --name "srv-onprem-01" \
  --local-user azureuser

# SSH with proxy jump (for non-AAD machines)
az ssh arc \
  --resource-group "rg-arc-servers" \
  --name "srv-onprem-01" \
  --local-user azureuser \
  --private-key-file ~/.ssh/id_rsa
```

---

## Arc-enabled Kubernetes

### Connect a Cluster

```bash
# Connect any kubeconfig-accessible cluster
az connectedk8s connect \
  --name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --location "eastus" \
  --kube-config ~/.kube/config \
  --kube-context "prod-cluster"

# Verify connection
az connectedk8s show \
  --name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --query "connectivityStatus"

# List all connected clusters
az connectedk8s list --resource-group "rg-arc-k8s" -o table

# Disconnect (removes Arc agent, keeps cluster)
az connectedk8s delete \
  --name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" --yes
```

### GitOps with Flux v2

```bash
# Create GitOps configuration — deploy from Git repo
az k8s-configuration flux create \
  --name "app-config" \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters \
  --namespace flux-system \
  --url "https://github.com/myorg/k8s-configs" \
  --branch main \
  --scope cluster \
  --kustomization name=infra path=./infrastructure prune=true \
  --kustomization name=apps path=./apps/prod prune=true dependsOn=infra

# With SSH auth (private repo)
az k8s-configuration flux create \
  --name "private-config" \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters \
  --url "git@github.com:myorg/private-configs.git" \
  --branch main \
  --ssh-private-key-file ~/.ssh/flux_deploy_key \
  --kustomization name=base path=./base

# Show GitOps config status
az k8s-configuration flux show \
  --name "app-config" \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters

# List all configs
az k8s-configuration flux list \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters -o table
```

### Cluster Extensions

```bash
# Install Azure Monitor Container Insights
az k8s-extension create \
  --name "azuremonitor-containers" \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters \
  --extension-type "Microsoft.AzureMonitor.Containers" \
  --configuration-settings \
    "logAnalyticsWorkspaceResourceID=/subscriptions/SUB/resourceGroups/rg/providers/Microsoft.OperationalInsights/workspaces/law-prod"

# Install Defender for Containers
az k8s-extension create \
  --name "microsoft.azuredefender.kubernetes" \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters \
  --extension-type "microsoft.azuredefender.kubernetes"

# Install Open Service Mesh
az k8s-extension create \
  --name "osm" \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters \
  --extension-type "Microsoft.openservicemesh" \
  --version 1.2.3

# List extensions
az k8s-extension list \
  --cluster-name "k8s-onprem-prod" \
  --resource-group "rg-arc-k8s" \
  --cluster-type connectedClusters -o table
```

### Azure Policy for Arc Kubernetes

```bash
# Enforce namespace isolation across Arc clusters
az policy assignment create \
  --name "k8s-require-ns-labels" \
  --policy "a8eff44f-8c98-4a71-abda-eb9e7efe9a00" \
  --scope "/subscriptions/SUB_ID/resourceGroups/rg-arc-k8s"

# View policy compliance per cluster
az policy state summarize \
  --resource "/subscriptions/SUB_ID/resourceGroups/rg-arc-k8s/providers/Microsoft.Kubernetes/connectedClusters/k8s-onprem-prod"
```

---

## Arc-enabled Data Services

### Data Controller

```bash
# Create custom location first (needed for data services)
ARC_CLUSTER_ID=$(az connectedk8s show --name "k8s-onprem-prod" -g "rg-arc-k8s" --query id -o tsv)

az customlocation create \
  --name "cl-onprem-prod" \
  --resource-group "rg-arc-data" \
  --namespace arc-data-ns \
  --host-resource-id "$ARC_CLUSTER_ID" \
  --cluster-extension-ids "$(az k8s-extension show --name arc-data-services --cluster-name k8s-onprem-prod -g rg-arc-k8s --cluster-type connectedClusters --query id -o tsv)"

# Create data controller (directly connected mode)
az arcdata dc create \
  --name "arc-dc-prod" \
  --resource-group "rg-arc-data" \
  --location "eastus" \
  --custom-location "cl-onprem-prod" \
  --profile-name "azure-arc-aks-premium-storage" \
  --auto-upload-logs true \
  --auto-upload-metrics true
```

### SQL Managed Instance on Arc

```bash
# Create Arc SQL MI (General Purpose)
az sql mi-arc create \
  --name "arc-sqlmi-prod" \
  --resource-group "rg-arc-data" \
  --custom-location "cl-onprem-prod" \
  --tier GeneralPurpose \
  --cores-limit 4 \
  --cores-request 2 \
  --memory-limit 8Gi \
  --memory-request 4Gi \
  --storage-class-data "local-path" \
  --volume-size-data 100Gi

# List Arc SQL MI instances
az sql mi-arc list --resource-group "rg-arc-data" -o table

# Get connection endpoint
az sql mi-arc show \
  --name "arc-sqlmi-prod" \
  --resource-group "rg-arc-data" \
  --query "properties.k8SRaw.status.endpoints"

# Connect via sqlcmd (from a pod in-cluster or with LB IP)
sqlcmd -S <endpoint>,1433 -U sa -P '<password>'
```

---

## Arc-enabled SQL Server

```bash
# Onboard existing on-prem SQL Server (runs alongside Arc server agent)
# The arc-enabled SQL Server extension is auto-deployed when SQL is detected

# View Arc SQL Servers
az sql server-arc list --resource-group "rg-arc-servers" -o table

# Enable best practices assessment
az sql server-arc update \
  --name "srv-sql-01_SQLServer2019" \
  --resource-group "rg-arc-servers" \
  --assessment-enabled true \
  --schedule-type weekly

# Enable Defender for SQL
az security atp storage update \
  --resource-group "rg-arc-servers" \
  --storage-account "law-prod" \
  --is-enabled true
```

---

## Update Manager for Arc Servers

```bash
# Check update compliance
az maintenance configuration list --resource-group "rg-arc-servers"

# Create maintenance config — patch every Tuesday 2am
az maintenance configuration create \
  --name "monthly-patching" \
  --resource-group "rg-arc-servers" \
  --location "eastus" \
  --maintenance-scope "InGuestPatch" \
  --recur-every "Week Tuesday" \
  --duration "02:00" \
  --start-date-time "2026-05-01 02:00" \
  --time-zone "UTC" \
  --install-patches-linux-parameters classifications="Critical,Security" \
  --install-patches-windows-parameters classifications="Critical,Security,UpdateRollup" \
  --reboot-setting "IfRequired"

# Assign config to Arc machine
az maintenance assignment create \
  --name "srv-onprem-01-patching" \
  --resource-group "rg-arc-servers" \
  --location "eastus" \
  --maintenance-configuration-id "/subscriptions/SUB_ID/resourceGroups/rg-arc-servers/providers/Microsoft.Maintenance/maintenanceConfigurations/monthly-patching" \
  --resource-id "/subscriptions/SUB_ID/resourceGroups/rg-arc-servers/providers/Microsoft.HybridCompute/machines/srv-onprem-01"
```

---

## Monitoring & Diagnostics

```bash
# Log Analytics query — Arc server heartbeat
az monitor log-analytics query \
  --workspace "law-prod" \
  --analytics-query "
    Heartbeat
    | where Category == 'Azure Arc'
    | summarize LastHeartbeat=max(TimeGenerated) by Computer, OSType
    | where LastHeartbeat < ago(15m)
  "

# Arc server connection events
az monitor log-analytics query \
  --workspace "law-prod" \
  --analytics-query "
    AzureActivity
    | where ResourceProvider == 'Microsoft.HybridCompute'
    | summarize count() by OperationName, bin(TimeGenerated, 1h)
  "
```

---

## Bicep — Arc Server Extension at Scale

```bicep
// Apply AMA extension to all Arc servers in RG via Azure Policy + DeployIfNotExists
resource amaPolicy 'Microsoft.Authorization/policyDefinitions@2021-06-01' existing = {
  name: '845857af-0333-4c5d-bbbc-6076697da122'  // Deploy AMA for Arc Linux
  scope: subscription()
}

resource policyAssignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'deploy-ama-arc-linux'
  location: 'eastus'
  identity: { type: 'SystemAssigned' }
  properties: {
    policyDefinitionId: amaPolicy.id
    parameters: {
      logAnalyticsWorkspace: {
        value: logAnalyticsWorkspaceId
      }
    }
  }
}

// Grant remediation rights
resource remediation 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(subscription().id, 'ama-arc-remediation')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'b24988ac-6180-42a0-ab88-20f7382dd24c')
    principalId: policyAssignment.identity.principalId
  }
}
```

---

## Quick Reference

| Resource | CLI Command |
|----------|------------|
| List Arc servers | `az connectedmachine list -g RG -o table` |
| Onboard server | `az connectedmachine generate-install-script` → run script |
| Add extension | `az connectedmachine extension create` |
| Connect K8s | `az connectedk8s connect --name NAME -g RG` |
| GitOps config | `az k8s-configuration flux create` |
| K8s extension | `az k8s-extension create` |
| Arc SQL MI | `az sql mi-arc create` |
| Update patching | `az maintenance configuration create` |
