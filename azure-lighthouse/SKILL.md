---
name: azure-lighthouse
description: Azure Lighthouse — cross-tenant management, delegated resource management, managed service offers, RBAC projections, MSP scenarios
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Lighthouse

Azure Lighthouse enables Managed Service Providers (MSPs) and enterprise IT teams
to manage resources across multiple Azure tenants from a single control plane,
without requiring guest accounts or per-subscription logins in each customer tenant.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Managing tenant** | Your MSP tenant — where your engineers sign in |
| **Managed tenant** | Customer's Azure tenant — where their resources live |
| **Delegated resource management** | ARM projection that lets managing-tenant principals act on managed-tenant resources |
| **Authorization** | A binding: `principalId` (your tenant) → `roleDefinitionId` → scope in customer tenant |
| **Eligible authorization** | Just-in-time elevation via Azure PIM (no standing access) |

---

## Onboarding: ARM Template

Customers deploy this template **in their own tenant** to grant your MSP access.

`lighthouse-delegation.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-08-01/subscriptionDeploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "mspName": {
      "type": "string",
      "defaultValue": "Contoso MSP"
    },
    "mspOfferDescription": {
      "type": "string",
      "defaultValue": "Managed Security and Operations"
    },
    "managedByTenantId": {
      "type": "string",
      "metadata": { "description": "Your MSP tenant ID" }
    },
    "authorizations": {
      "type": "array"
    }
  },
  "resources": [
    {
      "type": "Microsoft.ManagedServices/registrationDefinitions",
      "apiVersion": "2022-10-01",
      "name": "[guid(parameters('mspName'))]",
      "properties": {
        "registrationDefinitionName": "[parameters('mspName')]",
        "description": "[parameters('mspOfferDescription')]",
        "managedByTenantId": "[parameters('managedByTenantId')]",
        "authorizations": "[parameters('authorizations')]"
      }
    },
    {
      "type": "Microsoft.ManagedServices/registrationAssignments",
      "apiVersion": "2022-10-01",
      "name": "[guid(parameters('mspName'), 'assignment')]",
      "properties": {
        "registrationDefinitionId": "[resourceId('Microsoft.ManagedServices/registrationDefinitions', guid(parameters('mspName')))]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.ManagedServices/registrationDefinitions', guid(parameters('mspName')))]"
      ]
    }
  ]
}
```

`lighthouse-delegation.parameters.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-05-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "mspName":            { "value": "Contoso MSP" },
    "managedByTenantId":  { "value": "<YOUR-MSP-TENANT-ID>" },
    "authorizations": {
      "value": [
        {
          "principalId":        "<MSP-GROUP-OR-USER-OBJECT-ID>",
          "principalIdDisplayName": "MSP Operations Team",
          "roleDefinitionId":   "acdd72a7-3385-48ef-bd42-f606fba81ae7"
        },
        {
          "principalId":        "<MSP-SECURITY-ANALYST-GROUP-ID>",
          "principalIdDisplayName": "MSP Security Analysts",
          "roleDefinitionId":   "39bc4728-0917-49c7-9d2c-d95423bc2eb4"
        }
      ]
    }
  }
}
```

Deploy from the **customer's** tenant (subscription scope):

```bash
az deployment sub create \
  --name lighthouse-onboard \
  --location eastus \
  --template-file lighthouse-delegation.json \
  --parameters @lighthouse-delegation.parameters.json
```

---

## Role Definitions and Principal Assignments

Common built-in roles used in Lighthouse delegations:

| Role | ID |
|------|----|
| Reader | `acdd72a7-3385-48ef-bd42-f606fba81ae7` |
| Contributor | `b24988ac-6180-42a0-ab88-20f7382dd24c` |
| Monitoring Reader | `43d0d8ad-25c7-4714-9337-8ba259a9fe05` |
| Monitoring Contributor | `749f88d5-cbae-40b8-bcfc-e573ddc772fa` |
| Security Reader | `39bc4728-0917-49c7-9d2c-d95423bc2eb4` |
| Log Analytics Contributor | `92aaf0da-9dab-42b6-94a3-d43ce8d16293` |
| Virtual Machine Contributor | `9980e02c-c2be-4d73-94e8-173b1dc7cf3c` |
| Azure Sentinel Contributor | `ab8e14d6-4a74-4a29-9ba8-549422addade` |

> **Best practice:** Use groups, not individual users, as `principalId` values.
> Manage membership in your MSP tenant's Azure AD — no customer action required
> when staff changes occur.

---

## Eligible Authorizations (PIM / JIT)

Eligible authorizations require engineers to explicitly activate their role,
providing just-in-time access with approval and time limits.

```json
{
  "principalId": "<MSP-ENGINEER-GROUP-ID>",
  "principalIdDisplayName": "MSP Engineers (JIT)",
  "roleDefinitionId": "b24988ac-6180-42a0-ab88-20f7382dd24c",
  "justInTimeAccessPolicy": {
    "multiFactorAuthProvider": "Azure",
    "maximumActivationDuration": "PT8H",
    "managedByTenantApprovers": [
      {
        "principalId": "<MSP-APPROVER-GROUP-ID>",
        "principalIdDisplayName": "MSP Approvers"
      }
    ]
  }
}
```

Add to the `authorizations` array; requires `Microsoft.ManagedServices` API version
`2022-10-01` or later.

---

## Azure Marketplace Managed Service Offers

Publish a Lighthouse offer so customers can onboard directly from the Marketplace:

1. In Partner Center, create a **Managed Service** offer type.
2. Add one or more **plans** — each plan maps to an authorization set
   (same structure as the ARM template above).
3. After publication, customers find the offer in **Marketplace → Management Services**.
4. Purchasing the offer deploys the registration definition and assignment automatically.

Benefits over manual ARM deployment:
- Versioned, updatable offers
- Audit trail via Marketplace purchase history
- No need to share ARM templates directly

---

## Managing Customer Resources

Once delegated, sign in to **your MSP tenant** and query across all delegated subscriptions:

```bash
# List all delegated subscriptions visible to your tenant
az managedservices assignment list --output table

# List registration definitions (offers) in a customer subscription
az managedservices definition list \
  --subscription <CUSTOMER-SUB-ID> \
  --output table

# Query VMs across all delegated scopes
az vm list --subscription <CUSTOMER-SUB-ID> --output table

# Cross-tenant Resource Graph query (requires az extension)
az extension add --name resource-graph

az graph query -q "
  Resources
  | where type == 'microsoft.compute/virtualmachines'
  | project name, resourceGroup, subscriptionId, location
" --management-groups <YOUR-MANAGING-TENANT-ID>
```

### Azure Policy (cross-tenant)

```bash
# Assign policy in customer subscription from managing tenant
az policy assignment create \
  --name enforce-tagging \
  --policy /providers/Microsoft.Authorization/policyDefinitions/<POLICY-DEF-ID> \
  --scope /subscriptions/<CUSTOMER-SUB-ID> \
  --subscription <CUSTOMER-SUB-ID>

# View compliance across delegated subscriptions
az policy state list \
  --subscription <CUSTOMER-SUB-ID> \
  --filter "complianceState eq 'NonCompliant'" \
  --output table
```

### Azure Monitor (cross-tenant)

```bash
# Query Log Analytics workspace in customer tenant
az monitor log-analytics query \
  --workspace <CUSTOMER-WORKSPACE-ID> \
  --analytics-query "SecurityEvent | summarize count() by EventID | top 10 by count_" \
  --subscription <CUSTOMER-SUB-ID>

# Create alert rule in customer subscription
az monitor metrics alert create \
  --name high-cpu-alert \
  --resource-group <CUSTOMER-RG> \
  --subscription <CUSTOMER-SUB-ID> \
  --scopes /subscriptions/<CUSTOMER-SUB-ID>/resourceGroups/<CUSTOMER-RG>/providers/Microsoft.Compute/virtualMachines/myVM \
  --condition "avg Percentage CPU > 90" \
  --window-size 5m \
  --evaluation-frequency 1m
```

### Microsoft Sentinel (cross-tenant)

With Sentinel Contributor role delegated, you can manage incidents, analytics rules,
and workbooks across all customer workspaces from a single Sentinel instance.

```bash
# List Sentinel incidents in customer workspace
az sentinel incident list \
  --workspace-name <CUSTOMER-WORKSPACE> \
  --resource-group <CUSTOMER-RG> \
  --subscription <CUSTOMER-SUB-ID> \
  --output table

# Create analytics rule
az sentinel alert-rule create \
  --workspace-name <CUSTOMER-WORKSPACE> \
  --resource-group <CUSTOMER-RG> \
  --subscription <CUSTOMER-SUB-ID> \
  --rule-name suspicious-login \
  --kind Scheduled \
  --query "SigninLogs | where ResultType != 0" \
  --display-name "Failed Sign-ins" \
  --severity Medium \
  --enabled true
```

---

## Revoking Access

### Customer-initiated removal

```bash
# Customer removes the registration assignment (revokes MSP access)
az managedservices assignment delete \
  --assignment <ASSIGNMENT-ID> \
  --subscription <CUSTOMER-SUB-ID>
```

### MSP-initiated removal (if authorized)

Only possible if the delegation includes a role with `Microsoft.ManagedServices/*/delete`
permission (e.g., Managed Services Registration Assignment Delete Role):

```bash
az managedservices assignment delete \
  --assignment <ASSIGNMENT-ID> \
  --subscription <CUSTOMER-SUB-ID>
```

---

## Bicep Template

```bicep
targetScope = 'subscription'

param mspName string = 'Contoso MSP'
param managedByTenantId string
param operationsGroupId string
param securityGroupId string

var registrationName = guid(mspName)
var assignmentName = guid(mspName, 'assignment')

resource registrationDef 'Microsoft.ManagedServices/registrationDefinitions@2022-10-01' = {
  name: registrationName
  properties: {
    registrationDefinitionName: mspName
    description: 'Managed Operations and Security by Contoso MSP'
    managedByTenantId: managedByTenantId
    authorizations: [
      {
        principalId: operationsGroupId
        principalIdDisplayName: 'MSP Operations Team'
        roleDefinitionId: 'b24988ac-6180-42a0-ab88-20f7382dd24c' // Contributor
      }
      {
        principalId: securityGroupId
        principalIdDisplayName: 'MSP Security Analysts'
        roleDefinitionId: '39bc4728-0917-49c7-9d2c-d95423bc2eb4' // Security Reader
        justInTimeAccessPolicy: {
          multiFactorAuthProvider: 'Azure'
          maximumActivationDuration: 'PT4H'
          managedByTenantApprovers: [
            {
              principalId: operationsGroupId
              principalIdDisplayName: 'MSP Operations Team'
            }
          ]
        }
      }
    ]
  }
}

resource registrationAssignment 'Microsoft.ManagedServices/registrationAssignments@2022-10-01' = {
  name: assignmentName
  properties: {
    registrationDefinitionId: registrationDef.id
  }
}

output delegationId string = registrationDef.id
```

Deploy in customer tenant:

```bash
az deployment sub create \
  --location eastus \
  --template-file lighthouse.bicep \
  --parameters managedByTenantId=<MSP-TENANT-ID> \
               operationsGroupId=<OPS-GROUP-ID> \
               securityGroupId=<SEC-GROUP-ID>
```

---

## az managedservices CLI Reference

```bash
# List all delegations (run in managing tenant)
az managedservices assignment list --output table

# List definitions in a specific subscription
az managedservices definition list --subscription <SUB-ID>

# Show a specific assignment
az managedservices assignment show --assignment <ASSIGNMENT-ID>

# Delete an assignment
az managedservices assignment delete --assignment <ASSIGNMENT-ID>

# Create definition imperatively (rare — prefer ARM/Bicep)
az managedservices definition create \
  --name "Contoso MSP" \
  --managed-by-tenant-id <MSP-TENANT-ID> \
  --authorizations "<PRINCIPAL-ID>:b24988ac-6180-42a0-ab88-20f7382dd24c"
```

---

## Security Best Practices for MSPs

- **Use groups, not users** as principals — decouple Lighthouse grants from
  individual staff so onboarding/offboarding requires no customer action.
- **Least privilege** — grant Reader for monitoring tasks; reserve Contributor
  only for operations groups, and gate it behind PIM eligible authorizations.
- **MFA on eligible activations** — set `multiFactorAuthProvider: "Azure"` on
  all JIT eligible authorizations.
- **Separate duty groups** — SOC analysts (Security Reader), operations engineers
  (Contributor via JIT), billing reviewers (Billing Reader).
- **Audit regularly** — use Azure Activity Log and Resource Graph to audit
  actions taken by MSP principals in customer subscriptions.
- **No permanent Contributor** — standing Contributor access in production
  customer subscriptions is a security antipattern; use eligible authorizations.
- **Customer notification** — send customers the registration definition name
  and description; encourage them to review `Lighthouse → Service providers`
  blade in the portal periodically.
- **Marketplace offers over manual templates** — Marketplace offers are
  versioned and provide a clear audit trail of what was accepted and when.
