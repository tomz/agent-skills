---
name: azure-devtest-labs
description: Azure DevTest Labs — lab environments, VM management, formulas, artifacts, cost management, auto-shutdown, claimable VMs, policies
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure DevTest Labs

Azure DevTest Labs provides self-service sandbox environments for dev/test teams.
It enforces cost controls and policies while letting developers provision VMs and
PaaS environments without waiting for IT approvals.

---

## Lab Creation

```bash
# Register provider
az provider register --namespace Microsoft.DevTestLab

# Create a lab
az lab create \
  --name myDevLab \
  --resource-group myRG \
  --location eastus

# List labs
az lab list --resource-group myRG --output table

# Show lab details
az lab show --name myDevLab --resource-group myRG

# Delete lab
az lab delete --name myDevLab --resource-group myRG --yes
```

---

## Virtual Machine Management

### Create a VM from a Gallery Image

```bash
az lab vm create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-01 \
  --image-name "Ubuntu Server 22.04 LTS" \
  --image-type GalleryImage \
  --size Standard_D2s_v3 \
  --user-name devuser \
  --generate-ssh-keys \
  --notes "Developer workstation for feature/login branch"

# List VMs in a lab
az lab vm list \
  --lab-name myDevLab \
  --resource-group myRG \
  --output table

# Start / stop / restart
az lab vm start  --lab-name myDevLab --resource-group myRG --name dev-vm-01
az lab vm stop   --lab-name myDevLab --resource-group myRG --name dev-vm-01
az lab vm restart --lab-name myDevLab --resource-group myRG --name dev-vm-01

# Delete a VM
az lab vm delete --lab-name myDevLab --resource-group myRG --name dev-vm-01 --yes

# Show connection info
az lab vm show \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-01 \
  --query "{fqdn:fqdn, ssh:sshPort, rdp:rdpPort}"
```

### Create a Windows VM with RDP

```bash
az lab vm create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name win-dev-01 \
  --image-name "Windows 11 Pro" \
  --image-type GalleryImage \
  --size Standard_D4s_v3 \
  --user-name labadmin \
  --password "$(az keyvault secret show --vault-name myKV --name lab-pw --query value -o tsv)"
```

---

## Custom Images vs. Formulas

### Custom Images

Capture a VM's state as a reusable base image within the lab:

```bash
# Capture a running VM as a custom image
az lab custom-image create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "Ubuntu-DevBase-v2" \
  --description "Pre-installed: Docker, Node 20, Python 3.12" \
  --source-vm-id $(az lab vm show \
      --lab-name myDevLab --resource-group myRG --name dev-vm-01 \
      --query id -o tsv)

# List custom images
az lab custom-image list \
  --lab-name myDevLab \
  --resource-group myRG \
  --output table

# Create VM from custom image
az lab vm create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-02 \
  --image-name "Ubuntu-DevBase-v2" \
  --image-type CustomImage \
  --size Standard_D2s_v3 \
  --user-name devuser \
  --generate-ssh-keys
```

### Formulas

Formulas are reusable VM templates that include image, size, artifacts, and
network settings — developers fill in only what changes (VM name, notes):

```bash
# Create a formula
az lab formula create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "Ubuntu-Docker-Dev" \
  --description "Ubuntu 22.04 + Docker + VS Code artifacts" \
  --os-type Linux \
  --size Standard_D2s_v3 \
  --image-name "Ubuntu Server 22.04 LTS" \
  --image-type GalleryImage \
  --user-name devuser \
  --generate-ssh-keys

# List formulas
az lab formula list \
  --lab-name myDevLab \
  --resource-group myRG \
  --output table

# Create VM from formula
az lab vm create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-03 \
  --formula-id $(az lab formula show \
      --lab-name myDevLab --resource-group myRG --name "Ubuntu-Docker-Dev" \
      --query id -o tsv)
```

**When to use each:**

| | Custom Image | Formula |
|-|-------------|---------|
| Creation speed | Faster (no artifact install) | Slower (runs artifacts at provision) |
| Storage cost | Higher (full VHD) | Lower (just metadata + base image) |
| Freshness | Snapshot in time | Always uses latest artifacts |
| Best for | Stable, slow-changing environments | Frequently updated toolchains |

---

## Artifacts

Artifacts are installation/configuration scripts applied to VMs at creation time
or on-demand afterward.

### Built-in Artifacts

```bash
# List available public artifacts
az lab artifact list \
  --lab-name myDevLab \
  --resource-group myRG \
  --artifact-source-name "Public Repo" \
  --output table

# Apply artifact to a running VM
az lab vm apply-artifacts \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-01 \
  --artifacts '[
    {
      "artifactId": "/artifactsources/public repo/artifacts/linux-install-docker",
      "parameters": []
    },
    {
      "artifactId": "/artifactsources/public repo/artifacts/linux-install-nodejs",
      "parameters": [
        { "name": "nodeVersion", "value": "20" }
      ]
    }
  ]'
```

### Custom Artifact Repository

Store custom artifacts in GitHub or Azure Repos:

```
my-artifacts/
├── install-myapp/
│   ├── artifactfile.json
│   └── install.sh
└── configure-devenv/
    ├── artifactfile.json
    └── setup.ps1
```

`artifactfile.json` structure:

```json
{
  "$schema": "https://raw.githubusercontent.com/Azure/azure-devtestlab/master/schemas/2016-11-28/dtlArtifacts.json",
  "title": "Install MyApp",
  "description": "Installs MyApp and configures it for development",
  "iconUri": "https://example.com/icon.png",
  "targetOsType": "Linux",
  "parameters": {
    "version": {
      "type": "string",
      "displayName": "App Version",
      "description": "Version of MyApp to install",
      "defaultValue": "2.1.0"
    }
  },
  "runCommand": {
    "commandToExecute": "[concat('bash install.sh ', parameters('version'))]"
  }
}
```

Add the repo to the lab:

```bash
az lab artifact-source create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "MyCompany Artifacts" \
  --uri "https://github.com/myorg/dtl-artifacts.git" \
  --source-type GitHub \
  --branch main \
  --folder-path /artifacts \
  --personal-access-token <GITHUB-PAT>

# For Azure Repos
az lab artifact-source create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "Internal Artifacts" \
  --uri "https://dev.azure.com/myorg/myproject/_git/dtl-artifacts" \
  --source-type VsoGit \
  --branch main \
  --folder-path /artifacts \
  --personal-access-token <ADO-PAT>
```

---

## ARM Environment Templates

Deploy multi-resource PaaS environments (App Service + SQL + Redis) as a
single lab environment from ARM templates:

```bash
# Add ARM template repository
az lab artifact-source create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "Environment Templates" \
  --uri "https://github.com/myorg/dtl-environments.git" \
  --source-type GitHub \
  --branch main \
  --folder-path /environments \
  --personal-access-token <GITHUB-PAT>

# Create an environment
az lab environment create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "feature-branch-env" \
  --arm-template "WebApp-SQL" \
  --parameters "[{\"name\":\"appName\",\"value\":\"feature-x-app\"}]"

# List environments
az lab environment list \
  --lab-name myDevLab \
  --resource-group myRG \
  --output table

# Delete environment
az lab environment delete \
  --lab-name myDevLab \
  --resource-group myRG \
  --name "feature-branch-env" \
  --yes
```

---

## Claimable VMs

Claimable VMs are pre-provisioned but unclaimed — developers self-serve
from a pool without waiting for VM creation:

```bash
# Create a claimable VM (no --user-name assignment)
az lab vm create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name claimable-ubuntu-01 \
  --image-name "Ubuntu Server 22.04 LTS" \
  --image-type GalleryImage \
  --size Standard_D2s_v3 \
  --allow-claim true

# Developer claims an available VM
az lab vm claim \
  --lab-name myDevLab \
  --resource-group myRG \
  --name claimable-ubuntu-01

# Developer unclaims (returns to pool)
az lab vm unclaim \
  --lab-name myDevLab \
  --resource-group myRG \
  --name claimable-ubuntu-01
```

---

## Auto-Shutdown and Auto-Start

```bash
# Enable auto-shutdown at 7pm UTC
az lab vm auto-shutdown \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-01 \
  --time 1900 \
  --timezone "UTC"

# Set lab-wide default auto-shutdown
az lab update \
  --name myDevLab \
  --resource-group myRG \
  --shutdown-time 1900 \
  --timezone-id "UTC"

# Enable auto-start schedule (lab-wide)
az lab schedule create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name LabVmsStartup \
  --task-type LabVmsStartupTask \
  --time 0800 \
  --timezone-id "UTC" \
  --week-days Monday Tuesday Wednesday Thursday Friday
```

---

## Cost Management and Policies

### Lab Policies

```bash
# Limit VMs per user
az lab policy set \
  --lab-name myDevLab \
  --resource-group myRG \
  --policy-set-name default \
  --name MaxVmsAllowedPerUser \
  --fact-name MaxVmsAllowedPerUser \
  --threshold 3 \
  --evaluator-type MaxValuePolicy \
  --status Enabled

# Limit total VMs in lab
az lab policy set \
  --lab-name myDevLab \
  --resource-group myRG \
  --policy-set-name default \
  --name MaxVmsAllowedPerLab \
  --fact-name MaxVmsAllowedPerLab \
  --threshold 20 \
  --evaluator-type MaxValuePolicy \
  --status Enabled

# Restrict allowed VM sizes
az lab policy set \
  --lab-name myDevLab \
  --resource-group myRG \
  --policy-set-name default \
  --name AllowedVmSizesInLab \
  --fact-name LabVmSize \
  --threshold "[\"Standard_D2s_v3\",\"Standard_D4s_v3\",\"Standard_B2s\",\"Standard_B4ms\"]" \
  --evaluator-type AllowedValuesPolicy \
  --status Enabled

# List policies
az lab policy list \
  --lab-name myDevLab \
  --resource-group myRG \
  --policy-set-name default \
  --output table
```

### Cost Tracking

```bash
# View lab cost report for current month
az lab cost show \
  --lab-name myDevLab \
  --resource-group myRG \
  --name targetCost \
  --output table

# Set monthly budget alert thresholds
az lab cost create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name targetCost \
  --target-cost 500 \
  --currency-code USD \
  --cycle-start-date $(date +%Y-%m-01) \
  --cycle-type CalendarMonth \
  --cost-thresholds "[
    {\"thresholdId\":\"1\",\"percentageThreshold\":{\"thresholdValue\":75},\"displayOnChart\":true,\"sendNotificationWhenExceeded\":true},
    {\"thresholdId\":\"2\",\"percentageThreshold\":{\"thresholdValue\":100},\"displayOnChart\":true,\"sendNotificationWhenExceeded\":true}
  ]"
```

---

## Virtual Network Integration

```bash
# Add an existing VNet to the lab
az lab vnet create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name myVNet \
  --subnet myDevSubnet

# Create a VM in the lab's VNet
az lab vm create \
  --lab-name myDevLab \
  --resource-group myRG \
  --name dev-vm-vnet \
  --image-name "Ubuntu Server 22.04 LTS" \
  --image-type GalleryImage \
  --size Standard_D2s_v3 \
  --vnet-name myVNet \
  --subnet myDevSubnet \
  --disallow-public-ip-address true \
  --user-name devuser \
  --generate-ssh-keys
```

---

## CI/CD Integration

### GitHub Actions — Provision on PR Open

```yaml
# .github/workflows/dtl-environment.yml
name: Dev Environment
on:
  pull_request:
    types: [opened, reopened]

jobs:
  provision:
    runs-on: ubuntu-latest
    steps:
      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Create Lab VM
        run: |
          az lab vm create \
            --lab-name myDevLab \
            --resource-group myRG \
            --name "pr-${{ github.event.number }}-vm" \
            --formula-id "/subscriptions/${{ secrets.SUBSCRIPTION_ID }}/resourceGroups/myRG/providers/Microsoft.DevTestLab/labs/myDevLab/formulas/Ubuntu-Docker-Dev" \
            --notes "PR #${{ github.event.number }}: ${{ github.event.pull_request.title }}"

      - name: Get Connection Info
        run: |
          az lab vm show \
            --lab-name myDevLab \
            --resource-group myRG \
            --name "pr-${{ github.event.number }}-vm" \
            --query "{fqdn:fqdn}" -o tsv

  cleanup:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - name: Delete Lab VM
        run: |
          az lab vm delete \
            --lab-name myDevLab \
            --resource-group myRG \
            --name "pr-${{ github.event.number }}-vm" \
            --yes
```

---

## Bicep Example

```bicep
param labName string = 'myDevLab'
param location string = resourceGroup().location

resource lab 'Microsoft.DevTestLab/labs@2018-09-15' = {
  name: labName
  location: location
  properties: {
    labStorageType: 'Premium'
    environmentPermission: 'Contributor'
  }
}

// Auto-shutdown policy
resource autoShutdown 'Microsoft.DevTestLab/labs/schedules@2018-09-15' = {
  parent: lab
  name: 'LabVmsShutdown'
  properties: {
    status: 'Enabled'
    taskType: 'LabVmsShutdownTask'
    dailyRecurrence: {
      time: '1900'
    }
    timeZoneId: 'UTC'
    notificationSettings: {
      status: 'Enabled'
      timeInMinutes: 30
    }
  }
}

// Max VMs per user policy
resource vmPerUserPolicy 'Microsoft.DevTestLab/labs/policySets/policies@2018-09-15' = {
  name: '${labName}/default/MaxVmsAllowedPerUser'
  properties: {
    status: 'Enabled'
    factName: 'MaxVmsAllowedPerUser'
    threshold: '3'
    evaluatorType: 'MaxValuePolicy'
  }
}

// Allowed VM sizes
resource allowedSizesPolicy 'Microsoft.DevTestLab/labs/policySets/policies@2018-09-15' = {
  name: '${labName}/default/AllowedVmSizesInLab'
  properties: {
    status: 'Enabled'
    factName: 'LabVmSize'
    threshold: '["Standard_D2s_v3","Standard_D4s_v3","Standard_B2s"]'
    evaluatorType: 'AllowedValuesPolicy'
  }
}

output labId string = lab.id
output labUrl string = 'https://portal.azure.com/#resource${lab.id}'
```

---

## az lab CLI Quick Reference

```bash
# Labs
az lab create / list / show / delete / update

# VMs
az lab vm create / list / show / delete
az lab vm start / stop / restart
az lab vm claim / unclaim           # claimable pool management
az lab vm apply-artifacts           # install artifacts on running VM

# Formulas
az lab formula create / list / show / delete

# Custom Images
az lab custom-image create / list / show / delete

# Environments (ARM templates)
az lab environment create / list / show / delete

# Policies
az lab policy set / list / show

# Artifact Sources
az lab artifact-source create / list / show / delete
az lab artifact list                # list artifacts in a source

# Schedules
az lab schedule create / list / show / delete

# Cost
az lab cost show / create

# VNet
az lab vnet create / list / show / delete
```

---

## Best Practices

- **Formulas over custom images** for rapidly evolving toolchains; use custom
  images only when provisioning time is critical (e.g., GPU workloads).
- **Claimable VM pools** for teams with spiky demand — pre-warm a pool of 5–10
  VMs overnight so developers never wait.
- **Auto-shutdown + 30-min notification** — reduces idle cost significantly; let
  developers cancel shutdown from the notification email if they're still working.
- **Per-user VM limits (3–5)** prevent accidental cost overruns from forgotten VMs.
- **Allowed VM sizes** prevent developers from provisioning 32-core behemoths
  for light workloads; maintain an approved size list aligned with cost targets.
- **Monthly budget thresholds** at 75% and 100% give finance time to react
  before the billing period closes.
- **Artifact repos in Azure Repos** for internal tooling — use PATs with minimal
  scope (repo:read only) and rotate them on a schedule.
- **Tag VMs** with `project`, `team`, and `pr-number` to enable cost attribution
  and automated cleanup pipelines.
