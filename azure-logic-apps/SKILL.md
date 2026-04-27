---
name: azure-logic-apps
description: Azure Logic Apps — workflow automation, connectors, triggers, actions, Standard vs Consumption, integration with 1000+ services
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Logic Apps — Comprehensive Reference

Azure Logic Apps is a cloud platform for automating workflows and integrating apps,
data, and services via a visual designer and JSON-based Workflow Definition Language (WDL).

---

## 1. Hosting Models

| Feature                | Consumption (multi-tenant)      | Standard (single-tenant)          | ISE (deprecated Aug 2024) |
|------------------------|---------------------------------|-----------------------------------|---------------------------|
| Pricing                | Per action (~$0.000025/action)  | vCPU + memory (App Service Plan)  | Dedicated, fixed           |
| Workflows per app      | 1                               | Many (stateful + stateless)       | Many                       |
| VNet integration       | Limited                         | Full (inbound PE + outbound VNet) | Full                       |
| Local development      | No                              | Yes (VS Code extension)           | No                         |
| Built-in connectors    | Shared (metered)                | In-process (free)                 | In-process                 |
| B2B / Integration Acct | Required for EDI/maps           | Partially built-in                | Built-in                   |

**Choose Consumption** for low-volume, spiky workloads. **Choose Standard** for high-volume,
VNet requirements, multi-workflow apps, or local development needs.

---

## 2. Triggers

Every workflow starts with exactly one trigger.

### Recurrence
```json
"Recurrence": {
  "type": "Recurrence",
  "recurrence": { "frequency": "Hour", "interval": 4, "startTime": "2024-01-01T08:00:00Z", "timeZone": "Eastern Standard Time" }
}
```

### HTTP (Request) — exposes a POST webhook URL
```json
"manual": {
  "type": "Request", "kind": "Http",
  "inputs": {
    "schema": {
      "type": "object",
      "properties": { "orderId": { "type": "string" }, "amount": { "type": "number" } },
      "required": ["orderId"]
    }
  }
}
```
Retrieve callback URL: `listCallbackUrl()` expression or Portal → Overview.

### Service Bus (poll-based)
```json
"When_message_received": {
  "type": "ApiConnection",
  "inputs": {
    "host": { "connection": { "name": "@parameters('$connections')['servicebus']['connectionId']" } },
    "method": "get",
    "path": "/@{encodeURIComponent('myqueue')}/messages/head",
    "queries": { "queueType": "Main" }
  },
  "recurrence": { "frequency": "Minute", "interval": 1 }
}
```

### Event Grid (webhook-based)
Use `ApiConnectionWebhook` type; Logic Apps registers/deregisters subscriptions automatically.
Filter by `includedEventTypes` (e.g., `Microsoft.Storage.BlobCreated`).

### Blob Storage
Uses `ApiConnection` with polling recurrence; returns up to `maxFileCount` changed blobs per cycle.

### Manual / HTTP Webhook
`Request` kind triggers wait for a callback; useful for approval flows and async patterns.

---

## 3. Actions

### HTTP
```json
"Call_API": {
  "type": "Http",
  "inputs": {
    "method": "POST", "uri": "https://api.example.com/orders",
    "headers": { "Content-Type": "application/json", "x-api-key": "@parameters('apiKey')" },
    "body": { "orderId": "@triggerBody()?['orderId']", "amount": "@triggerBody()?['amount']" },
    "retryPolicy": { "type": "exponential", "count": 4, "interval": "PT7S", "maximumInterval": "PT1H" }
  }
}
```

### Condition (If/Else)
```json
"Check_Amount": {
  "type": "If",
  "expression": { "and": [{ "greater": ["@triggerBody()?['amount']", 1000] }] },
  "actions": { "Send_Approval": {} },
  "else": { "actions": { "Auto_Approve": {} } },
  "runAfter": {}
}
```

### For-Each Loop
```json
"Process_Items": {
  "type": "Foreach", "foreach": "@body('Get_Items')?['value']",
  "actions": { "Insert_Row": {} },
  "runtimeConfiguration": { "concurrency": { "repetitions": 10 } },
  "runAfter": { "Get_Items": ["Succeeded"] }
}
```
Default is sequential (repetitions=1). Max 50 for Consumption.

### Until Loop
```json
"Poll_Until_Done": {
  "type": "Until",
  "expression": "@equals(body('Check_Status')?['status'], 'completed')",
  "limit": { "count": 20, "timeout": "PT2H" },
  "actions": {
    "Check_Status": { "type": "Http", "inputs": { "method": "GET", "uri": "https://api.example.com/status/@{variables('jobId')}" } },
    "Wait": { "type": "Wait", "inputs": { "interval": { "count": 30, "unit": "Second" } }, "runAfter": { "Check_Status": ["Succeeded"] } }
  }
}
```

### Switch, Scope, Variables
```json
"Route_By_Type": {
  "type": "Switch", "expression": "@triggerBody()?['eventType']",
  "cases": {
    "OrderCreated":   { "case": "OrderCreated",   "actions": {} },
    "OrderCancelled": { "case": "OrderCancelled", "actions": {} }
  },
  "default": { "actions": {} }
}
```
**Variables:** `InitializeVariable`, `SetVariable`, `IncrementVariable`, `AppendToArrayVariable`.
**Scope:** wraps a group of actions — foundation for try-catch (see §6).

---

## 4. Connectors

### Built-in (in-process, free in Standard)
HTTP, Request, Schedule, Control (Condition/Loop/Switch/Scope/Terminate), Variables,
Data Operations (Compose, Parse JSON, Filter Array, Select, Join), Date/Time, Inline Code
(JS in Consumption; C#/JS in Standard), Azure Functions, Liquid/XML transforms.

### Managed (shared service, billed per call)
- **Standard:** Office 365, SharePoint, Dynamics 365, Salesforce, Slack, Twilio, SQL Azure
- **Enterprise:** SAP, IBM MQ, IBM 3270, IBM DB2
- **On-premises (via Data Gateway):** SQL Server, Oracle, File System, SharePoint on-prem

### Custom Connectors
Import from OpenAPI 2.0 (Swagger) or Postman collection. Shared across Logic Apps,
Power Automate, and Power Apps within the same tenant.

---

## 5. Expressions & WDL Functions

All expressions begin with `@`; embed in strings as `@{...}`.

```plaintext
# Trigger / action output
@triggerBody()                          -- parsed trigger JSON payload
@triggerBody()?['field']               -- safe navigation (null if missing)
@triggerOutputs()?['headers']['x-id']  -- trigger HTTP header
@body('ActionName')                    -- body of a named action
@outputs('ActionName')?['statusCode']  -- HTTP status code of action
@result('ScopeName')?[0]?['error']?['message']  -- scope/action error detail

# Strings
@concat('Order-', triggerBody()?['id'])
@toLower(triggerBody()?['email'])  |  @toUpper(variables('code'))
@replace(body('A')?['text'], 'old', 'new')
@substring('Hello World', 6, 5)    -- 'World'
@split(triggerBody()?['csv'], ',') -- string → array
@trim('  spaces  ')

# JSON / encoding
@json('{"key":"val"}')             -- parse string to object
@string(body('Parse_JSON'))        -- serialize object to string
@base64(body('Action'))
@base64ToString(triggerBody()?['data'])
@xml(body('Compose'))
@xpath(xml(body('XML')), 'string(//OrderId)')

# Arrays
@first(body('Get')?['value'])  |  @last(...)  |  @length(...)
@union(arr1, arr2)  |  @intersection(arr1, arr2)
@skip(array, 2)    |  @take(array, 5)

# Math
@add(variables('counter'), 1)
@mul(triggerBody()?['qty'], triggerBody()?['price'])
@div(variables('total'), 100)  |  @mod(variables('i'), 10)

# Date/Time
@utcNow()  |  @utcNow('yyyy-MM-ddTHH:mm:ssZ')
@addDays(utcNow(), 7)  |  @addHours(utcNow(), -5)
@formatDateTime(utcNow(), 'yyyy-MM-dd')
@convertTimeZone(utcNow(), 'UTC', 'Eastern Standard Time')
@ticks(utcNow())                   -- Int64 ticks, useful for unique IDs

# Workflow metadata
@workflow().run.id                 -- current run ID
@workflow().name                   -- workflow name
@parameters('paramName')
@variables('varName')

# Conditionals
@if(greater(triggerBody()?['amount'], 1000), 'high', 'normal')
@coalesce(triggerBody()?['opt'], 'default')
@empty(body('Get')?['value'])
@not(empty(triggerBody()?['email']))
```

---

## 6. Error Handling

### Retry Policies (per action)
```json
"retryPolicy": { "type": "exponential", "count": 4, "interval": "PT7S", "maximumInterval": "PT1H", "minimumInterval": "PT5S" }
```
Types: `exponential` (default, 4 retries), `fixed`, `none`.

### runAfter — execution dependency
```json
"runAfter": { "Previous_Action": ["Succeeded", "Failed", "Skipped", "TimedOut"] }
```

### Try-Catch-Finally Pattern
```json
{
  "Try": {
    "type": "Scope", "runAfter": {},
    "actions": { "Risky_HTTP": { "type": "Http", "inputs": {} } }
  },
  "Catch": {
    "type": "Scope", "runAfter": { "Try": ["Failed", "TimedOut"] },
    "actions": {
      "Log_Error": {
        "type": "Http",
        "inputs": {
          "method": "POST", "uri": "https://logs.example.com/errors",
          "body": { "runId": "@workflow().run.id", "error": "@result('Try')?[0]?['error']?['message']" }
        }
      }
    }
  },
  "Finally": {
    "type": "Scope",
    "runAfter": {
      "Try":   ["Succeeded","Failed","Skipped","TimedOut"],
      "Catch": ["Succeeded","Failed","Skipped","TimedOut"]
    },
    "actions": { "Cleanup": {} }
  }
}
```

### Terminate Action
```json
"Fail_Workflow": {
  "type": "Terminate",
  "inputs": { "runStatus": "Failed", "runError": { "code": "VALIDATION_ERROR", "message": "@concat('Invalid: ', triggerBody()?['orderId'])" } }
}
```

---

## 7. Integration Account & B2B

Required for EDI/transforms in Consumption tier; partially built-in for Standard.

**Tiers:** Free (dev/test) · Basic (B2B messaging) · Standard (full EDI + maps + schemas)

**Artifacts:** Partners (business identities) · Agreements (AS2/X12/EDIFACT send/receive rules)
· Schemas (XSD for XML validation) · Maps (XSLT or Liquid for transformation)
· Certificates (AS2 signing/encryption)

**B2B Flow:** `Receive X12 → Decode X12 → Route by transaction set → Process → Encode X12 → Send`

Link in Portal: Logic App → **Workflow settings** → Integration account.

---

## 8. Stateful vs Stateless (Standard only)

| Feature           | Stateful                    | Stateless                    |
|-------------------|-----------------------------|------------------------------|
| Run history       | Stored in Azure Storage     | In-memory only               |
| Max duration      | Up to 1 year                | ~5 minutes                   |
| Resubmit / debug  | Yes                         | No                           |
| Throughput        | Good                        | Higher (no storage I/O)      |
| Best for          | Auditable, long-running     | Low-latency, high-volume     |

Enable stateless: in `workflow.json` set `"operationOptions": "EnableStatelessMode"`.

---

## 9. Managed Identity Authentication

```bash
# Enable system-assigned identity
az logic workflow update --resource-group myRG --name myLogicApp --identity-type SystemAssigned

# Grant access to Key Vault
az keyvault set-policy --name myKV --object-id <principalId> --secret-permissions get list
```

Use in HTTP action:
```json
"authentication": { "type": "ManagedServiceIdentity", "audience": "https://management.azure.com" }
```

---

## 10. Deployment: ARM / Bicep & CLI

### ARM Template (Consumption)
```json
{
  "type": "Microsoft.Logic/workflows", "apiVersion": "2019-05-01",
  "name": "[parameters('logicAppName')]", "location": "[parameters('location')]",
  "identity": { "type": "SystemAssigned" },
  "properties": {
    "state": "Enabled",
    "definition": {
      "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
      "contentVersion": "1.0.0.0",
      "triggers": {}, "actions": {}
    }
  }
}
```

### Bicep
```bicep
resource logicApp 'Microsoft.Logic/workflows@2019-05-01' = {
  name: logicAppName
  location: location
  identity: { type: 'SystemAssigned' }
  properties: { state: 'Enabled', definition: { triggers: {}, actions: {} } }
}
```

### Key CLI Commands
```bash
az logic workflow create --resource-group myRG --name myLogicApp --location eastus --definition @def.json
az logic workflow show  --resource-group myRG --name myLogicApp
az logic workflow list  --resource-group myRG --output table
az logic workflow show  --resource-group myRG --name myLogicApp --query properties.definition -o json > def.json
az logic workflow update --resource-group myRG --name myLogicApp --state Disabled
az logic workflow delete --resource-group myRG --name myLogicApp --yes
az logic workflow run list        --resource-group myRG --name myLogicApp --output table
az logic workflow run show        --resource-group myRG --name myLogicApp --run-name <runId>
az logic workflow run action list --resource-group myRG --name myLogicApp --run-name <runId>
az deployment group create --resource-group myRG --template-file azuredeploy.json --parameters logicAppName=myLogicApp

# Standard tier: zip deploy
cd logic-apps/standard && zip -r ../app.zip .
az logicapp deployment source config-zip --resource-group myRG --name myLogicApp --src ../app.zip
```

---

## 11. CI/CD — GitHub Actions

```yaml
name: Deploy Logic App
on:
  push:
    branches: [main]
    paths: ['logic-apps/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v1
        with: { creds: "${{ secrets.AZURE_CREDENTIALS }}" }
      - uses: azure/arm-deploy@v1
        with:
          resourceGroupName: myRG
          template: ./logic-apps/azuredeploy.json
          parameters: logicAppName=myLogicApp
```
For Standard tier, replace the last step with the `az logicapp deployment source config-zip` command.

---

## 12. Monitoring & Observability

### Tracked Properties (custom run metadata)
```json
"Send_Email": {
  "type": "ApiConnection",
  "trackedProperties": { "orderId": "@triggerBody()?['orderId']", "customer": "@triggerBody()?['email']" },
  "inputs": {}
}
```

### Enable Log Analytics Diagnostics
```bash
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Logic/workflows/myLogicApp \
  --workspace <workspaceResourceId> --name logic-diag \
  --logs '[{"category":"WorkflowRuntime","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

### KQL Queries
```kql
// Failed runs last 24h
AzureDiagnostics
| where ResourceType == "WORKFLOWS" and status_s == "Failed" and TimeGenerated > ago(24h)
| project TimeGenerated, resource_workflowName_s, error_message_s | order by TimeGenerated desc

// Action failure rate
AzureDiagnostics | where ResourceType == "WORKFLOWS"
| summarize total=count(), failed=countif(status_s=="Failed") by resource_actionName_s
| extend failureRate = round(100.0 * failed / total, 2) | order by failureRate desc
```

---

## 13. Security

**IP Restrictions** — `accessControl.triggers.allowedCallerIpAddresses` in workflow properties.

**Secure inputs/outputs** — hide data from run history:
```json
"runtimeConfiguration": { "secureData": { "properties": ["inputs", "outputs"] } }
```

**Private Endpoints (Standard):**
```bash
az network private-endpoint create --name myPE --resource-group myRG \
  --vnet-name myVNet --subnet mySubnet \
  --private-connection-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myLogicApp \
  --group-id sites --connection-name myConn

az webapp vnet-integration add --resource-group myRG --name myLogicApp --vnet myVNet --subnet mySubnet
```

**Encryption:** Data at rest uses Microsoft-managed keys by default; customer-managed keys (CMK)
available for Standard. TLS 1.2+ enforced in transit.

---

## 14. Performance Patterns

| Pattern              | Configuration                                                     |
|----------------------|-------------------------------------------------------------------|
| Parallel for-each    | `"runtimeConfiguration": { "concurrency": { "repetitions": 20 } }` |
| Trigger concurrency  | `"concurrency": { "runs": 10, "maximumWaitingRuns": 100 }`        |
| Large payload        | `"contentTransfer": { "transferMode": "Chunked" }`                |
| Auto-pagination      | `"paginationPolicy": { "minimumItemCount": 1000 }`                |
| Message batching     | Batch trigger with `releaseCriteria` (messageCount + recurrence)  |

---

## 15. Common Patterns

**Approval Workflow:**
`HTTP Trigger → Send approval email (O365) → Condition → If approved: proceed; else: notify`

**Scheduled ETL:**
`Recurrence → Query SQL → For-each row → Transform (Select action) → Upsert target → Email summary`

**File Processing:**
`Blob Created → Get content → Validate → Move to /processed/ or /errors/ + alert`

**API Orchestration:** Multiple HTTP actions share a step for implicit parallelism.
Use Azure Durable Functions for true fan-out/fan-in with joins.

**Dead-letter Queue:**
`Service Bus DLQ trigger → Parse → Remediate → Re-enqueue or log/alert`

---

## 16. Pricing

### Consumption (East US approx.)
| Resource                    | Rate                      |
|-----------------------------|---------------------------|
| Actions (first 5M/month)    | Free                      |
| Actions (per 1M after)      | ~$0.025                   |
| Standard connector calls    | ~$0.00025 each            |
| Enterprise connector calls  | ~$0.001 each              |
| Integration Account Basic   | ~$275/month               |
| Integration Account Standard| ~$1,100/month             |

### Standard (East US approx.)
| Plan                      | Rate                       |
|---------------------------|----------------------------|
| WS1 (1 vCPU, 3.5 GB RAM) | ~$0.192/hr (~$140/month)   |
| WS2 (2 vCPU, 7 GB RAM)   | ~$0.383/hr (~$280/month)   |
| WS3 (4 vCPU, 14 GB RAM)  | ~$0.767/hr (~$560/month)   |
| Built-in connector calls  | Included                   |
| Managed connector calls   | Same as Consumption rates  |

---

## 17. References

- Docs: https://learn.microsoft.com/azure/logic-apps/
- WDL functions: https://learn.microsoft.com/azure/logic-apps/workflow-definition-language-functions-reference
- Connector catalog: https://learn.microsoft.com/connectors/
- WDL JSON schema: `https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json`
- VS Code extension: `ms-azuretools.vscode-azurelogicapps`
- Pricing calculator: https://azure.microsoft.com/pricing/calculator/
