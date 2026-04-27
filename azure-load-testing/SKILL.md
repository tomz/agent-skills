---
name: azure-load-testing
description: Azure Load Testing — JMeter-based load tests, CI/CD integration, server-side metrics, auto-stop criteria, VNet injection, test plans
license: MIT
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Load Testing

Azure Load Testing (ALT) is a fully managed load-testing service that generates
high-scale traffic using Apache JMeter scripts or URL-based quick tests.

---

## Resource Creation

```bash
# Register provider (once per subscription)
az provider register --namespace Microsoft.LoadTestService

# Create a Load Testing resource
az load create \
  --name myLoadTest \
  --resource-group myRG \
  --location eastus

# List resources
az load list --resource-group myRG --output table

# Delete resource
az load delete --name myLoadTest --resource-group myRG --yes
```

---

## Test Plans

### URL-based Quick Test (no JMX needed)

```bash
az load test create \
  --test-id quicktest-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --display-name "Homepage quick test" \
  --description "Quick smoke load test" \
  --engine-instances 1 \
  --env TARGET_URL=https://myapp.azurewebsites.net
```

### JMeter JMX Test Plan

A minimal `test-plan.jmx` structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan testname="My Load Test" enabled="true">
      <hashTree>
        <ThreadGroup testname="Users" enabled="true">
          <stringProp name="ThreadGroup.num_threads">${__P(threads,10)}</stringProp>
          <stringProp name="ThreadGroup.ramp_time">${__P(rampUpTime,30)}</stringProp>
          <stringProp name="ThreadGroup.duration">${__P(duration,120)}</stringProp>
          <boolProp name="ThreadGroup.scheduler">true</boolProp>
          <hashTree>
            <HTTPSamplerProxy testname="GET Homepage" enabled="true">
              <stringProp name="HTTPSampler.domain">${TARGET_HOST}</stringProp>
              <stringProp name="HTTPSampler.port">443</stringProp>
              <stringProp name="HTTPSampler.protocol">https</stringProp>
              <stringProp name="HTTPSampler.path">/</stringProp>
              <stringProp name="HTTPSampler.method">GET</stringProp>
              <hashTree/>
            </HTTPSamplerProxy>
          </hashTree>
        </ThreadGroup>
      </hashTree>
    </TestPlan>
  </hashTree>
</jmeterTestPlan>
```

Upload and create the test:

```bash
az load test create \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --display-name "API load test" \
  --test-plan test-plan.jmx \
  --engine-instances 5

# Upload additional files (CSVs, certs)
az load test file upload \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --path users.csv \
  --file-type ADDITIONAL_ARTIFACTS
```

---

## Test Configuration

### Engine Instances and Load Parameters

```bash
# Update engine count and JMeter properties
az load test update \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --engine-instances 10 \
  --env threads=50 rampUpTime=60 duration=300

# View test config
az load test show \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG
```

### Environment Variables

```bash
az load test update \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --env TARGET_HOST=myapp.azurewebsites.net \
        API_VERSION=v2 \
        REGION=eastus
```

### Secrets from Key Vault

Reference Key Vault secrets so credentials never appear in test config:

```bash
# Grant ALT resource access to Key Vault
az keyvault set-policy \
  --name myKeyVault \
  --object-id $(az load show --name myLoadTest --resource-group myRG \
                  --query identity.principalId -o tsv) \
  --secret-permissions get list

# Reference secret in test
az load test update \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --secret API_KEY="keyvaultref:https://myKeyVault.vault.azure.net/secrets/ApiKey,akv"
```

In JMX, reference as `${API_KEY}`.

---

## Server-Side Metrics

Enable resource metrics so test results include backend telemetry alongside
client-side latency and throughput.

```bash
# App Service
az load test server-metric add \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --metric-id "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/myApp" \
  --metric-name "CpuPercentage" \
  --metric-namespace "microsoft.web/sites" \
  --aggregation Average

# AKS node CPU
az load test server-metric add \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --metric-id "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.ContainerService/managedClusters/myAKS" \
  --metric-name "node_cpu_usage_percentage" \
  --metric-namespace "Microsoft.ContainerService/managedClusters" \
  --aggregation Average

# Azure SQL DTU consumption
az load test server-metric add \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --metric-id "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Sql/servers/mySql/databases/myDb" \
  --metric-name "dtu_consumption_percent" \
  --metric-namespace "Microsoft.Sql/servers/databases" \
  --aggregation Average

# Cosmos DB RU consumption
az load test server-metric add \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --metric-id "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.DocumentDB/databaseAccounts/myCosmos" \
  --metric-name "NormalizedRUConsumption" \
  --metric-namespace "Microsoft.DocumentDB/databaseAccounts" \
  --aggregation Maximum
```

---

## Auto-Stop Criteria

Automatically halt a test run when thresholds are breached to avoid runaway costs:

```bash
az load test update \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --autostop error-rate=10.0 time-window=60

# Error rate: stop if >10% errors over 60-second rolling window
# Response time: define as pass/fail criteria (not autostop, see below)
```

### Pass/Fail Criteria

```bash
az load test update \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --passFailCriteria "{
    \"passFailMetrics\": {
      \"pf_01\": {
        \"clientmetric\": \"response_time_ms\",
        \"aggregate\": \"p90\",
        \"condition\": \">\",
        \"value\": 2000,
        \"action\": \"stop\"
      },
      \"pf_02\": {
        \"clientmetric\": \"error\",
        \"aggregate\": \"percentage\",
        \"condition\": \">\",
        \"value\": 5,
        \"action\": \"continue\"
      }
    }
  }"
```

---

## VNet Injection for Private Endpoints

When your app is only reachable on a private network, inject ALT engines into
a delegated subnet:

```bash
# Delegate the subnet to ALT
az network vnet subnet update \
  --name alt-subnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --delegations Microsoft.LoadTestService/loadTests

# Create test with VNet injection
az load test create \
  --test-id private-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --subnet "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/alt-subnet"
```

The ALT engines are deployed inside your VNet for the duration of the test and
can reach private endpoints, internal load balancers, and App Service with
access restrictions.

---

## Running Tests

```bash
# Start a test run
az load test-run create \
  --test-id jmx-test-001 \
  --test-run-id run-$(date +%Y%m%d-%H%M%S) \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --display-name "Prod smoke 2026-04-24"

# List runs for a test
az load test-run list \
  --test-id jmx-test-001 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --output table

# Show run details (status, metrics summary)
az load test-run show \
  --test-run-id run-20260424-120000 \
  --load-test-resource myLoadTest \
  --resource-group myRG

# Stop a running test
az load test-run stop \
  --test-run-id run-20260424-120000 \
  --load-test-resource myLoadTest \
  --resource-group myRG

# Download client-side results CSV
az load test-run download-files \
  --test-run-id run-20260424-120000 \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --path ./results
```

---

## Test Results: Metrics and Comparison

Key client metrics in results:

| Metric | Description |
|--------|-------------|
| `throughput` | Requests/sec |
| `response_time_ms` (p50/p90/p95/p99) | Latency percentiles |
| `error` | Error percentage |
| `active_users` | Concurrent virtual users |

```bash
# Compare two test runs
az load test-run compare \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --test-run-ids run-baseline run-candidate
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/load-test.yml
name: Load Test
on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Run Azure Load Test
        uses: azure/load-testing@v1
        with:
          loadTestConfigFile: load-test-config.yaml
          loadTestResource: myLoadTest
          resourceGroup: myRG
          env: |
            [
              { "name": "TARGET_HOST", "value": "myapp.azurewebsites.net" }
            ]

      - name: Upload Results
        uses: actions/upload-artifact@v4
        with:
          name: load-test-results
          path: ${{ github.workspace }}/loadTest
```

`load-test-config.yaml`:

```yaml
version: v0.1
updated: 2026-04-24
testId: jmx-test-001
displayName: GitHub Actions load test
description: Triggered from CI pipeline
engineInstances: 3
testPlan: test-plan.jmx
failureCriteria:
  - p90(response_time_ms) > 2000
  - percentage(error) > 5
env:
  - name: threads
    value: "25"
  - name: duration
    value: "120"
```

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

steps:
  - task: AzureLoadTest@1
    inputs:
      azureSubscription: myServiceConnection
      loadTestConfigFile: load-test-config.yaml
      loadTestResource: myLoadTest
      resourceGroup: myRG
      outputVariables:
        - engine_health_metrics
```

---

## Bicep Example

```bicep
param location string = resourceGroup().location
param loadTestName string = 'myLoadTest'

resource loadTest 'Microsoft.LoadTestService/loadTests@2022-12-01' = {
  name: loadTestName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    description: 'Load testing resource for myApp'
  }
}

// Grant Key Vault access to the managed identity
resource kvPolicy 'Microsoft.KeyVault/vaults/accessPolicies@2022-07-01' = {
  name: '${keyVaultName}/add'
  properties: {
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: loadTest.identity.principalId
        permissions: {
          secrets: ['get', 'list']
        }
      }
    ]
  }
}

output loadTestPrincipalId string = loadTest.identity.principalId
```

---

## Tips and Best Practices

- **Ramp up gradually** — use a ramp-up period of at least 20–30% of test duration
  to avoid cold-start spikes masking steady-state behavior.
- **Isolate test data** — use dedicated test databases or test-mode feature flags.
- **Engine sizing** — each engine handles ~250 threads at comfortable concurrency;
  use `engine-instances` to scale horizontally.
- **Private tests** — always use VNet injection when testing internal endpoints;
  never expose them temporarily for a test.
- **Baseline first** — run a low-load baseline before ramp tests to establish
  healthy p90 and error-rate benchmarks.
- **Cost** — billed per VUh (virtual user-hour); auto-stop criteria prevent
  runaway costs from stuck tests.
