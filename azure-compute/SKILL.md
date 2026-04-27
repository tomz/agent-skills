---
name: azure-compute
description: Azure compute — VMs, VMSS, App Service, Container Apps, Functions, AKS, ACI — sizing, availability, deployment, scaling patterns
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

# Azure Compute Skills

## Virtual Machines

### Creating VMs
```bash
# Linux VM with SSH key
az vm create \
  --resource-group myRG \
  --name myLinuxVM \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name myVNet \
  --subnet WebSubnet \
  --public-ip-sku Standard \
  --no-wait

# Windows VM with password
az vm create \
  --resource-group myRG \
  --name myWinVM \
  --image Win2022Datacenter \
  --size Standard_D4s_v5 \
  --admin-username adminuser \
  --admin-password "P@ssw0rd123!" \
  --no-public-ip-address   # no public IP — access via Bastion

# Spot VM (significantly cheaper, can be evicted)
az vm create \
  --resource-group myRG \
  --name mySpotVM \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1   # -1 = pay up to on-demand price
```

### VM Common Operations
```bash
# Start / stop / restart / deallocate
az vm start -g myRG -n myVM
az vm stop -g myRG -n myVM       # stops OS but still bills for compute
az vm deallocate -g myRG -n myVM  # stops billing for compute
az vm restart -g myRG -n myVM

# Power state
az vm get-instance-view -g myRG -n myVM \
  --query "instanceView.statuses[1].displayStatus" -o tsv

# Resize a VM
az vm resize -g myRG -n myVM --size Standard_D8s_v5

# List available sizes in a region
az vm list-sizes --location eastus --output table | grep Standard_D

# Run a command on a VM (via VM agent — no SSH needed)
az vm run-command invoke \
  -g myRG -n myVM \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y nginx"

# Create a snapshot of OS disk
DISK_ID=$(az vm show -g myRG -n myVM --query "storageProfile.osDisk.managedDisk.id" -o tsv)
az snapshot create -g myRG -n mySnapshot --source "$DISK_ID"

# Open a port (creates NSG rule)
az vm open-port -g myRG -n myVM --port 443 --priority 900
```

### VM Availability

```bash
# Create Availability Set (max 2 fault domains for most regions)
az vm availability-set create \
  --name myAvSet \
  --resource-group myRG \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5

az vm create -g myRG -n myVM1 --availability-set myAvSet ...
az vm create -g myRG -n myVM2 --availability-set myAvSet ...

# Availability Zones (preferred — uses separate physical infrastructure)
az vm create -g myRG -n myVM-zone1 --zone 1 ...
az vm create -g myRG -n myVM-zone2 --zone 2 ...
az vm create -g myRG -n myVM-zone3 --zone 3 ...

# Check zone support for a VM size
az vm list-skus \
  --location eastus \
  --size Standard_D4s_v5 \
  --query "[].{Name:name, Zones:locationInfo[0].zones}" \
  --output table
```

### VM Size Families (Quick Reference)
```
B-series    → Burstable; cheap for dev/test (Standard_B2s, B4ms)
D-series    → General purpose; balanced CPU/memory (Standard_D4s_v5)
E-series    → Memory-optimized; databases, caches (Standard_E8s_v5)
F-series    → Compute-optimized; CPU-heavy workloads (Standard_F8s_v2)
L-series    → Storage-optimized; NVMe local SSDs (Standard_L8s_v3)
M-series    → Massive memory; up to 11.4 TB RAM (Standard_M128ms)
N-series    → GPU; AI/ML training (Standard_NC6s_v3, NV-series for viz)
HB/HC       → HPC; InfiniBand interconnect
```

## Virtual Machine Scale Sets (VMSS)

```bash
# Create VMSS with autoscale
az vmss create \
  --resource-group myRG \
  --name myVMSS \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 2 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name myVNet \
  --subnet AppSubnet \
  --upgrade-policy-mode Automatic \
  --load-balancer myLB \
  --backend-pool-name BackendPool

# Configure autoscale
az monitor autoscale create \
  --resource-group myRG \
  --resource myVMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name myAutoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Add scale-out rule (CPU > 70% for 5 min → add 2 instances)
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myAutoscale \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Add scale-in rule
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myAutoscale \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1

# Manually scale
az vmss scale --resource-group myRG --name myVMSS --new-capacity 5

# Rolling upgrade
az vmss update-instances --resource-group myRG --name myVMSS --instance-ids '*'
```

## App Service

```bash
# Create App Service Plan
az appservice plan create \
  --name myPlan \
  --resource-group myRG \
  --sku P1v3 \
  --is-linux

# Create Web App
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name myapp-uniquename \
  --runtime "NODE:20-lts"

# Deploy from GitHub (CI/CD)
az webapp deployment source config \
  --name myapp-uniquename \
  --resource-group myRG \
  --repo-url https://github.com/myorg/myrepo \
  --branch main \
  --manual-integration

# Deploy from local ZIP
az webapp deploy \
  --resource-group myRG \
  --name myapp-uniquename \
  --src-path ./build.zip \
  --type zip

# Configure app settings
az webapp config appsettings set \
  --resource-group myRG \
  --name myapp-uniquename \
  --settings NODE_ENV=production DB_HOST=mydb.database.windows.net

# Stream logs
az webapp log tail --resource-group myRG --name myapp-uniquename

# Create deployment slot
az webapp deployment slot create \
  --resource-group myRG \
  --name myapp-uniquename \
  --slot staging

# Swap slots (blue-green deployment)
az webapp deployment slot swap \
  --resource-group myRG \
  --name myapp-uniquename \
  --slot staging \
  --target-slot production

# Scale out (App Service Plan)
az appservice plan update \
  --name myPlan \
  --resource-group myRG \
  --number-of-workers 3
```

### App Service SKUs
```
F1 / D1    → Free / Shared (no SLA, no custom domain)
B1/B2/B3   → Basic — no autoscale, no slots; dev/test
S1/S2/S3   → Standard — autoscale, 5 slots, custom domains + SSL
P1v3/P2v3/P3v3 → Premium v3 — faster compute, VNET integration, 20 slots
P0v3       → Budget Premium — cheapest VNet integration option
EP1/EP2/EP3 → Elastic Premium (for Functions)
I1v2/I2v2  → Isolated v2 (App Service Environment — single tenant)
```

## Azure Container Apps

```bash
# Create Container Apps environment (with VNet injection)
az containerapp env create \
  --name myEnv \
  --resource-group myRG \
  --location eastus \
  --infrastructure-subnet-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/ContainerAppsSubnet

# Deploy a container app
az containerapp create \
  --name myapp \
  --resource-group myRG \
  --environment myEnv \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1.0Gi

# Scale rules — HTTP-based autoscale
az containerapp update \
  --name myapp \
  --resource-group myRG \
  --scale-rule-name http-scale \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50

# Update container image (triggers rolling deployment)
az containerapp update \
  --name myapp \
  --resource-group myRG \
  --image myregistry.azurecr.io/myapp:v2.0

# Stream logs
az containerapp logs show \
  --name myapp \
  --resource-group myRG \
  --follow

# Create a job (one-shot or scheduled)
az containerapp job create \
  --name myjob \
  --resource-group myRG \
  --environment myEnv \
  --trigger-type Schedule \
  --cron-expression "0 6 * * *" \
  --image myregistry.azurecr.io/myjob:latest \
  --cpu 1 \
  --memory 2Gi
```

## Azure Functions

```bash
# Create Function App — Consumption plan (Linux)
az functionapp create \
  --resource-group myRG \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name myfuncapp-unique \
  --storage-account mystorageacct \
  --os-type linux

# Create Function App — Elastic Premium (VNet, no cold start)
az functionapp create \
  --resource-group myRG \
  --plan myPremiumPlan \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name myfuncapp-premium \
  --storage-account mystorageacct

# Deploy via ZIP
az functionapp deployment source config-zip \
  --resource-group myRG \
  --name myfuncapp-unique \
  --src ./function.zip

# Configure settings
az functionapp config appsettings set \
  --name myfuncapp-unique \
  --resource-group myRG \
  --settings AzureWebJobsStorage="<conn-string>" FUNCTIONS_WORKER_RUNTIME=python

# List functions
az functionapp function list \
  --name myfuncapp-unique \
  --resource-group myRG \
  --output table
```

### Function Plans Comparison
```
Consumption   → Pay per execution; cold starts; max 10min timeout; no VNet (without Premium)
Flex          → New in 2024; faster scale, VNet support, per-execution billing
Elastic Prem  → Always warm; VNet; max 60min; higher base cost
Dedicated     → App Service Plan; predictable cost; max timeout unlimited
Container Apps → Full container control; KEDA-based scaling
```

## AKS (Azure Kubernetes Service)

```bash
# Create AKS cluster
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --network-plugin azure \
  --network-policy azure \
  --vnet-subnet-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/AKSSubnet \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --zones 1 2 3 \
  --kubernetes-version 1.29 \
  --tier standard \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myRG --name myAKS --overwrite-existing

# Add a node pool (e.g., GPU or spot nodes)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name gpupool \
  --node-vm-size Standard_NC6s_v3 \
  --node-count 1 \
  --node-taints sku=gpu:NoSchedule \
  --labels accelerator=nvidia

# Add a spot node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name spotnodes \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule

# Upgrade cluster
az aks get-upgrades --resource-group myRG --name myAKS --output table
az aks upgrade --resource-group myRG --name myAKS --kubernetes-version 1.30

# Enable addons
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons monitoring \
  --workspace-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLAW

# AKS RBAC — assign cluster role to a user
az role assignment create \
  --assignee user@example.com \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ContainerService/managedClusters/myAKS
```

### AKS Networking Modes
```
Kubenet          → Simple overlay; nodes get VNet IPs, pods get NAT'd RFC1918 IPs (limited scale)
Azure CNI        → Pods get VNet IPs; requires more IP space; full VNet visibility
Azure CNI Overlay → Pods get private overlay IPs; nodes in VNet; best of both (recommended 2024+)
Azure CNI Dynamic IP → Efficient IP allocation for Azure CNI
```

## Azure Container Instances (ACI)

```bash
# Quick single container deployment (no orchestration overhead)
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image nginx:alpine \
  --cpu 1 \
  --memory 1.5 \
  --ports 80 \
  --ip-address Public \
  --dns-name-label mycontainer-uniquelabel

# Container with environment variables and secrets
az container create \
  --resource-group myRG \
  --name myapp \
  --image myregistry.azurecr.io/myapp:latest \
  --registry-login-server myregistry.azurecr.io \
  --registry-username <acr-user> \
  --registry-password <acr-password> \
  --environment-variables APP_ENV=production \
  --secure-environment-variables DB_PASSWORD=s3cr3t \
  --cpu 2 \
  --memory 4

# Logs
az container logs --resource-group myRG --name mycontainer --follow

# Exec into container
az container exec --resource-group myRG --name mycontainer --exec-command "/bin/sh"

# Container group with VNet injection (no public IP)
az container create \
  --resource-group myRG \
  --name myprivateci \
  --image myapp:latest \
  --vnet myVNet \
  --subnet AciSubnet \
  --ip-address Private
```

## Gotchas

- **VM stop ≠ deallocate**: `az vm stop` keeps the VM allocated (still billed). Use `az vm deallocate` to stop billing.
- **VMSS Uniform vs Flexible**: Flexible mode is preferred for new deployments — supports mixing instance types and AZs.
- **App Service VNET integration**: Requires at least Basic plan for regional VNet integration; requires Premium for full outbound VNet routing (`WEBSITE_VNET_ROUTE_ALL=1`).
- **Functions cold starts**: Consumption plan can take 5-15s to cold start. Use Flex or Elastic Premium for latency-sensitive workloads.
- **AKS node pool OS disk**: Default is Ephemeral OS disk (fast, local SSD) — always prefer this when VM supports it. Managed disk is slower.
- **AKS workload identity** (preferred over pod identity): Uses OIDC + federated credentials. Never mount service principal secrets in pods.
- **Container Apps min-replicas=0**: Apps scale to zero when idle (save cost), but first request is slow. Set min-replicas=1 for latency-sensitive apps.
- **ACI is not for production HA**: No autoscale, no persistent storage across restarts (without Azure Files mount). Use Container Apps or AKS for production.
- **Availability Zones ≠ Availability Sets**: AZs provide hardware isolation across datacenters; AS provides fault/update domain separation within one datacenter. Prefer AZs.
