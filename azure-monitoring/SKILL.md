---
name: azure-monitoring
description: Azure Monitor, Log Analytics, KQL query patterns, Application Insights, Alerts, Diagnostic Settings, Workbooks, Azure Monitor Agent, metric namespaces
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

# Azure Monitoring Skills

## Azure Monitor Overview

```
Azure Monitor
├── Metrics (numeric time-series, 93-day retention, near real-time)
│   ├── Platform metrics (auto-collected from Azure resources)
│   └── Custom metrics (from SDK, agent, or REST API)
├── Logs (structured/semi-structured, stored in Log Analytics workspace)
│   ├── Resource logs (diagnostic settings → workspace)
│   ├── Activity Log (subscription-level events)
│   └── Custom logs (agent, API ingestion)
├── Application Insights (APM for apps — traces, requests, dependencies)
└── Alerts
    ├── Metric alerts (fast, ~1-min eval)
    ├── Log alerts (KQL queries, 5-min+ eval)
    └── Activity log alerts (subscription events)
```

## Log Analytics Workspace

```bash
# Create workspace
az monitor log-analytics workspace create \
  --resource-group myRG \
  --workspace-name myLAW \
  --location eastus \
  --sku PerGB2018 \
  --retention-time 90     # days; 30-730 range; 90 is default

# Get workspace ID (needed for diagnostic settings)
az monitor log-analytics workspace show \
  --resource-group myRG \
  --workspace-name myLAW \
  --query customerId -o tsv

# Get workspace resource ID
az monitor log-analytics workspace show \
  --resource-group myRG \
  --workspace-name myLAW \
  --query id -o tsv

# Set data retention per table (override workspace default)
az monitor log-analytics workspace table update \
  --resource-group myRG \
  --workspace-name myLAW \
  --name SecurityEvent \
  --retention-time 365

# Query workspace via CLI
az monitor log-analytics query \
  --workspace myLAW-workspace-id \
  --analytics-query "Heartbeat | summarize LastSeen=max(TimeGenerated) by Computer | top 10 by LastSeen desc" \
  --output table
```

## Diagnostic Settings (Route Logs & Metrics to Workspace)

```bash
# Enable diagnostic settings on a VM (sends to Log Analytics)
az monitor diagnostic-settings create \
  --name "SendToLAW" \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLAW \
  --logs '[{"category":"Administrative","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Diagnostic settings for Azure SQL Database
az monitor diagnostic-settings create \
  --name "SQLDiag" \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Sql/servers/myserver/databases/mydb \
  --workspace <workspace-resource-id> \
  --logs '[
    {"category":"SQLInsights","enabled":true},
    {"category":"QueryStoreRuntimeStatistics","enabled":true},
    {"category":"Errors","enabled":true},
    {"category":"Deadlocks","enabled":true}
  ]' \
  --metrics '[{"category":"Basic","enabled":true}]'

# Diagnostic settings for Key Vault (audit all access)
az monitor diagnostic-settings create \
  --name "KVAudit" \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV \
  --workspace <workspace-resource-id> \
  --logs '[{"category":"AuditEvent","enabled":true}]'

# Activity log → workspace (subscription level)
az monitor diagnostic-settings create \
  --name "ActivityToLAW" \
  --resource /subscriptions/<sub-id> \
  --workspace <workspace-resource-id> \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Security","enabled":true},{"category":"Alert","enabled":true},{"category":"Policy","enabled":true}]'

# List diagnostic settings on a resource
az monitor diagnostic-settings list \
  --resource <resource-id> \
  --output table
```

## KQL (Kusto Query Language) — Essential Patterns

KQL is used in Log Analytics, Application Insights, Sentinel, and Azure Data Explorer.

### Basic Query Structure
```kql
TableName
| where TimeGenerated > ago(1h)
| where Computer contains "web"
| project TimeGenerated, Computer, EventID, RenderedDescription
| order by TimeGenerated desc
| take 100
```

### Time Filters
```kql
// Relative time
| where TimeGenerated > ago(24h)
| where TimeGenerated > ago(7d)

// Absolute time
| where TimeGenerated between (datetime(2024-01-01) .. datetime(2024-01-31))

// Bin (bucket) by time
| summarize count() by bin(TimeGenerated, 5m)
```

### String Operations
```kql
| where Message contains "error"              // case-insensitive
| where Message has "connection refused"      // has is faster than contains
| where Message matches regex @"error\s+\d+"  // regex
| where Computer startswith "prod-"
| where Computer in ("web-01", "web-02", "web-03")
| where Computer !in ("legacy-01")
| extend AppName = extract(@"app=(\w+)", 1, Message)  // extract with regex
```

### Aggregation
```kql
// Count by category
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU=avg(CounterValue), MaxCPU=max(CounterValue) by Computer, bin(TimeGenerated, 1h)
| order by AvgCPU desc

// Count errors per hour
Event
| where EventLevelName == "Error"
| summarize ErrorCount=count() by Computer, bin(TimeGenerated, 1h)
| render timechart

// Top N
requests  // Application Insights table
| summarize Count=count() by name
| top 10 by Count desc
```

### Joins
```kql
// Join two tables
SecurityEvent
| where EventID == 4625  // failed logon
| join kind=leftouter (
    SigninLogs
    | project UserPrincipalName, IPAddress
) on $left.TargetUserName == $right.UserPrincipalName
| project TimeGenerated, Computer, TargetUserName, IpAddress, IPAddress
```

### Useful Tables & Their Contents
```
Heartbeat          → Agent check-in (is VM connected to Log Analytics?)
Perf               → Performance counters (CPU, memory, disk)
Event              → Windows Event Log
Syslog             → Linux syslog
SecurityEvent      → Windows security events (4624=login, 4625=failed login, 4688=process)
AzureActivity      → Azure subscription-level operations
AzureDiagnostics   → Resource logs (multi-resource, polymorphic schema)
AzureMetrics       → Platform metrics sent to workspace
ContainerLog       → Container stdout/stderr (AKS)
KubePodInventory   → AKS pod inventory
requests           → App Insights HTTP requests
exceptions         → App Insights exceptions
traces             → App Insights custom traces / console logs
dependencies       → App Insights outbound calls (SQL, HTTP, etc.)
customEvents       → App Insights custom events
availabilityResults → App Insights availability test results
```

### Common KQL Queries
```kql
// VM CPU usage — last 1 hour
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where InstanceName == "_Total"
| summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart

// Failed logins — last 24h
SecurityEvent
| where EventID == 4625
| summarize FailedCount=count() by TargetUserName, IpAddress, Computer
| order by FailedCount desc

// App Insights — slow requests (> 2s)
requests
| where duration > 2000
| summarize SlowCount=count(), AvgDuration=avg(duration) by name
| order by SlowCount desc

// App Insights — error rate by operation
requests
| summarize Total=count(), Errors=countif(success==false) by name
| extend ErrorRate = round(todouble(Errors)/Total*100, 2)
| order by ErrorRate desc

// AKS pod restarts
KubePodInventory
| where PodRestartCount > 0
| summarize Restarts=max(PodRestartCount) by Name, Namespace, Computer
| order by Restarts desc

// Azure Activity — who deleted what
AzureActivity
| where OperationNameValue endswith "DELETE"
| where ActivityStatusValue == "Success"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, _ResourceId
| order by TimeGenerated desc

// Key Vault secret access audit
AzureDiagnostics
| where ResourceType == "VAULTS"
| where OperationName == "SecretGet"
| project TimeGenerated, identity_claim_unique_name_s, id_s, httpStatusCode_d
| order by TimeGenerated desc

// Container log errors
ContainerLog
| where LogEntry contains "ERROR" or LogEntry contains "Exception"
| project TimeGenerated, ContainerID, LogEntry
| order by TimeGenerated desc
```

## Application Insights

```bash
# Create Application Insights (workspace-based — required for new deployments)
az monitor app-insights component create \
  --app myAppInsights \
  --resource-group myRG \
  --location eastus \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLAW \
  --kind web \
  --application-type web

# Get instrumentation key and connection string
az monitor app-insights component show \
  --app myAppInsights \
  --resource-group myRG \
  --query "{ConnectionString:connectionString, InstrumentationKey:instrumentationKey}" \
  -o json

# Query App Insights via CLI
az monitor app-insights query \
  --app myAppInsights \
  --resource-group myRG \
  --analytics-query "requests | summarize count() by resultCode | order by count_ desc" \
  --output table

# Configure availability test (ping test)
az monitor app-insights web-test create \
  --resource-group myRG \
  --name myAvailTest \
  --app myAppInsights \
  --location eastus \
  --defined-web-test-kind ping \
  --request-url https://myapp.azurewebsites.net/health \
  --frequency 300 \
  --timeout 30 \
  --retry-enabled true
```

### App Insights SDK Integration (Python)
```python
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace

configure_azure_monitor(
    connection_string="InstrumentationKey=xxx;IngestionEndpoint=..."
)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("my-operation") as span:
    span.set_attribute("user.id", "12345")
    # do work
```

## Alerts

### Metric Alerts
```bash
# Alert when CPU > 80% for 5 minutes
az monitor metrics alert create \
  --name "HighCPUAlert" \
  --resource-group myRG \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action-group /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myActionGroup \
  --description "CPU above 80% for 5 minutes"

# Storage account — throttled requests alert
az monitor metrics alert create \
  --name "StorageThrottling" \
  --resource-group myRG \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --condition "total Transactions where ResponseType includes 'ThrottlingError' > 100" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action-group <action-group-id>
```

### Log Alerts (KQL-based)
```bash
# Alert when >5 failed logins in 10 minutes
az monitor scheduled-query create \
  --resource-group myRG \
  --name "FailedLoginsAlert" \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLAW \
  --condition "count > 5" \
  --condition-query "SecurityEvent | where EventID == 4625 | summarize count()" \
  --evaluation-frequency 5m \
  --window-duration 10m \
  --severity 2 \
  --action-group <action-group-id>
```

### Action Groups
```bash
# Create action group with email + webhook
az monitor action-group create \
  --resource-group myRG \
  --name myActionGroup \
  --short-name myAG \
  --action email ops-team ops@example.com \
  --action webhook mywebhook https://hooks.example.com/alert

# Add SMS
az monitor action-group update \
  --resource-group myRG \
  --name myActionGroup \
  --add-action sms phone1 1 2025551234

# PagerDuty integration (via webhook)
# Teams/Slack: use Logic App or Azure Function as webhook target
```

### Activity Log Alerts
```bash
# Alert on any resource deletion in a resource group
az monitor activity-log alert create \
  --resource-group myRG \
  --name "ResourceDeletionAlert" \
  --scope /subscriptions/<sub>/resourceGroups/myRG \
  --condition "category=Administrative and operationName contains Microsoft.Resources/delete" \
  --action-group <action-group-id>

# Alert on service health events
az monitor activity-log alert create \
  --resource-group myRG \
  --name "ServiceHealthAlert" \
  --scope /subscriptions/<sub> \
  --condition "category=ServiceHealth" \
  --action-group <action-group-id>
```

## Azure Monitor Agent (AMA)

AMA replaces the deprecated Log Analytics Agent (MMA/OMS) and Dependency Agent.

```bash
# Install AMA via VM extension (Linux)
az vm extension set \
  --resource-group myRG \
  --vm-name myVM \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 \
  --enable-auto-upgrade true

# Install AMA on Windows VM
az vm extension set \
  --resource-group myRG \
  --vm-name myWinVM \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 \
  --enable-auto-upgrade true

# Create Data Collection Rule (DCR) — defines what data to collect
az monitor data-collection rule create \
  --resource-group myRG \
  --name myDCR \
  --location eastus \
  --rule-file dcr.json

# Associate DCR with a VM
az monitor data-collection rule association create \
  --resource-group myRG \
  --rule-name myDCR \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --association-name myDCRAssociation
```

### Sample DCR JSON
```json
{
  "location": "eastus",
  "properties": {
    "dataSources": {
      "performanceCounters": [{
        "streams": ["Microsoft-Perf"],
        "samplingFrequencyInSeconds": 60,
        "counterSpecifiers": [
          "\\Processor Information(_Total)\\% Processor Time",
          "\\Memory\\Available Bytes",
          "\\LogicalDisk(_Total)\\Disk Reads/sec"
        ],
        "name": "perfCounterDataSource"
      }],
      "syslog": [{
        "streams": ["Microsoft-Syslog"],
        "facilityNames": ["kern", "daemon", "auth"],
        "logLevels": ["Warning", "Error", "Critical"],
        "name": "syslogDataSource"
      }]
    },
    "destinations": {
      "logAnalytics": [{
        "workspaceResourceId": "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLAW",
        "name": "myLAW"
      }]
    },
    "dataFlows": [
      { "streams": ["Microsoft-Perf"], "destinations": ["myLAW"] },
      { "streams": ["Microsoft-Syslog"], "destinations": ["myLAW"] }
    ]
  }
}
```

## Metric Namespaces (Common)

```
Microsoft.Compute/virtualMachines    → Percentage CPU, Available Memory Bytes, Disk Read/Write Bytes
Microsoft.Storage/storageAccounts    → Transactions, Availability, UsedCapacity
Microsoft.Sql/servers/databases      → cpu_percent, dtu_consumption_percent, deadlock
Microsoft.Web/sites                  → CpuTime, Requests, BytesSent, Http5xx, ResponseTime
Microsoft.ContainerService/managedClusters → node_cpu_usage_percentage, node_memory_working_set_percentage
Microsoft.Cache/Redis                → cache_hits, cache_misses, used_memory, connected_clients
Microsoft.ServiceBus/namespaces      → IncomingMessages, OutgoingMessages, ActiveMessages
Microsoft.KeyVault/vaults            → ServiceApiHit, ServiceApiLatency, Availability
```

## Workbooks & Dashboards

```bash
# List workbooks
az monitor app-insights workbook list \
  --resource-group myRG \
  --output table

# Create workbook from template (via ARM/Bicep — no CLI support for complex workbooks)
# Recommended approach: use Azure portal to design, then export ARM template

# Pin a chart to dashboard via portal:
# Log Analytics query → New alert rule / Pin to dashboard

# Create dashboard via CLI
az portal dashboard create \
  --resource-group myRG \
  --name myDashboard \
  --input-path dashboard.json \
  --location eastus
```

## Gotchas

- **Diagnostic settings are not retroactive**: Enabling them only captures data from that point forward.
- **`AzureDiagnostics` table schema**: Multi-resource table — columns are resource-type specific and use dynamic naming (`category_s`, `operationName_s`). Always filter by `ResourceType` first.
- **Log Analytics ingestion delay**: Data can take 2-5 minutes to appear. Metric alerts fire faster (< 1 min) than log alerts (5 min+).
- **KQL `has` vs `contains`**: `has` checks whole tokens (faster, uses index); `contains` is substring search (slower). Prefer `has` for known terms.
- **Workspace retention cost**: Data older than 31 days (Basic) or 90 days (Analytics tier) incurs additional archival charges.
- **Application Insights classic vs workspace-based**: Classic is deprecated. Always create workspace-based (linked to Log Analytics workspace).
- **AMA vs MMA**: Log Analytics Agent (MMA) is deprecated since August 2024. Migrate to Azure Monitor Agent + DCRs.
- **Action group limits**: Max 10 action groups per alert rule. Each group can have multiple receivers (email, SMS, webhook, ITSM, Logic App, etc.).
- **Alert evaluation window**: Log alerts have minimum 5-minute evaluation frequency. For sub-minute alerting, use metric alerts.
- **`ago()` in KQL**: Always use in the query — workspace queries without time filters can scan all data and be very slow.
