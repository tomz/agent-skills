---
name: azure-security
description: Azure security — Microsoft Entra ID, RBAC, Key Vault, Managed Identity, Defender for Cloud, Sentinel, Conditional Access, PIM, Azure Policy
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

# Azure Security Skills

## Microsoft Entra ID (formerly Azure Active Directory)

### Users & Groups
```bash
# Create user
az ad user create \
  --display-name "Jane Doe" \
  --user-principal-name jane.doe@contoso.com \
  --password "TempPass123!" \
  --force-change-password-next-sign-in true

# Get user info
az ad user show --id jane.doe@contoso.com
az ad user list --filter "startswith(displayName,'Jane')" --output table

# Create group
az ad group create \
  --display-name "DevOps Engineers" \
  --mail-nickname devops-engineers

# Add member to group
az ad group member add \
  --group "DevOps Engineers" \
  --member-id <user-object-id>

# Check if user is in group
az ad group member check \
  --group "DevOps Engineers" \
  --member-id <user-object-id>

# List group members
az ad group member list --group "DevOps Engineers" --output table
```

### App Registrations & Service Principals
```bash
# Create app registration
az ad app create \
  --display-name "MyApp" \
  --sign-in-audience AzureADMyOrg

# Create service principal for the app
az ad sp create --id <app-id>

# Create SP with RBAC in one step (creates both app + SP)
az ad sp create-for-rbac \
  --name "my-github-actions-sp" \
  --role Contributor \
  --scopes /subscriptions/<sub-id>/resourceGroups/myRG \
  --json-auth   # output federated credentials format for GitHub Actions

# Add client secret (expires in 1 year by default)
az ad app credential reset \
  --id <app-id> \
  --years 1 \
  --display-name "CI Secret"

# Add API permission (e.g., Microsoft Graph User.Read)
az ad app permission add \
  --id <app-id> \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope   # User.Read

# Grant admin consent
az ad app permission admin-consent --id <app-id>

# Federated credentials (for GitHub Actions — no secrets needed)
az ad app federated-credential create \
  --id <app-id> \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

## Role-Based Access Control (RBAC)

### Built-in Roles (Most Used)
```
Owner                        → Full access including role assignments
Contributor                  → Full resource management, no role assignments
Reader                       → Read-only
User Access Administrator    → Manage role assignments only
Storage Blob Data Contributor → Read/write/delete blobs (separate from account management)
Storage Blob Data Reader      → Read blobs
Key Vault Secrets Officer     → Get/set/delete secrets
Key Vault Secrets User        → Read secrets only
AcrPull                       → Pull from Azure Container Registry
AcrPush                       → Push to Azure Container Registry
Monitoring Contributor        → Create/manage Monitor resources
Network Contributor           → Manage network resources
Virtual Machine Contributor   → Manage VMs (not VNet or storage)
AKS Cluster Admin Role        → Admin kubeconfig access
AKS Cluster User Role         → User kubeconfig access
```

### Assigning Roles
```bash
# Assign role to user
az role assignment create \
  --assignee jane.doe@contoso.com \
  --role "Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG

# Assign role to group
az role assignment create \
  --assignee-object-id <group-object-id> \
  --assignee-principal-type Group \
  --role "Reader" \
  --scope /subscriptions/<sub-id>

# Assign role to managed identity
az role assignment create \
  --assignee <managed-identity-client-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/myaccount

# List role assignments for a resource group
az role assignment list \
  --resource-group myRG \
  --output table

# List role assignments for a user
az role assignment list \
  --assignee jane.doe@contoso.com \
  --all \
  --output table

# Remove role assignment
az role assignment delete \
  --assignee jane.doe@contoso.com \
  --role "Contributor" \
  --resource-group myRG
```

### Custom Roles
```bash
# Create custom role definition
cat > custom-role.json << 'EOF'
{
  "Name": "VM Operator",
  "Description": "Can start/stop/restart VMs, read everything",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/*/read",
    "Microsoft.Resources/*/read",
    "Microsoft.Network/*/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<sub-id>"
  ]
}
EOF

az role definition create --role-definition custom-role.json

# Update custom role
az role definition update --role-definition custom-role.json

# List custom roles
az role definition list --custom-role-only true --output table
```

### RBAC Scope Hierarchy
```
Management Group   /providers/Microsoft.Management/managementGroups/<mg>
  └── Subscription /subscriptions/<sub>
        └── Resource Group /subscriptions/<sub>/resourceGroups/<rg>
              └── Resource  /subscriptions/<sub>/resourceGroups/<rg>/providers/<type>/<name>
```
Roles assigned at higher scope are inherited by all child scopes.

## Azure Key Vault

```bash
# Create Key Vault (soft-delete + purge protection for production)
az keyvault create \
  --name myKV-unique \
  --resource-group myRG \
  --location eastus \
  --sku standard \
  --enable-rbac-authorization true \
  --enable-soft-delete true \
  --soft-delete-retention-days 90 \
  --enable-purge-protection true

# RBAC on Key Vault (preferred over access policies)
az role assignment create \
  --assignee jane.doe@contoso.com \
  --role "Key Vault Secrets Officer" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV-unique

# Secrets
az keyvault secret set --vault-name myKV-unique --name MySecret --value "s3cr3tV@lue"
az keyvault secret show --vault-name myKV-unique --name MySecret --query value -o tsv
az keyvault secret list --vault-name myKV-unique --output table
az keyvault secret delete --vault-name myKV-unique --name MySecret
az keyvault secret recover --vault-name myKV-unique --name MySecret   # undo delete
az keyvault secret set-attributes --vault-name myKV-unique --name MySecret \
  --expires 2025-12-31T00:00:00Z   # expiry

# Keys (for encryption)
az keyvault key create \
  --vault-name myKV-unique \
  --name myKey \
  --kty RSA \
  --size 4096 \
  --ops encrypt decrypt sign verify

az keyvault key list --vault-name myKV-unique --output table

# Certificates
az keyvault certificate create \
  --vault-name myKV-unique \
  --name myCert \
  --policy "$(az keyvault certificate get-default-policy)"

# Download certificate
az keyvault certificate download \
  --vault-name myKV-unique \
  --name myCert \
  --file cert.pem \
  --encoding PEM

# Key Vault diagnostic logs (audit all access)
az monitor diagnostic-settings create \
  --name KVAuditLogs \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV-unique \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --workspace <log-analytics-workspace-id>
```

### Key Vault Network Security
```bash
# Restrict to specific VNet + your IP only
az keyvault update \
  --name myKV-unique \
  --resource-group myRG \
  --default-action Deny \
  --bypass AzureServices

az keyvault network-rule add \
  --name myKV-unique \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet AppSubnet

az keyvault network-rule add \
  --name myKV-unique \
  --resource-group myRG \
  --ip-address 1.2.3.4/32
```

## Managed Identity

### System-Assigned (tied to resource lifecycle)
```bash
# Enable system-assigned MI on a VM
az vm identity assign -g myRG -n myVM

# Enable on App Service
az webapp identity assign -g myRG -n myapp-unique

# Enable on Function App
az functionapp identity assign -g myRG -n myfuncapp

# Get the principal ID (for role assignments)
az vm show -g myRG -n myVM \
  --query "identity.principalId" -o tsv
```

### User-Assigned (reusable, independent lifecycle)
```bash
# Create user-assigned MI
az identity create \
  --name myMI \
  --resource-group myRG \
  --location eastus

# Get client ID and principal ID
az identity show -n myMI -g myRG \
  --query "{clientId:clientId, principalId:principalId}" -o json

# Assign to a VM
az vm identity assign \
  -g myRG -n myVM \
  --identities /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myMI

# Assign role to the MI
az role assignment create \
  --assignee <principal-id> \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/myacct
```

### Using MI in Code
```python
# Python — Azure SDK uses DefaultAzureCredential
# Works locally (az login), in CI (env vars), and in Azure (managed identity)
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://myKV-unique.vault.azure.net/", credential=credential)
secret = client.get_secret("MySecret")
```

## Microsoft Defender for Cloud

```bash
# Enable Defender plans
az security pricing create \
  --name VirtualMachines \
  --tier Standard   # Standard = paid Defender; Free = basic policy only

az security pricing create --name SqlServers --tier Standard
az security pricing create --name Containers --tier Standard
az security pricing create --name StorageAccounts --tier Standard

# Get secure score
az security secure-score-controls list --output table

# List recommendations
az security task list --output table

# Enable auto-provisioning of agents
az security auto-provisioning-setting update \
  --name mma \
  --auto-provision on
```

## Microsoft Sentinel

```bash
# Create Log Analytics workspace (Sentinel requires one)
az monitor log-analytics workspace create \
  --resource-group myRG \
  --workspace-name mySentinelWS \
  --location eastus

# Enable Sentinel on workspace
az sentinel onboarding-state create \
  --resource-group myRG \
  --workspace-name mySentinelWS \
  --name default

# List data connectors
az sentinel data-connector list \
  --resource-group myRG \
  --workspace-name mySentinelWS \
  --output table

# List analytics rules
az sentinel alert-rule list \
  --resource-group myRG \
  --workspace-name mySentinelWS \
  --output table
```

## Azure Policy

```bash
# List built-in policy definitions
az policy definition list --query "[?policyType=='BuiltIn'].{Name:name, DisplayName:displayName}" -o table | head -30

# Assign a built-in policy (e.g., require TLS 1.2 on storage accounts)
az policy assignment create \
  --name RequireTLS12 \
  --policy /providers/Microsoft.Authorization/policyDefinitions/<policy-id> \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG \
  --enforcement-mode Default   # Default=enforce; DoNotEnforce=audit only

# Assign with audit effect (non-destructive — just report)
az policy assignment create \
  --name AuditPublicBlobs \
  --policy /providers/Microsoft.Authorization/policyDefinitions/<policy-id> \
  --scope /subscriptions/<sub-id> \
  --params '{"effect":{"value":"Audit"}}'

# Create policy initiative (group of policies)
az policy set-definition create \
  --name "CIS Benchmark Baseline" \
  --definitions @policy-set.json

# List compliance state
az policy state list \
  --resource-group myRG \
  --filter "complianceState eq 'NonCompliant'" \
  --output table

# Trigger compliance scan
az policy state trigger-scan --resource-group myRG
```

## Conditional Access (via Microsoft Graph API)

Conditional Access policies must be managed via Entra ID portal or MS Graph API.

```bash
# Get existing CA policies via CLI (requires Graph API)
az rest \
  --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --headers "Content-Type=application/json"

# Common CA Policy patterns:
# 1. Require MFA for all users
# 2. Block legacy auth protocols (SMTP AUTH, IMAP, POP3)
# 3. Require compliant device for sensitive apps
# 4. Block access from risky sign-in locations
# 5. Require MFA for Azure portal access
# 6. Session controls for unmanaged devices (no download, watermark)
```

## Privileged Identity Management (PIM)

PIM provides just-in-time (JIT) privileged access. Managed primarily through portal or Graph API.

```bash
# List eligible role assignments (requires MS Graph)
az rest \
  --method GET \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilitySchedules" \
  --headers "Content-Type=application/json"

# Activate an eligible role assignment (self-activation)
az rest \
  --method POST \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignmentScheduleRequests" \
  --body '{
    "action": "selfActivate",
    "principalId": "<your-object-id>",
    "roleDefinitionId": "<role-def-id>",
    "directoryScopeId": "/",
    "justification": "Need to deploy production release",
    "scheduleInfo": {
      "startDateTime": "2024-01-15T00:00:00Z",
      "expiration": { "type": "AfterDuration", "duration": "PT4H" }
    }
  }'
```

## Security Best Practices

1. **Never use account keys for Storage** — use Managed Identity + RBAC (`Storage Blob Data Contributor`).
2. **Never store secrets in code or environment variables** — use Key Vault references in App Service/Functions.
3. **Enable Key Vault purge protection** for production — prevents accidental permanent deletion.
4. **Use User-Assigned MI** when identity needs to be shared across resources or persists beyond a single resource.
5. **Scope RBAC as narrowly as possible** — resource level, not subscription level, when feasible.
6. **Enable Defender for Cloud** on all subscriptions — free tier provides basic security posture.
7. **Require MFA via Conditional Access** — don't rely on per-user MFA settings (legacy).
8. **Use PIM for privileged roles** — no permanent Owner/Contributor assignments for humans.
9. **Enable diagnostic logs on Key Vault** — audit all secret access via Log Analytics.
10. **Federated credentials over client secrets** — for GitHub Actions and other OIDC-capable CI systems.

## Gotchas

- **`--enable-rbac-authorization true` on Key Vault**: Switches from legacy Access Policies to RBAC. Cannot mix both modes easily. New vaults should always use RBAC.
- **Soft delete is now mandatory**: Key Vault soft-delete cannot be disabled on new vaults. Purge protection is optional but recommended for production.
- **RBAC propagation**: Role assignments can take up to 5 minutes to take effect globally.
- **Data plane vs management plane RBAC**: `Contributor` on a storage account gives management access but NOT data access. Need `Storage Blob Data Contributor` for data.
- **Managed Identity in AKS**: Use Workload Identity (OIDC) not the deprecated Pod Identity (aad-pod-identity).
- **Conditional Access and MFA**: CA policies apply to interactive sign-ins. Managed identity tokens are not affected by CA policies.
- **PIM activation**: Requires the user to have an eligible assignment first. Active assignments bypass PIM (always on, no approval).
