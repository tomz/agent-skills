---
name: ibm-cli
description: IBM Cloud CLI (ibmcloud) — login, targeting, IAM, plugins, resource management, output formats
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

# IBM Cloud CLI Skill

Comprehensive guide for the `ibmcloud` CLI — authentication, targeting, IAM, plugins, and resource management.

---

## Installation & Updates

```bash
# Install (Linux/macOS)
curl -fsSL https://clis.cloud.ibm.com/install/linux | sh

# Update CLI and all installed plugins
ibmcloud update
ibmcloud plugin update --all

# Check version
ibmcloud version

# List installed plugins
ibmcloud plugin list
```

---

## Login & Authentication

### Interactive login (federated/SSO — most common in enterprise)
```bash
ibmcloud login --sso
# Opens browser for SSO; paste one-time passcode back in terminal
```

### API key login (CI/CD pipelines, automation)
```bash
# Login with API key directly
ibmcloud login --apikey "$IBMCLOUD_API_KEY" -r us-south

# From a file
ibmcloud login --apikey @~/.ibmcloud/apikey.json -r us-south

# Non-interactive (no prompts — essential for scripts)
ibmcloud login --apikey "$IBMCLOUD_API_KEY" -r us-south -g my-resource-group --no-region
```

### Service ID login (preferred for automated workloads)
```bash
# A service ID has its own API key — treat like a service account
ibmcloud login --apikey "$SERVICE_ID_APIKEY"
```

### Logout
```bash
ibmcloud logout
```

### Check current session
```bash
ibmcloud target          # Shows account, region, resource group, CF org/space
ibmcloud account show    # Detailed account info
```

---

## Targeting (Account, Region, Resource Group)

Targeting sets the implicit context for all subsequent commands.

```bash
# Set region
ibmcloud target -r us-south      # Dallas
ibmcloud target -r us-east       # Washington DC
ibmcloud target -r eu-de         # Frankfurt
ibmcloud target -r eu-gb         # London
ibmcloud target -r jp-tok        # Tokyo
ibmcloud target -r au-syd        # Sydney

# Set resource group
ibmcloud target -g my-resource-group
ibmcloud target -g Default

# Set both at once
ibmcloud target -r us-south -g production

# List available resource groups
ibmcloud resource groups

# List regions
ibmcloud regions
```

**Gotcha:** Forgetting to set the resource group is the #1 cause of "resource not found" errors.
Always confirm with `ibmcloud target` before running resource commands.

---

## Plugins — Essential Installs

```bash
# Kubernetes / OpenShift
ibmcloud plugin install container-service   # iks, oc cluster management
ibmcloud plugin install container-registry  # icr — image registry

# Code Engine (serverless)
ibmcloud plugin install code-engine

# Cloud Databases
ibmcloud plugin install cloud-databases     # cdb

# Secrets Manager
ibmcloud plugin install secrets-manager

# VPC Infrastructure
ibmcloud plugin install vpc-infrastructure  # is

# Functions (OpenWhisk)
ibmcloud plugin install cloud-functions

# Verify a plugin
ibmcloud ks init   # container-service namespace check
```

---

## Resource Management

```bash
# List all resources in current account/group
ibmcloud resource service-instances
ibmcloud resource service-instances --type all   # includes platform services

# Filter by service
ibmcloud resource service-instances --service-name databases-for-postgresql

# Create a resource (generic pattern)
ibmcloud resource service-instance-create NAME SERVICE-NAME PLAN REGION \
  -g RESOURCE-GROUP \
  -p '{"key": "value"}'   # service-specific parameters as JSON

# Get details of an instance
ibmcloud resource service-instance my-instance

# Delete
ibmcloud resource service-instance-delete my-instance -f   # -f skips confirmation

# List service keys (credentials)
ibmcloud resource service-keys --instance-name my-instance

# Create credentials
ibmcloud resource service-key-create my-key Writer \
  --instance-name my-instance

# Show credentials (contains connection strings, API keys, etc.)
ibmcloud resource service-key my-key

# Extract credential as JSON
ibmcloud resource service-key my-key --output json | jq '.credentials'
```

---

## IAM — Identity & Access Management

### API Keys
```bash
# List your API keys
ibmcloud iam api-keys

# Create a user API key
ibmcloud iam api-key-create my-key -d "CI pipeline key" --file ~/my-key.json

# Create API key for a service ID
ibmcloud iam service-api-key-create my-svc-key my-service-id \
  -d "Automation key" --file ~/svc-key.json

# Delete API key
ibmcloud iam api-key-delete my-key -f
```

### Service IDs
```bash
# List service IDs
ibmcloud iam service-ids

# Create a service ID
ibmcloud iam service-id-create my-service-id -d "App workload identity"

# Get service ID details
ibmcloud iam service-id my-service-id

# Delete
ibmcloud iam service-id-delete my-service-id -f
```

### Access Groups
```bash
# List access groups
ibmcloud iam access-groups

# Create
ibmcloud iam access-group-create developers -d "Developer team"

# Add user to group
ibmcloud iam access-group-user-add developers user@example.com

# Add service ID to group
ibmcloud iam access-group-service-id-add developers my-service-id

# List members
ibmcloud iam access-group-users developers
ibmcloud iam access-group-service-ids developers

# Delete group
ibmcloud iam access-group-delete developers -f
```

### Policies
```bash
# Assign policy to user (Writer role on a specific service instance)
ibmcloud iam user-policy-create user@example.com \
  --roles Writer \
  --service-name databases-for-postgresql \
  --service-instance my-pg-instance

# Assign policy to access group (Viewer on all resources in resource group)
ibmcloud iam access-group-policy-create developers \
  --roles Viewer \
  --resource-group-name production

# List policies for a user
ibmcloud iam user-policies user@example.com

# List policies for an access group
ibmcloud iam access-group-policies developers

# Assign platform + service roles together
ibmcloud iam access-group-policy-create operators \
  --roles Administrator,Manager \
  --service-name containers-kubernetes
```

**Common IAM roles:**
- Platform: Viewer, Operator, Editor, Administrator
- Service: Reader, Writer, Manager (vary by service)

---

## Output Formats & Scripting

```bash
# JSON output (pipe to jq for scripting)
ibmcloud resource service-instances --output json

# YAML output
ibmcloud resource service-instances --output yaml

# Extract specific field with jq
ibmcloud resource service-instances --output json | \
  jq -r '.[] | select(.name=="my-instance") | .id'

# Quiet mode (IDs only, useful in scripts)
ibmcloud resource service-instances -q

# Suppress update check in scripts
IBMCLOUD_SKIP_VERSION_CHECK=true ibmcloud resource service-instances --output json
```

---

## Common Patterns & Guardrails

### Pattern: Get CRN of a resource
```bash
CRN=$(ibmcloud resource service-instance my-instance --output json | jq -r '.[0].crn')
echo "$CRN"
```

### Pattern: Wait for resource to become active
```bash
while true; do
  STATE=$(ibmcloud resource service-instance my-instance --output json | jq -r '.[0].state')
  [ "$STATE" = "active" ] && break
  echo "State: $STATE — waiting..."
  sleep 10
done
```

### Pattern: Iterate over all instances of a type
```bash
ibmcloud resource service-instances \
  --service-name kms --output json | \
  jq -r '.[].name' | while read -r name; do
    echo "Processing: $name"
  done
```

### Guardrails
- **Never commit API keys** — use `ibmcloud iam api-key-create --file` and store in a secrets manager.
- **Use service IDs over user API keys** for automation — user keys are tied to individuals.
- **Scope IAM policies tightly** — prefer resource-group–scoped policies over account-wide ones.
- **Always set `-g` (resource group) and `-r` (region)** in scripts — don't rely on ambient `ibmcloud target`.
- **`-f` flag skips confirmation** — only use in scripts when you're certain about the target.
- **Session tokens expire** — API key login is preferred in automation; SSO tokens expire ~1 hour.
- **Plugin versions drift** — run `ibmcloud plugin update --all` before troubleshooting weird CLI behavior.
