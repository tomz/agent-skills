---
name: azure-costs
description: Azure Cost Management — Cost Analysis, Budgets, Advisor, Reservations, Savings Plans, right-sizing, storage tiers, dev/test pricing, Hybrid Benefit, spot VMs
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

# Azure Cost Management Skills

## Cost Management Basics

```bash
# Get current month spend for a subscription
az consumption usage list \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --query "[].{Service:instanceName, Cost:pretaxCost, Currency:currency}" \
  --output table

# Get budget list
az consumption budget list --output table

# Get spending by resource group (requires Cost Management API)
az rest \
  --method GET \
  --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.CostManagement/query?api-version=2023-11-01" \
  --body '{
    "type": "ActualCost",
    "timeframe": "MonthToDate",
    "dataset": {
      "granularity": "Daily",
      "grouping": [{"type": "Dimension", "name": "ResourceGroup"}],
      "aggregation": {"totalCost": {"name": "PreTaxCost", "function": "Sum"}}
    }
  }'
```

## Budgets & Alerts

```bash
# Create a budget with email alert at 80% and 100%
az consumption budget create \
  --budget-name "MonthlyBudget" \
  --amount 1000 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2026-12-31 \
  --resource-group myRG \
  --notifications '[
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["ops@example.com"],
      "thresholdType": "Actual"
    },
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["ops@example.com", "management@example.com"],
      "thresholdType": "Actual"
    },
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 110,
      "contactEmails": ["ops@example.com"],
      "thresholdType": "Forecasted"
    }
  ]'

# Create subscription-level budget
az consumption budget create \
  --budget-name "SubBudget" \
  --amount 5000 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2026-12-31 \
  --notifications '[{"enabled":true,"operator":"GreaterThan","threshold":90,"contactEmails":["billing@example.com"],"thresholdType":"Actual"}]'

# Budget action groups — trigger automation (e.g., shut down VMs when budget exceeded)
# Configure budget action group in portal: Cost Management > Budgets > Action Groups
```

## Azure Advisor Recommendations

```bash
# List all Advisor recommendations
az advisor recommendation list --output table

# Filter by category (Cost, Security, Performance, Reliability, OperationalExcellence)
az advisor recommendation list \
  --category Cost \
  --output table

# Get details of a specific recommendation
az advisor recommendation show \
  --recommendation-id <recommendation-id>

# Common cost recommendations to act on:
# - Shut down/resize underutilized VMs
# - Purchase reservations for consistent workloads
# - Delete unattached managed disks
# - Remove unused public IP addresses
# - Switch to reserved capacity for SQL databases
# - Delete idle App Service plans

# Dismiss a recommendation (e.g., intentional underutilization)
az advisor recommendation disable \
  --recommendation-id <recommendation-id> \
  --days 90   # snooze for 90 days
```

## Reservations (1-year or 3-year commit)

Reservations provide **up to 72% savings** vs pay-as-you-go for stable workloads.

```bash
# List available reservation offers for VMs
az reservations catalog show \
  --subscription-id <sub-id> \
  --reserved-resource-type VirtualMachines \
  --location eastus \
  --output table

# Purchase a 1-year VM reservation (via REST — CLI has limited support)
az rest \
  --method PUT \
  --url "https://management.azure.com/providers/Microsoft.Capacity/reservationOrders/<order-id>?api-version=2022-11-01" \
  --body '{
    "sku": {"name": "Standard_D4s_v5"},
    "location": "eastus",
    "properties": {
      "reservedResourceType": "VirtualMachines",
      "billingScopeId": "/subscriptions/<sub-id>",
      "term": "P1Y",
      "billingPlan": "Monthly",
      "quantity": 3,
      "appliedScopes": null,
      "appliedScopeType": "Shared"
    }
  }'

# List existing reservations
az reservations reservation list \
  --reservation-order-id <order-id>

# View reservation utilization
az reservations reservation list-history \
  --reservation-id <reservation-id> \
  --reservation-order-id <order-id>
```

### What to Reserve (Priority Order)
```
High priority (very predictable):
  ✅ Production VMs running 24/7 (D/E/F series)
  ✅ Azure SQL Database / Managed Instance (vCore)
  ✅ Azure Cosmos DB (provisioned RU/s, not serverless)
  ✅ Azure Synapse Analytics (dedicated SQL pools)
  ✅ Azure Databricks (DBUs)
  ✅ Redis Cache (Premium)
  ✅ App Service Plans (Premium v3)

Medium priority (fairly predictable):
  ⚠️ AKS node pools (if cluster is always running)
  ⚠️ Azure SQL Elastic Pools

Don't reserve:
  ❌ Dev/test workloads
  ❌ Spot VMs (already discounted)
  ❌ Serverless resources (auto-scale, pay per execution)
  ❌ Resources expected to resize within the reservation term
```

### Reservation Tips
- **Shared scope** (not subscription-scoped): Allows reservation to apply across multiple subscriptions in a billing account. Always use shared scope when possible.
- **Flexibility**: VM reservations have **instance size flexibility** within a family (e.g., D4s_v5 reservation covers D2s_v5 × 2 or D8s_v5 × 0.5).
- **Exchange**: Within 12 months, exchange unused reservations penalty-free for a different SKU or region.
- **Cancellation**: Can cancel reservations (with 12% early termination fee) if circumstances change.

## Azure Savings Plans (Flexible Commitments)

Savings Plans are newer (2022+) and more flexible than Reservations — commitment is by $/hour spend, not specific VM SKU.

```bash
# Savings Plans apply to compute spend across VMs, Container Instances, Container Apps, Azure Functions Premium
# Up to 65% savings vs pay-as-you-go

# Purchase via portal: Cost Management > Savings Plans > Add
# Or via Azure CLI:
az billing savings-plan create \
  --display-name "ComputeSavingsPlan" \
  --billing-scope /subscriptions/<sub-id> \
  --commitment-amount 10 \
  --commitment-currency USD \
  --term P1Y \
  --billing-plan P1M   # P1M = monthly payments; P1Y = upfront
```

### Reservations vs Savings Plans
```
Reservations:
  + Higher discount (up to 72%)
  - Locked to specific SKU/region/service
  - Best for known, stable workloads

Savings Plans:
  + Flexible — applies across any compute service
  + Auto-applies to highest hourly savings
  - Lower discount than reservation for same workload
  - Best for diverse or evolving compute footprint

Strategy: Use Reservations for stable, specific workloads + Savings Plans for variable/unknown remainder
```

## Right-Sizing VMs

```bash
# Advisor shows underutilized VMs — also check manually:

# Get average CPU for a VM over 30 days
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --metric "Percentage CPU" \
  --aggregation Average \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z \
  --interval PT1H \
  --query "value[0].timeseries[0].data[*].average" \
  --output tsv | awk '{sum+=$1; count++} END {print "Avg CPU:", sum/count "%"}'

# Get memory usage (requires Azure Monitor agent + DCR)
az monitor metrics list \
  --resource <vm-resource-id> \
  --metric "Available Memory Bytes" \
  --aggregation Average \
  --output table

# VM size families and typical downsizing paths:
# Standard_D8s_v5 (8 cores) → Standard_D4s_v5 (4 cores): 50% compute cost reduction
# Standard_E16s_v5 → Standard_E8s_v5: 50% reduction
# Check: is P95 CPU < 40%? Consider halving cores
# Check: is avg memory < 30%? Consider downsizing memory tier
```

### VM Right-Sizing Checklist
```
1. Collect 30-day P95 CPU utilization (not average — use P95 to avoid spikes)
2. Collect P95 memory utilization
3. Check network throughput vs VM NIC limit
4. Check disk IOPS vs disk limit
5. If all metrics < 40% of limit → candidate for downsizing
6. Test on a non-production copy first
7. Schedule resize during maintenance window (VM restart required)
```

## Storage Tier Optimization

```bash
# Analyze storage account by access tier
az storage account show \
  --name mystorageacct \
  --resource-group myRG \
  --query "properties.accessTier" -o tsv

# Get blob counts by tier (PowerShell/Storage Analytics — not CLI)
# Use Storage Insights Workbook in Azure Portal for visual breakdown

# Move specific blobs to Cool tier
az storage blob set-tier \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.log \
  --tier Cool \
  --auth-mode login

# Rehydrate from Archive (can take hours)
az storage blob set-tier \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name archived-file.zip \
  --tier Hot \
  --rehydrate-priority High \  # High (1h) or Standard (15h)
  --auth-mode login

# Blob inventory report (understand what's in storage)
az storage account blob-inventory-policy create \
  --account-name mystorageacct \
  --resource-group myRG \
  --policy @inventory-policy.json
```

### Storage Cost Levers
```
Action                                    Savings Estimate
────────────────────────────────────────────────────────
Move blobs >30 days old to Cool tier     ~40% storage cost
Move blobs >90 days old to Archive       ~90% storage cost
Delete snapshots older than 90 days      100% of snapshot cost
Use LRS instead of GRS (dev/test)        ~50% reduction
Use Standard_LRS instead of Premium      ~80% reduction
Enable lifecycle management policy       Automates tier transitions
Delete orphaned disks                    100% of disk cost
Reduce disk size class (P30→P20)         ~50% disk cost
```

## Dev/Test Pricing

```bash
# Dev/Test subscriptions: Visual Studio subscribers get significant discounts
# Available through Enterprise Agreement or PAYG with VS subscriptions

# Dev/Test pricing discounts (approximate):
# Windows VMs:        up to 80% (no Windows license cost)
# SQL Database:       40% discount on DTU/vCore
# Logic Apps:        40% discount
# App Service:       60% discount on Standard/Premium plans
# Azure DevOps:      Included for VS subscribers

# Apply dev/test pricing: set during subscription creation
# Can't convert existing production subscription to dev/test

# Azure Dev/Test Labs: self-service lab environments with cost controls
az lab create \
  --resource-group myRG \
  --name myDevLab \
  --location eastus

# Auto-shutdown policy (stop VMs at night)
az lab policy set \
  --lab-name myDevLab \
  --resource-group myRG \
  --policy-set default \
  --name LabVmsShutdown \
  --status Enabled \
  --fact-data '{"time":"1900","timeZoneId":"UTC"}'
```

## Azure Hybrid Benefit

Azure Hybrid Benefit lets you reuse existing on-premises Windows Server and SQL Server licenses with Software Assurance.

```bash
# Enable Hybrid Benefit on a VM (Windows)
az vm update \
  --resource-group myRG \
  --name myWindowsVM \
  --license-type Windows_Server
# Savings: eliminates Windows license cost (~40% of Windows VM cost)

# Enable on SQL IaaS VM
az vm update \
  --resource-group myRG \
  --name mySQLVM \
  --license-type SQL_Server_Enterprise   # or SQL_Server_Standard

# Enable Hybrid Benefit on Azure SQL Database (PaaS)
az sql db update \
  --resource-group myRG \
  --server myserver \
  --name mydb \
  --license-type BasePrice   # BasePrice = Hybrid Benefit; LicenseIncluded = regular pricing
# Savings: ~30% on vCore model databases

# Enable on AKS (Windows node pools)
az aks nodepool update \
  --resource-group myRG \
  --cluster-name myAKS \
  --name winnodepool \
  --enable-ahub

# Check how many Hybrid Benefit eligible VMs you have
az vm list \
  --query "[?licenseType=='Windows_Server']" \
  --output table
```

## Spot VMs (Up to 90% Discount)

```bash
# Create spot VM
az vm create \
  --resource-group myRG \
  --name mySpotVM \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1   # -1 = pay up to current on-demand price; or set a max $ price

# Create spot VMSS (for batch workloads)
az vmss create \
  --resource-group myRG \
  --name mySpotVMSS \
  --image Ubuntu2204 \
  --vm-sku Standard_D4s_v5 \
  --priority Spot \
  --eviction-policy Delete \
  --instance-count 5

# Check current spot price
az vm list-skus \
  --location eastus \
  --size Standard_D4s_v5 \
  --query "[].{Name:name, MaxPrice:restrictions}" \
  --output table

# AKS spot node pool (automatically handles evictions)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKS \
  --name spotnodes \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 50 \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule \
  --labels kubernetes.azure.com/scalesetpriority=spot
```

### Spot VM Best Practices
```
✅ Use for: batch jobs, CI/CD agents, dev/test, stateless workloads, ML training
❌ Avoid for: databases, stateful apps, anything with long session state
✅ Always handle eviction: use checkpoint/restart pattern
✅ Use multiple VM sizes: if D4s_v5 spot capacity is low, D4s_v4 or D2s_v5 may be available
✅ Use eviction-policy=Deallocate for jobs that can resume; Delete for disposable workers
✅ Diversify across AZs to reduce simultaneous eviction
```

## Cost Analysis Patterns

### Tags for Cost Allocation
```bash
# Tag all resources in a resource group
az tag update \
  --resource-id /subscriptions/<sub>/resourceGroups/myRG \
  --operation merge \
  --tags CostCenter=12345 Project=Phoenix Owner=TeamA Environment=Production

# Policy to enforce tags (prevent untagged resources)
# Built-in policy: "Require a tag and its value on resources"
az policy assignment create \
  --name RequireCostCenterTag \
  --policy /providers/Microsoft.Authorization/policyDefinitions/<policy-id> \
  --params '{"tagName":{"value":"CostCenter"}}'

# Export cost data to storage for analysis
az costmanagement export create \
  --name MonthlyExport \
  --type ActualCost \
  --scope /subscriptions/<sub> \
  --storage-account-id <storage-account-resource-id> \
  --storage-container costs \
  --storage-directory exports \
  --schedule-recurrence Monthly \
  --recurrence-period from="2024-01-01" to="2026-12-31" \
  --timeframe MonthToDate
```

### Quick Cost Wins Checklist
```
Week 1 — Quick wins (no risk):
  □ Delete unattached managed disks
  □ Delete unused public IP addresses (billed even when unattached in Standard SKU)
  □ Delete unused load balancers
  □ Stop/deallocate dev/test VMs outside business hours
  □ Delete old disk snapshots (> 90 days)
  □ Delete unused App Service slots (staging, etc.)

Month 1 — Medium effort:
  □ Right-size VMs with < 40% P95 CPU utilization
  □ Enable lifecycle management on Storage Accounts (move to Cool/Archive)
  □ Set up budget alerts at 80%/100%/110% (forecasted)
  □ Enable Azure Hybrid Benefit on eligible VMs and SQL
  □ Enable auto-shutdown on dev/test VMs (7 PM daily)
  □ Switch dev storage from GRS to LRS

Month 2-3 — Strategic:
  □ Purchase Reservations for top 5 VM SKUs running 24/7
  □ Purchase Azure Savings Plan for flexible compute
  □ Analyze Cosmos DB RU/s — switch over-provisioned containers to autoscale
  □ Evaluate Serverless SQL vs provisioned for dev databases
  □ Consider Spot VMs for batch/CI workloads
  □ Set up tag policy enforcement + cost allocation by team
```

## Gotchas

- **Public IPs cost money when unattached** (Standard SKU): ~$3.65/month per unattached Standard IP. Free tier public IPs were retired — all new IPs are Standard.
- **Stopped VMs vs deallocated VMs**: `az vm stop` keeps the VM allocated — still billed for compute. Only `az vm deallocate` stops compute billing. OS disk storage is still billed.
- **Managed disks are billed by provisioned size**, not used size. A 1TB P70 disk costs ~$135/month whether it has 10MB or 900GB of data.
- **Reservation utilization**: Unused reservation capacity is wasted money. Monitor utilization in Cost Management > Reservations > Utilization.
- **Hybrid Benefit requires Software Assurance**: You must have an active SA agreement. Don't apply without verifying license eligibility — Microsoft audits.
- **Spot VM eviction is sudden**: Spot VMs get a 30-second warning before eviction (via Azure Metadata Service). Design accordingly.
- **Budget alerts are not real-time**: Budget alerts can lag 8-24 hours behind actual spend. Don't rely on them for hard limits.
- **Cost Management data latency**: Billing data for the current day may take 8-24 hours to appear in Cost Management.
- **Dev/Test subscription limits**: Can't run production workloads on Dev/Test subscriptions — violates Microsoft licensing terms.
- **Savings Plans scope matters**: Subscription-scoped Savings Plans only apply to that subscription. Management-group scoped plans cover all child subscriptions (better for large orgs).
