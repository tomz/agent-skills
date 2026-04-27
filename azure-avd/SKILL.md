---
name: azure-avd
description: Azure Virtual Desktop — host pools, session hosts, application groups, FSLogix, MSIX app attach, scaling plans, RemoteApp
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Virtual Desktop (AVD) Skill

Azure Virtual Desktop (formerly Windows Virtual Desktop) delivers cloud-hosted
Windows desktops and RemoteApp applications to any device.

---

## Architecture Overview

```
AVD Control Plane (Microsoft-managed)
├── Host Pool                          # Collection of session hosts
│   ├── Pooled (multi-session)         # Multiple users per VM (Windows 10/11 multi-session)
│   └── Personal (VDI)                 # 1 user per VM (persistent or automatic assignment)
├── Session Hosts                      # Azure VMs running AVD agent
├── Application Groups
│   ├── Desktop App Group              # Full desktop (one per host pool max)
│   └── RemoteApp Group                # Published individual apps (multiple per pool)
├── Workspace                          # Logical grouping shown in client feed
└── Scaling Plans                      # Autoscale session hosts by schedule/load
```

---

## Prerequisites & Registration

```bash
az provider register --namespace Microsoft.DesktopVirtualization

# Install extension
az extension add --name desktopvirtualization

# Required roles
# - Desktop Virtualization Contributor (on the subscription/RG)
# - Virtual Machine Contributor (on the vNet RG, for session host deployment)
```

---

## Host Pools

### Create a Pooled Host Pool

```bash
az desktopvirtualization hostpool create \
  --name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --host-pool-type Pooled \
  --load-balancer-type BreadthFirst \
  --max-session-limit 10 \
  --preferred-app-group-type Desktop \
  --registration-info expiration-time="$(date -u -d '+48 hours' '+%Y-%m-%dT%H:%M:%SZ')" \
                       registration-token-operation=Update \
  --start-vm-on-connect true \
  --validation-environment false \
  --custom-rdp-property "audiocapturemode:i:1;audiomode:i:0;redirectclipboard:i:1;redirectprinters:i:0;devicestoredirect:s:*"
```

### Create a Personal Host Pool

```bash
az desktopvirtualization hostpool create \
  --name "hp-personal-dev" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --host-pool-type Personal \
  --personal-desktop-assignment-type Automatic \
  --load-balancer-type Persistent \
  --preferred-app-group-type Desktop \
  --registration-info expiration-time="$(date -u -d '+48 hours' '+%Y-%m-%dT%H:%M:%SZ')" \
                       registration-token-operation=Update
```

### Host Pool Operations

```bash
# List host pools
az desktopvirtualization hostpool list -g "rg-avd" -o table

# Get registration token (for adding session hosts)
az desktopvirtualization hostpool retrieve-registration-token \
  --name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --query token -o tsv

# Update RDP properties
az desktopvirtualization hostpool update \
  --name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --custom-rdp-property "audiocapturemode:i:1;drivestoredirect:s:;redirectclipboard:i:1"

# Common RDP properties
# drivestoredirect:s:*        → redirect all drives
# drivestoredirect:s:         → redirect no drives
# redirectclipboard:i:1       → enable clipboard
# audiocapturemode:i:1        → redirect microphone
# camerastoredirect:s:*       → redirect cameras
# usbdevicestoredirect:s:*    → redirect USB
```

---

## Session Hosts

### Deploy Session Hosts via ARM/Bicep

```bash
# Get registration token
REG_TOKEN=$(az desktopvirtualization hostpool retrieve-registration-token \
  --name "hp-pooled-prod" -g "rg-avd" --query token -o tsv)

# Deploy VMs with AVD extension (ARM template pattern)
# The AVD agent extension joins the VM to the host pool
az deployment group create \
  --resource-group "rg-avd-hosts" \
  --template-uri "https://raw.githubusercontent.com/Azure/RDS-Templates/master/ARM-wvd-templates/DSC/Configuration.zip" \
  --parameters hostpoolToken="$REG_TOKEN"
```

```bicep
// Session host VM — Bicep snippet
resource sessionHostNic 'Microsoft.Network/networkInterfaces@2023-06-01' = {
  name: 'nic-avd-host-${i}'
  location: location
  properties: {
    ipConfigurations: [{
      name: 'ipconfig1'
      properties: {
        subnet: { id: subnetId }
        privateIPAllocationMethod: 'Dynamic'
      }
    }]
  }
}

resource sessionHostVm 'Microsoft.Compute/virtualMachines@2023-09-01' = {
  name: 'vm-avd-host-${padLeft(i, 2, '0')}'
  location: location
  properties: {
    hardwareProfile: { vmSize: 'Standard_D4s_v5' }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsDesktop'
        offer: 'windows-11'
        sku: 'win11-23h2-avd'           // Multi-session: 'win11-23h2-avd'
        version: 'latest'
      }
      osDisk: { createOption: 'FromImage', managedDisk: { storageAccountType: 'Premium_LRS' } }
    }
    osProfile: {
      computerName: 'avdhost${i}'
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    networkProfile: {
      networkInterfaces: [{ id: sessionHostNic.id }]
    }
  }
}

// Join to Entra ID (AAD Join — no domain controller needed)
resource aadLoginExtension 'Microsoft.Compute/virtualMachines/extensions@2023-09-01' = {
  parent: sessionHostVm
  name: 'AADLoginForWindows'
  location: location
  properties: {
    publisher: 'Microsoft.Azure.ActiveDirectory'
    type: 'AADLoginForWindows'
    typeHandlerVersion: '1.0'
    settings: { mdmId: '' }     // Leave empty if not using Intune
  }
}

// AVD DSC extension — register with host pool
resource avdDscExtension 'Microsoft.Compute/virtualMachines/extensions@2023-09-01' = {
  parent: sessionHostVm
  name: 'DSC'
  location: location
  dependsOn: [aadLoginExtension]
  properties: {
    publisher: 'Microsoft.Powershell'
    type: 'DSC'
    typeHandlerVersion: '2.73'
    settings: {
      modulesUrl: 'https://wvdportalstorageblob.blob.core.windows.net/galleryartifacts/Configuration_1.0.02797.442.zip'
      configurationFunction: 'Configuration.ps1\\AddSessionHost'
      properties: {
        HostPoolName: 'hp-pooled-prod'
        RegistrationInfoToken: regToken
        AadJoin: true
        UseAgentDownloadEndpoint: true
      }
    }
  }
}
```

### Manage Session Hosts

```bash
# List session hosts in a host pool
az desktopvirtualization sessionhost list \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" -o table

# Drain a host (no new sessions) for maintenance
az desktopvirtualization sessionhost update \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --name "vm-avd-host-01.domain.com" \
  --allow-new-session false

# Remove session host registration
az desktopvirtualization sessionhost delete \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --name "vm-avd-host-01.domain.com" --yes

# List active user sessions
az desktopvirtualization usersession list \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --host-name "vm-avd-host-01.domain.com"

# Log off a user session
az desktopvirtualization usersession delete \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --host-name "vm-avd-host-01.domain.com" \
  --user-session-id 1
```

---

## Application Groups

```bash
# Create Desktop Application Group
az desktopvirtualization applicationgroup create \
  --name "ag-desktop-prod" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --host-pool-arm-path "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod" \
  --application-group-type Desktop

# Create RemoteApp Application Group
az desktopvirtualization applicationgroup create \
  --name "ag-remoteapp-office" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --host-pool-arm-path "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod" \
  --application-group-type RemoteApp

# Publish a RemoteApp (e.g., Excel)
az desktopvirtualization application create \
  --name "Excel" \
  --application-group-name "ag-remoteapp-office" \
  --resource-group "rg-avd" \
  --file-path "C:\\Program Files\\Microsoft Office\\root\\Office16\\EXCEL.EXE" \
  --friendly-name "Microsoft Excel" \
  --command-line-policy DoNotAllow \
  --show-in-portal true \
  --icon-path "C:\\Program Files\\Microsoft Office\\root\\Office16\\EXCEL.EXE" \
  --icon-index 0

# List published apps
az desktopvirtualization application list \
  --application-group-name "ag-remoteapp-office" \
  --resource-group "rg-avd" -o table

# Assign AAD group to application group
az role assignment create \
  --role "Desktop Virtualization User" \
  --assignee-object-id "aad-group-object-id" \
  --assignee-principal-type Group \
  --scope "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/applicationGroups/ag-desktop-prod"
```

---

## Workspaces

```bash
# Create workspace
az desktopvirtualization workspace create \
  --name "ws-prod" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --friendly-name "Production Workspace" \
  --application-group-references \
    "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/applicationGroups/ag-desktop-prod" \
    "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/applicationGroups/ag-remoteapp-office"

# Add app group to existing workspace
az desktopvirtualization workspace update \
  --name "ws-prod" \
  --resource-group "rg-avd" \
  --application-group-references \
    "/subscriptions/SUB_ID/.../ag-desktop-prod" \
    "/subscriptions/SUB_ID/.../ag-remoteapp-office" \
    "/subscriptions/SUB_ID/.../ag-new-group"
```

---

## FSLogix Profile Containers

FSLogix redirects user profiles to VHD(x) files on a network share (Azure Files recommended).

```powershell
# Install FSLogix on session host (run via Custom Script Extension or image build)
# Download from: https://aka.ms/fslogix_download
# Silent install:
& "FSLogixAppsSetup.exe" /install /quiet /norestart

# Configure FSLogix via registry
$registryPath = "HKLM:\SOFTWARE\FSLogix\Profiles"
New-Item -Path $registryPath -Force
Set-ItemProperty -Path $registryPath -Name "Enabled"           -Value 1
Set-ItemProperty -Path $registryPath -Name "VHDLocations"      -Value "\\storageaccount.file.core.windows.net\fslogix-profiles"
Set-ItemProperty -Path $registryPath -Name "SizeInMBs"         -Value 30720        # 30 GB
Set-ItemProperty -Path $registryPath -Name "VolumeType"        -Value "VHDX"
Set-ItemProperty -Path $registryPath -Name "FlipFlopProfileDirectoryName" -Value 1  # username_SID format
Set-ItemProperty -Path $registryPath -Name "PreventLoginWithFailure" -Value 1
Set-ItemProperty -Path $registryPath -Name "PreventLoginWithTempProfile" -Value 1
```

```bash
# Create Azure Files share for FSLogix
az storage account create \
  --name "stavdprofiles" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --sku "Premium_LRS" \
  --kind "FileStorage" \
  --https-only true

az storage share-rm create \
  --storage-account "stavdprofiles" \
  --name "fslogix-profiles" \
  --resource-group "rg-avd" \
  --quota 5120  # GB — use large quota, billing is per used GB (Premium)

# Grant storage SMB access to AVD users AAD group
STORAGE_ID=$(az storage account show --name "stavdprofiles" -g "rg-avd" --query id -o tsv)
az role assignment create \
  --role "Storage File Data SMB Share Contributor" \
  --assignee-object-id "avd-users-group-object-id" \
  --assignee-principal-type Group \
  --scope "$STORAGE_ID"

# Grant Elevated rights for admins
az role assignment create \
  --role "Storage File Data SMB Share Elevated Contributor" \
  --assignee-object-id "avd-admins-group-object-id" \
  --assignee-principal-type Group \
  --scope "$STORAGE_ID"
```

---

## MSIX App Attach

MSIX App Attach delivers apps as MSIX packages mounted from VHD on a share —
no installation on session hosts.

```powershell
# On session host: enable MSIX staging
# Handled by AVD host pool config + MSIX package registration

# Create MSIX image from .msix package (requires MSIX Packaging Tool)
# 1. Package app as .msix
# 2. Convert to VHD/CIM using MSIXMGR tool:
#    msixmgr.exe -Unpack -packagePath "MyApp.msix" -destination "\\share\msix\MyApp.vhd" -applyACLs -create -filetype "vhd" -rootDirectory "apps"
```

```bash
# Register MSIX package in AVD
az desktopvirtualization msixpackage create \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --msix-package-full-name "MyApp_1.0.0.0_x64__xyz" \
  --msix-image-path "\\\\stavdapps.file.core.windows.net\\msix\\MyApp.vhd" \
  --package-name "MyApp" \
  --package-relative-path "\\apps\\MyApp" \
  --display-name "My Application" \
  --version "1.0.0.0" \
  --is-active true \
  --is-regular-registration false  # true = register on logon, false = on-demand

# List MSIX packages
az desktopvirtualization msixpackage list \
  --host-pool-name "hp-pooled-prod" \
  --resource-group "rg-avd" -o table
```

---

## Scaling Plans

```bash
# Create scaling plan
az desktopvirtualization scalingplan create \
  --name "sp-weekday" \
  --resource-group "rg-avd" \
  --location "eastus" \
  --host-pool-type Pooled \
  --time-zone "Eastern Standard Time" \
  --schedules '[{
    "name": "weekday-schedule",
    "daysOfWeek": ["Monday","Tuesday","Wednesday","Thursday","Friday"],
    "rampUpStartTime": {"hour": 7, "minute": 0},
    "rampUpLoadBalancingAlgorithm": "BreadthFirst",
    "rampUpMinimumHostsPct": 20,
    "rampUpCapacityThresholdPct": 60,
    "peakStartTime": {"hour": 9, "minute": 0},
    "peakLoadBalancingAlgorithm": "BreadthFirst",
    "rampDownStartTime": {"hour": 18, "minute": 0},
    "rampDownLoadBalancingAlgorithm": "DepthFirst",
    "rampDownMinimumHostsPct": 10,
    "rampDownCapacityThresholdPct": 90,
    "rampDownForceLogoffUsers": false,
    "rampDownWaitTimeMinutes": 30,
    "rampDownNotificationMessage": "Session ending in 30 minutes. Please save your work.",
    "offPeakStartTime": {"hour": 20, "minute": 0},
    "offPeakLoadBalancingAlgorithm": "DepthFirst"
  }]'

# Assign scaling plan to host pool
az desktopvirtualization scalingplan update \
  --name "sp-weekday" \
  --resource-group "rg-avd" \
  --host-pool-references '[{
    "hostPoolArmPath": "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/hostPools/hp-pooled-prod",
    "scalingPlanEnabled": true
  }]'
```

---

## Image Management with Azure Compute Gallery

```bash
# Create Azure Compute Gallery + image definition
az sig create --gallery-name "avd_gallery" --resource-group "rg-avd" --location "eastus"

az sig image-definition create \
  --gallery-name "avd_gallery" \
  --resource-group "rg-avd" \
  --gallery-image-definition "win11-avd-base" \
  --publisher "MyOrg" \
  --offer "windows-11-avd" \
  --sku "win11-23h2-avd" \
  --os-type Windows \
  --os-state Generalized \
  --hyper-v-generation V2 \
  --features "SecurityType=TrustedLaunch"

# Capture VM as image version (sysprep the VM first!)
az sig image-version create \
  --gallery-name "avd_gallery" \
  --resource-group "rg-avd" \
  --gallery-image-definition "win11-avd-base" \
  --gallery-image-version "1.0.0" \
  --managed-image "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.Compute/images/avd-base-image" \
  --replica-count 1 \
  --replication-regions "eastus" "westus2"

# Use in session host deployment — reference gallery image
# imageReference:
#   id: /subscriptions/SUB/resourceGroups/rg-avd/providers/Microsoft.Compute/galleries/avd_gallery/images/win11-avd-base/versions/latest
```

---

## Entra ID Join (No Domain Controller)

```bash
# Host pool: Entra ID join requirements
# - VM size must support Trusted Launch (Gen2)
# - Session hosts need line-of-sight to Azure AD endpoints (no proxy blocking)
# - Users need Entra ID joined device OR set AllowWebAuthN in RDP client

# After Entra ID join, assign role to allow login
VM_ID="/subscriptions/SUB_ID/resourceGroups/rg-avd-hosts/providers/Microsoft.Compute/virtualMachines/vm-avd-host-01"

# Standard users
az role assignment create \
  --role "Virtual Machine User Login" \
  --assignee-object-id "avd-users-group-oid" \
  --assignee-principal-type Group \
  --scope "$VM_ID"

# Admins (full local admin)
az role assignment create \
  --role "Virtual Machine Administrator Login" \
  --assignee-object-id "avd-admins-group-oid" \
  --assignee-principal-type Group \
  --scope "$VM_ID"
```

---

## Monitoring with AVD Insights

```bash
# Enable diagnostic settings on host pool
az monitor diagnostic-settings create \
  --name "avd-hp-diag" \
  --resource "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/hostpools/hp-pooled-prod" \
  --workspace "/subscriptions/SUB_ID/resourceGroups/rg-mon/providers/Microsoft.OperationalInsights/workspaces/law-avd" \
  --logs '[
    {"category":"Checkpoint","enabled":true},
    {"category":"Error","enabled":true},
    {"category":"Management","enabled":true},
    {"category":"Connection","enabled":true},
    {"category":"HostRegistration","enabled":true},
    {"category":"AgentHealthStatus","enabled":true}
  ]'

# Useful KQL queries
az monitor log-analytics query --workspace "law-avd" --analytics-query "
  WVDConnections
  | where TimeGenerated > ago(1d)
  | summarize SessionCount=count() by UserName, bin(TimeGenerated, 1h)
  | order by TimeGenerated desc
"

az monitor log-analytics query --workspace "law-avd" --analytics-query "
  WVDErrors
  | where TimeGenerated > ago(1d)
  | summarize count() by CodeSymbolic, Message
  | order by count_ desc
"
```

---

## Networking

### RDP Shortpath (UDP transport — lower latency)

```bash
# Enable RDP Shortpath for managed networks (direct UDP)
az desktopvirtualization hostpool update \
  --name "hp-pooled-prod" \
  --resource-group "rg-avd" \
  --custom-rdp-property "audiocapturemode:i:1;enablerdsaadauth:i:1"

# On session host (via GPO or registry):
# HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations
# fUseUdpPortRedirector = 1  (DWORD)
# UdpPortNumber = 3390        (DWORD)
# Open UDP 3390 on NSG inbound from client IPs
```

### Private Link

```bash
# Create Private Link for AVD control plane (workspace + host pool feed)
az network private-endpoint create \
  --name "pe-avd-workspace" \
  --resource-group "rg-avd" \
  --vnet-name "vnet-avd" \
  --subnet "subnet-private-endpoints" \
  --private-connection-resource-id "/subscriptions/SUB_ID/resourceGroups/rg-avd/providers/Microsoft.DesktopVirtualization/workspaces/ws-prod" \
  --connection-name "avd-workspace-pe-conn" \
  --group-ids "feed"

# DNS zone for AVD
az network private-dns zone create -g "rg-avd" -n "privatelink.wvd.microsoft.com"
az network private-dns link vnet create \
  -g "rg-avd" \
  -z "privatelink.wvd.microsoft.com" \
  -n "avd-dns-link" \
  --virtual-network "vnet-avd" \
  --registration-enabled false
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Create pooled host pool | `az desktopvirtualization hostpool create --host-pool-type Pooled` |
| Get registration token | `az desktopvirtualization hostpool retrieve-registration-token` |
| List session hosts | `az desktopvirtualization sessionhost list` |
| Drain host | `az desktopvirtualization sessionhost update --allow-new-session false` |
| Create RemoteApp group | `az desktopvirtualization applicationgroup create --application-group-type RemoteApp` |
| Publish app | `az desktopvirtualization application create` |
| Create workspace | `az desktopvirtualization workspace create` |
| Create scaling plan | `az desktopvirtualization scalingplan create` |
| FSLogix share | Azure Files Premium + Storage File Data SMB Share Contributor RBAC |
