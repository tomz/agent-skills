---
name: azure-app-config
description: Azure App Configuration — centralized config management, feature flags, key-value store, Key Vault references, labels, snapshots, .NET/Java/Python/JS SDKs
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure App Configuration

Centralized, managed configuration service for Azure applications. Decouples config
from code — store key-values, feature flags, and Key Vault references in one place,
with dynamic refresh, versioning, and audit history.

---

## Tiers

| Feature | Free | Standard |
|---------|------|----------|
| Storage | 10 MB | 1 GB |
| Requests/day | 1,000 | Unlimited |
| Geo-replication | ❌ | ✅ |
| Private endpoints | ❌ | ✅ |
| Snapshots | ❌ | ✅ |
| Customer-managed keys | ❌ | ✅ |
| SLA | None | 99.9% |
| Price | Free | ~$1.20/day |

Use Free for dev/test, Standard for production.

---

## Core Concepts

### Keys and Labels

Keys are hierarchical strings (use `/` or `:` as separator):

```
myapp/database/connectionString
myapp:feature:darkMode
```

**Labels** add a dimension (environment, region, version):
- `myapp/timeout` with label `production`
- `myapp/timeout` with label `staging`
- Unlabeled keys use the null label (`\0`)

**Content types** signal value format:
- `application/json` — parsed as JSON
- `application/vnd.microsoft.appconfig.ff+json` — feature flag (managed automatically)
- `application/vnd.microsoft.appconfig.keyvaultref+json` — Key Vault reference

### Revision History

Every write creates a new revision. The store retains 7 days of history (Standard)
for point-in-time reads and auditing. Use `az appconfig revision list` to browse.

---

## az appconfig CLI Reference

### Create and Manage Stores

```bash
# Create a store
az appconfig create \
  --name myapp-config \
  --resource-group myapp-rg \
  --location eastus \
  --sku Standard

# List stores
az appconfig list --resource-group myapp-rg

# Get connection string (for local dev)
az appconfig credential list \
  --name myapp-config \
  --resource-group myapp-rg \
  --query "[?name=='Primary Read Only'].connectionString" -o tsv

# Delete a store
az appconfig delete --name myapp-config --resource-group myapp-rg --yes
```

### Key-Value Operations

```bash
# Set a key-value
az appconfig kv set \
  --name myapp-config \
  --key "myapp/database/timeout" \
  --value "30" \
  --label production \
  --yes

# Set JSON content
az appconfig kv set \
  --name myapp-config \
  --key "myapp/cache/settings" \
  --value '{"ttl":300,"maxItems":1000}' \
  --content-type "application/json" \
  --yes

# Get a value
az appconfig kv show \
  --name myapp-config \
  --key "myapp/database/timeout" \
  --label production

# List all key-values (filtered)
az appconfig kv list \
  --name myapp-config \
  --key "myapp/*" \
  --label production

# Delete a key
az appconfig kv delete \
  --name myapp-config \
  --key "myapp/database/timeout" \
  --label production \
  --yes

# Lock a key (prevent writes without unlock)
az appconfig kv lock --name myapp-config --key "myapp/database/timeout"
az appconfig kv unlock --name myapp-config --key "myapp/database/timeout"
```

### Import / Export (CI/CD)

```bash
# Import from JSON file (bulk config load)
az appconfig kv import \
  --name myapp-config \
  --source file \
  --path ./config/production.json \
  --format json \
  --label production \
  --separator ":" \
  --yes

# Export to JSON file
az appconfig kv export \
  --name myapp-config \
  --destination file \
  --path ./config/exported.json \
  --format json \
  --label production \
  --yes

# Import from another App Configuration store
az appconfig kv import \
  --name myapp-config-eastus \
  --source appconfig \
  --src-name myapp-config-westus \
  --label production \
  --yes
```

Input JSON format for import:
```json
{
  "database": {
    "host": "prod-db.example.com",
    "port": 5432,
    "timeout": 30
  },
  "cache": {
    "ttl": 300
  }
}
```
With `--separator ":"` this creates keys: `database:host`, `database:port`, etc.

### Feature Flags

```bash
# Create a feature flag (enabled)
az appconfig feature set \
  --name myapp-config \
  --feature dark-mode \
  --label production \
  --yes

# Enable / disable
az appconfig feature enable  --name myapp-config --feature dark-mode --label production --yes
az appconfig feature disable --name myapp-config --feature dark-mode --label production --yes

# Show feature flag
az appconfig feature show --name myapp-config --feature dark-mode --label production

# List feature flags
az appconfig feature list --name myapp-config --label production

# Add a percentage rollout filter (50% of users)
az appconfig feature filter add \
  --name myapp-config \
  --feature dark-mode \
  --filter-name Microsoft.Percentage \
  --filter-parameters Value=50 \
  --label production \
  --yes

# Add a targeting filter (specific users/groups)
az appconfig feature filter add \
  --name myapp-config \
  --feature dark-mode \
  --filter-name Microsoft.Targeting \
  --filter-parameters '{"Audience":{"Users":["alice@example.com"],"Groups":[{"Name":"beta-testers","RolloutPercentage":25}],"DefaultRolloutPercentage":0}}' \
  --label production \
  --yes

# Add a time window filter
az appconfig feature filter add \
  --name myapp-config \
  --feature holiday-banner \
  --filter-name Microsoft.TimeWindow \
  --filter-parameters Start="2026-12-24T00:00:00Z" End="2026-12-26T00:00:00Z" \
  --label production \
  --yes

# Delete a feature flag
az appconfig feature delete --name myapp-config --feature dark-mode --label production --yes
```

### Snapshots (Immutable Config Versions)

```bash
# Create a snapshot (point-in-time copy, Standard tier only)
az appconfig snapshot create \
  --name myapp-config \
  --snapshot-name "release-v2.1.0" \
  --filters '[{"key":"myapp/*","label":"production"}]'

# List snapshots
az appconfig snapshot list --name myapp-config

# Show a snapshot
az appconfig snapshot show \
  --name myapp-config \
  --snapshot-name "release-v2.1.0"

# Read key-values from a snapshot (for rollback/audit)
az appconfig kv list \
  --name myapp-config \
  --snapshot "release-v2.1.0"

# Archive / recover a snapshot
az appconfig snapshot archive  --name myapp-config --snapshot-name "release-v2.1.0"
az appconfig snapshot recover  --name myapp-config --snapshot-name "release-v2.1.0"
```

---

## Key Vault References

Store a reference (not the secret value itself) so secrets stay in Key Vault:

```bash
az appconfig kv set-keyvaultref \
  --name myapp-config \
  --key "myapp/database/password" \
  --secret-identifier "https://myapp-kv.vault.azure.net/secrets/db-password" \
  --label production \
  --yes
```

The SDK resolves references transparently — the app sees the secret value,
not the JSON reference. Requires the app's identity to have `Key Vault Secrets User`
on the vault.

---

## Authentication

### Managed Identity (Production — recommended)

```bash
# Assign the app's managed identity the data reader role
az role assignment create \
  --assignee <managed-identity-object-id> \
  --role "App Configuration Data Reader" \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.AppConfiguration/configurationStores/myapp-config
```

Use `DefaultAzureCredential` in SDKs — picks up managed identity automatically.

### Connection String (Dev/Test)

Store in user secrets or `.env`, never in source control:
```
AZURE_APP_CONFIG_CONNECTION_STRING=Endpoint=https://myapp-config.azconfig.io;Id=...;Secret=...
```

---

## SDK Integration Patterns

### .NET — Builder Pattern with Dynamic Refresh

```csharp
// Program.cs
using Azure.Identity;
using Microsoft.Extensions.Configuration.AzureAppConfiguration;

var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddAzureAppConfiguration(options =>
{
    var endpoint = builder.Configuration["AppConfig:Endpoint"];
    options
        .Connect(new Uri(endpoint!), new DefaultAzureCredential())
        // Load all keys with label matching environment
        .Select(KeyFilter.Any, builder.Environment.EnvironmentName)
        // Sentinel key — triggers full refresh when changed
        .ConfigureRefresh(refresh =>
        {
            refresh.Register("myapp/sentinel", refreshAll: true)
                   .SetRefreshInterval(TimeSpan.FromSeconds(30));
        })
        // Resolve Key Vault references
        .ConfigureKeyVault(kv =>
        {
            kv.SetCredential(new DefaultAzureCredential());
        })
        // Feature flags
        .UseFeatureFlags(ff =>
        {
            ff.Select(KeyFilter.Any, builder.Environment.EnvironmentName)
              .SetRefreshInterval(TimeSpan.FromSeconds(30));
        });
});

// Register the middleware that triggers refresh on each request
builder.Services.AddAzureAppConfiguration();
builder.Services.AddFeatureManagement();

var app = builder.Build();
app.UseAzureAppConfiguration();  // <-- must be before other middleware
```

```csharp
// Use feature flags in a controller
using Microsoft.FeatureManagement;

public class HomeController : Controller
{
    private readonly IFeatureManager _featureManager;
    public HomeController(IFeatureManager featureManager)
        => _featureManager = featureManager;

    public async Task<IActionResult> Index()
    {
        if (await _featureManager.IsEnabledAsync("dark-mode"))
            ViewBag.Theme = "dark";
        return View();
    }
}
```

### Java Spring Boot

```java
// pom.xml dependency:
// com.azure.spring:spring-cloud-azure-appconfiguration-config:5.x

// application.properties
spring.cloud.azure.appconfiguration.stores[0].endpoint=https://myapp-config.azconfig.io
spring.cloud.azure.appconfiguration.stores[0].feature-flags.enabled=true
spring.cloud.azure.appconfiguration.stores[0].selects[0].label-filter=${spring.profiles.active}
spring.cloud.azure.appconfiguration.refresh.enabled=true
spring.cloud.azure.appconfiguration.refresh.interval=30s
```

```java
@RefreshScope
@RestController
public class ConfigController {

    @Value("${myapp.database.timeout:30}")
    private int dbTimeout;

    @Autowired
    private FeatureManager featureManager;

    @GetMapping("/info")
    public Map<String, Object> info() {
        return Map.of(
            "dbTimeout", dbTimeout,
            "darkMode", featureManager.isEnabledAsync("dark-mode").block()
        );
    }
}
```

### Python

```python
from azure.appconfiguration import AzureAppConfigurationClient
from azure.identity import DefaultAzureCredential

endpoint = "https://myapp-config.azconfig.io"
credential = DefaultAzureCredential()
client = AzureAppConfigurationClient(endpoint, credential)

# Read a single key-value
kv = client.get_configuration_setting(key="myapp/database/timeout", label="production")
print(kv.value)

# List all settings with a prefix
settings = client.list_configuration_settings(
    key_filter="myapp/*",
    label_filter="production"
)
for setting in settings:
    print(f"{setting.key} = {setting.value}")

# Set a key-value
from azure.appconfiguration import ConfigurationSetting
client.set_configuration_setting(ConfigurationSetting(
    key="myapp/cache/ttl",
    value="600",
    label="production"
))

# Dynamic refresh: poll for sentinel changes
import time

last_etag = None
while True:
    kv = client.get_configuration_setting(key="myapp/sentinel", label="production")
    if kv.etag != last_etag:
        last_etag = kv.etag
        # Reload all config
        reload_config(client)
    time.sleep(30)
```

### JavaScript / TypeScript

```typescript
import { AppConfigurationClient } from "@azure/app-configuration";
import { DefaultAzureCredential } from "@azure/identity";

const endpoint = process.env.AZURE_APP_CONFIG_ENDPOINT!;
const client = new AppConfigurationClient(endpoint, new DefaultAzureCredential());

// Read a value
const setting = await client.getConfigurationSetting({
  key: "myapp/database/timeout",
  label: "production",
});
console.log(setting.value);

// List with prefix
for await (const item of client.listConfigurationSettings({
  keyFilter: "myapp/*",
  labelFilter: "production",
})) {
  console.log(`${item.key} [${item.label}] = ${item.value}`);
}

// Feature flag check (manual — use SDK for full filtering)
const ff = await client.getConfigurationSetting({
  key: ".appconfig.featureflag/dark-mode",
  label: "production",
});
const flag = JSON.parse(ff.value!);
const isEnabled: boolean = flag.enabled;
```

---

## Sentinel Key Pattern (Dynamic Refresh)

A "sentinel" is a dedicated key whose only job is to signal that config has changed.
Apps poll only the sentinel (cheap) and reload all settings on change:

```bash
# After bulk-updating config, bump the sentinel
az appconfig kv set \
  --name myapp-config \
  --key "myapp/sentinel" \
  --value "$(date -u +%Y%m%dT%H%M%SZ)" \
  --label production \
  --yes
```

In CI/CD pipelines, add this as the last step after importing config files.

---

## Event-Driven Refresh via Event Grid

For near-real-time refresh without polling:

```bash
# Subscribe to App Configuration events → Azure Function or webhook
az eventgrid event-subscription create \
  --source-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.AppConfiguration/configurationStores/myapp-config \
  --name config-changed-sub \
  --endpoint-type azurefunction \
  --endpoint /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Web/sites/myapp-func/functions/ConfigChanged

# Event types: KeyValueModified, KeyValueDeleted, FeatureFlagModified
```

The function receives the event and signals the app (via SignalR, Service Bus, or Redis pub/sub)
to call `IConfigurationRefresher.TryRefreshAsync()` immediately.

---

## Private Endpoints

```bash
# Create a private endpoint (Standard tier)
az network private-endpoint create \
  --name myapp-config-pe \
  --resource-group myapp-rg \
  --vnet-name myapp-vnet \
  --subnet private-endpoints \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.AppConfiguration/configurationStores/myapp-config \
  --group-id configurationStores \
  --connection-name myapp-config-conn

# Create DNS zone and link
az network private-dns zone create \
  --resource-group myapp-rg \
  --name privatelink.azconfig.io

az network private-dns link vnet create \
  --resource-group myapp-rg \
  --zone-name privatelink.azconfig.io \
  --name myapp-dns-link \
  --virtual-network myapp-vnet \
  --registration-enabled false

# Add DNS record for the private endpoint IP
az network private-endpoint dns-zone-group create \
  --resource-group myapp-rg \
  --endpoint-name myapp-config-pe \
  --name default \
  --private-dns-zone privatelink.azconfig.io \
  --zone-name configurationStores
```

---

## Geo-Replication (Standard Tier)

```bash
# Add a replica in a secondary region
az appconfig replica create \
  --store-name myapp-config \
  --resource-group myapp-rg \
  --name myapp-config-westus \
  --location westus

# List replicas
az appconfig replica list \
  --store-name myapp-config \
  --resource-group myapp-rg

# Delete a replica
az appconfig replica delete \
  --store-name myapp-config \
  --resource-group myapp-rg \
  --name myapp-config-westus --yes
```

SDKs automatically fail over to replicas — no code changes needed.

---

## Bicep Example

```bicep
param location string = resourceGroup().location
param storeName string = 'myapp-config'
param appPrincipalId string

resource appConfig 'Microsoft.AppConfiguration/configurationStores@2024-05-01' = {
  name: storeName
  location: location
  sku: { name: 'Standard' }
  properties: {
    disableLocalAuth: true  // Managed identity only
    publicNetworkAccess: 'Disabled'
  }
}

// Assign data reader role to app's managed identity
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(appConfig.id, appPrincipalId, 'App Configuration Data Reader')
  scope: appConfig
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions',
      '516239f1-63e1-4d78-a4de-a74fb236a071')  // App Configuration Data Reader
    principalId: appPrincipalId
    principalType: 'ServicePrincipal'
  }
}

// Seed a key-value via ARM
resource dbTimeout 'Microsoft.AppConfiguration/configurationStores/keyValues@2024-05-01' = {
  parent: appConfig
  name: 'myapp~1database~1timeout$production'  // / → ~1, label after $
  properties: {
    value: '30'
    contentType: ''
  }
}

output endpoint string = appConfig.properties.endpoint
```

## Terraform Example

```hcl
resource "azurerm_app_configuration" "main" {
  name                = "myapp-config"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "standard"

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_role_assignment" "app_config_reader" {
  scope                = azurerm_app_configuration.main.id
  role_definition_name = "App Configuration Data Reader"
  principal_id         = azurerm_linux_web_app.main.identity[0].principal_id
}

resource "azurerm_app_configuration_key" "db_timeout" {
  configuration_store_id = azurerm_app_configuration.main.id
  key                    = "myapp/database/timeout"
  label                  = "production"
  value                  = "30"
}

resource "azurerm_app_configuration_feature" "dark_mode" {
  configuration_store_id = azurerm_app_configuration.main.id
  name                   = "dark-mode"
  label                  = "production"
  enabled                = false
}
```

---

## CI/CD Pipeline Pattern

```yaml
# GitHub Actions — deploy config before app
- name: Import App Config
  run: |
    az appconfig kv import \
      --name ${{ vars.APP_CONFIG_NAME }} \
      --source file \
      --path ./config/${{ vars.ENVIRONMENT }}.json \
      --format json \
      --label ${{ vars.ENVIRONMENT }} \
      --separator ":" \
      --yes

    # Take a snapshot for this release
    az appconfig snapshot create \
      --name ${{ vars.APP_CONFIG_NAME }} \
      --snapshot-name "release-${{ github.sha }}" \
      --filters "[{\"key\":\"myapp/*\",\"label\":\"${{ vars.ENVIRONMENT }}\"}]"

    # Bump sentinel to trigger app refresh
    az appconfig kv set \
      --name ${{ vars.APP_CONFIG_NAME }} \
      --key "myapp/sentinel" \
      --value "${{ github.sha }}" \
      --label ${{ vars.ENVIRONMENT }} \
      --yes
```

---

## Monitoring

```bash
# View revision history (who changed what, when)
az appconfig revision list \
  --name myapp-config \
  --key "myapp/*" \
  --label production \
  --datetime "2026-04-01T00:00:00Z"

# Enable diagnostic logs → Log Analytics
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.AppConfiguration/configurationStores/myapp-config \
  --name myapp-config-diag \
  --workspace /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/myapp-logs \
  --logs '[{"category":"HttpRequest","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

Key metrics to alert on:
- `HttpIncomingRequestCount` — request volume
- `ThrottledRequestCount` — hitting rate limits (scale to Standard)
- `RequestDuration` — latency spikes

KQL query for failed requests:
```kusto
AzureMetrics
| where ResourceProvider == "MICROSOFT.APPCONFIGURATION"
| where MetricName == "HttpIncomingRequestCount"
| where Total > 0
| summarize total=sum(Total) by bin(TimeGenerated, 5m)
| render timechart
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Set a key | `az appconfig kv set --name <store> --key <k> --value <v> --yes` |
| Get a key | `az appconfig kv show --name <store> --key <k>` |
| List keys | `az appconfig kv list --name <store> --key "prefix/*"` |
| Import JSON | `az appconfig kv import --name <store> --source file --path f.json --format json --yes` |
| Enable feature | `az appconfig feature enable --name <store> --feature <f> --yes` |
| Add % filter | `az appconfig feature filter add --filter-name Microsoft.Percentage --filter-parameters Value=50` |
| Create snapshot | `az appconfig snapshot create --name <store> --snapshot-name <n> --filters '[...]'` |
| Add KV ref | `az appconfig kv set-keyvaultref --name <store> --key <k> --secret-identifier <uri> --yes` |
| Bump sentinel | `az appconfig kv set --name <store> --key sentinel --value $(date +%s) --yes` |
