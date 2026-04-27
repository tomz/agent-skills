---
name: azure-acr
description: Azure Container Registry — image management, ACR Tasks, geo-replication, vulnerability scanning, OCI artifacts, Helm charts, connected registries, token-based access
license: MIT
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Container Registry (ACR) Skill

## Tier Comparison

| Feature                      | Basic    | Standard  | Premium   |
|-----------------------------|----------|-----------|-----------|
| Storage (GB)                 | 10       | 100       | 500       |
| Webhooks                     | 2        | 10        | 500       |
| Geo-replication              | ✗        | ✗         | ✓         |
| Private endpoints            | ✗        | ✗         | ✓         |
| Dedicated data endpoints     | ✗        | ✗         | ✓         |
| Content trust / quarantine   | ✗        | ✗         | ✓         |
| Connected registries         | ✗        | ✗         | ✓         |
| Customer-managed keys        | ✗        | ✗         | ✓         |
| Throughput (ops/min)         | 1,000    | 10,000    | 100,000   |

Upgrade without downtime: `az acr update --name $REGISTRY --sku Premium`

---

## Setup

```bash
az login && az account set --subscription "My Subscription"
az provider register --namespace Microsoft.ContainerRegistry
REGISTRY=myregistry; RG=myresourcegroup; LOCATION=eastus
```

---

## Registry Lifecycle

```bash
# Create
az acr create --name $REGISTRY --resource-group $RG \
  --location $LOCATION --sku Standard --admin-enabled false
# Premium with zone redundancy
az acr create --name $REGISTRY -g $RG --location $LOCATION \
  --sku Premium --zone-redundancy enabled

# Inspect / update / delete
az acr show --name $REGISTRY -g $RG -o json
az acr list -o table
az acr update --name $REGISTRY --sku Premium
az acr update --name $REGISTRY --public-network-enabled false
az acr delete --name $REGISTRY -g $RG --yes
```

---

## Authentication

```bash
az acr login --name $REGISTRY                    # interactive (Docker cred store)
az acr login --name $REGISTRY --expose-token     # get raw ACR refresh token
docker login $REGISTRY.azurecr.io --username $SP_APP_ID --password $SP_PASSWORD

# Admin account (avoid in production)
az acr update --name $REGISTRY --admin-enabled true
az acr credential show --name $REGISTRY
az acr credential renew --name $REGISTRY --password-name password
```

---

## Image Operations

```bash
# Cloud build + push (no local Docker daemon needed)
az acr build --registry $REGISTRY --image myapp:v1.0 --file Dockerfile .
az acr build --registry $REGISTRY --image myapp:$(git rev-parse --short HEAD) \
  --build-arg ENV=production .

# Local build + push
docker build -t $REGISTRY.azurecr.io/myapp:latest . && docker push $REGISTRY.azurecr.io/myapp:latest

# Multi-platform buildx with registry cache
docker buildx build --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=$REGISTRY.azurecr.io/myapp:buildcache \
  --cache-to   type=registry,ref=$REGISTRY.azurecr.io/myapp:buildcache,mode=max \
  --tag $REGISTRY.azurecr.io/myapp:latest --push .

# Import from external registries
az acr import --name $REGISTRY --source docker.io/library/nginx:alpine --image nginx:alpine
az acr import --name $REGISTRY --source mcr.microsoft.com/dotnet/aspnet:8.0 --image dotnet/aspnet:8.0
az acr import --name $REGISTRY --source sourceregistry.azurecr.io/myapp:v1 --image myapp:v1

# Repository management
az acr repository list          --name $REGISTRY -o table
az acr repository show-tags     --name $REGISTRY --repository myapp --orderby time_desc
az acr repository show-manifests --name $REGISTRY --repository myapp --detail
az acr repository show          --name $REGISTRY --image myapp:latest
az acr repository delete        --name $REGISTRY --image myapp:old-tag --yes
az acr repository delete        --name $REGISTRY --repository myapp --yes
az acr repository untag         --name $REGISTRY --image myapp:canary
```

---

## ACR Tasks

### Triggered Tasks

```bash
# Commit-triggered build
az acr task create --registry $REGISTRY --name build-on-commit \
  --image "myapp:{{.Run.ID}}" \
  --context "https://github.com/myorg/myrepo.git#main" --file Dockerfile \
  --git-access-token "$GITHUB_PAT" --commit-trigger-enabled true

# Base-image-update-triggered build
az acr task create --registry $REGISTRY --name build-on-base-update \
  --image "myapp:{{.Run.ID}}" \
  --context "https://github.com/myorg/myrepo.git" --file Dockerfile \
  --git-access-token "$GITHUB_PAT" --base-image-trigger-enabled true

# Scheduled (cron)
az acr task create --registry $REGISTRY --name nightly-build \
  --image "myapp:nightly" --context "https://github.com/myorg/myrepo.git" \
  --schedule "0 2 * * *" --git-access-token "$GITHUB_PAT"

# Run / manage
az acr task run      --registry $REGISTRY --name build-on-commit
az acr task list     --registry $REGISTRY -o table
az acr task list-runs --registry $REGISTRY --task build-on-commit
az acr task logs     --registry $REGISTRY --run-id aa1b
az acr task delete   --registry $REGISTRY --name nightly-build --yes
```

### Multi-Step Task YAML (`acr-task.yaml`)

```yaml
version: v1.1.0
updated: 2026-04-24
stepTimeout: 600
steps:
  - id: build
    build: >-
      -t {{.Run.Registry}}/myapp:{{.Run.ID}}
      -t {{.Run.Registry}}/myapp:latest
      --build-arg BUILD_DATE={{.Run.Date}} -f Dockerfile .

  - id: test
    cmd: >-
      {{.Run.Registry}}/myapp:{{.Run.ID}}
      python -m pytest tests/ -v --tb=short
    when: ["build"]

  - id: push
    push: ["{{.Run.Registry}}/myapp:{{.Run.ID}}", "{{.Run.Registry}}/myapp:latest"]
    when: ["test"]

  - id: notify
    cmd: >-
      curlimages/curl curl -X POST -H "Content-Type: application/json"
      -d '{"text":"Build {{.Run.ID}} pushed"}' $SLACK_WEBHOOK
    when: ["push"]
    envs: [SLACK_WEBHOOK]
```

```bash
az acr run --registry $REGISTRY --file acr-task.yaml \
  --set ENV=production --secret SLACK_WEBHOOK="https://hooks.slack.com/..." .
az acr task create --registry $REGISTRY --name multi-step \
  --context "https://github.com/myorg/myrepo.git" \
  --file acr-task.yaml --git-access-token "$GITHUB_PAT"
```

---

## Geo-Replication (Premium)

```bash
az acr replication create --registry $REGISTRY --location westeurope --zone-redundancy enabled
az acr replication create --registry $REGISTRY --location southeastasia
az acr replication list   --registry $REGISTRY -o table
az acr replication show   --registry $REGISTRY --name westeurope
az acr replication delete --registry $REGISTRY --name southeastasia --yes
```

> Clients route automatically to the nearest healthy replica — no client changes needed.

---

## Vulnerability Scanning

```bash
# Enable Microsoft Defender for Containers (scans on push automatically)
az security pricing create --name Containers --tier Standard

# View CVE findings
az security assessment list -g $RG \
  --query "[?contains(name,'containerRegistryVulnerabilities')]"

# Quarantine policy — blocks unscanned/failing images from pull (Premium)
az acr config content-trust update --registry $REGISTRY --status enabled
```

---

## OCI Artifacts (ORAS)

```bash
# Install ORAS CLI
curl -sLO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
tar -xzf oras_1.1.0_linux_amd64.tar.gz && sudo mv oras /usr/local/bin/
oras login $REGISTRY.azurecr.io --username $SP_APP_ID --password $SP_PASSWORD

# Push arbitrary OCI artifact
oras push $REGISTRY.azurecr.io/myartifact:v1 \
  --artifact-type application/vnd.my.artifact ./config.json:application/json

# Attach SBOM/attestation to existing image
oras attach $REGISTRY.azurecr.io/myapp:v1 \
  --artifact-type application/spdx+json ./sbom.spdx.json:application/spdx+json

oras pull     $REGISTRY.azurecr.io/myartifact:v1 -o ./output/
oras discover $REGISTRY.azurecr.io/myapp:v1   # list referrers
```

---

## Helm Charts (OCI-Based, Helm 3.8+)

```bash
helm registry login $REGISTRY.azurecr.io --username $SP_APP_ID --password $SP_PASSWORD
helm package ./mychart --destination ./charts/
helm push    ./charts/mychart-1.0.0.tgz oci://$REGISTRY.azurecr.io/helm
helm pull    oci://$REGISTRY.azurecr.io/helm/mychart --version 1.0.0
helm install myrelease oci://$REGISTRY.azurecr.io/helm/mychart --version 1.0.0
az acr repository show-tags --name $REGISTRY --repository helm/mychart
```

---

## Image Signing — Notation / Notary v2

```bash
# Install Notation
curl -sLO https://github.com/notaryproject/notation/releases/download/v1.0.0/notation_1.0.0_linux_amd64.tar.gz
tar -xzf notation_1.0.0_linux_amd64.tar.gz && sudo mv notation /usr/local/bin/

notation login $REGISTRY.azurecr.io --username $SP_APP_ID --password $SP_PASSWORD
notation sign   $REGISTRY.azurecr.io/myapp:v1 --plugin azure-kv --id "$KEY_ID"
notation verify $REGISTRY.azurecr.io/myapp:v1 --policy ./trustpolicy.json
notation list   $REGISTRY.azurecr.io/myapp:v1
```

**`trustpolicy.json`:**
```json
{
  "version": "1.0",
  "trustPolicies": [{
    "name": "myapp-policy",
    "registryScopes": ["myregistry.azurecr.io/myapp"],
    "signatureVerification": { "level": "strict" },
    "trustStores": ["ca:mystore"],
    "trustedIdentities": ["x509.subject: CN=MyOrg,O=MyOrg,C=US"]
  }]
}
```

---

## Token-Based Access (Scope Maps)

```bash
# Scope maps define per-repository permissions
az acr scope-map create --registry $REGISTRY --name readonly-myapp \
  --repository myapp content/read metadata/read
az acr scope-map create --registry $REGISTRY --name readwrite-myapp \
  --repository myapp content/read content/write content/delete metadata/read metadata/write

# Token scoped to a map + generate credentials
az acr token create --registry $REGISTRY --name ci-token --scope-map readonly-myapp
az acr token credential generate --registry $REGISTRY --name ci-token \
  --password1 --expiration-in-days 90

az acr scope-map list --registry $REGISTRY -o table
az acr token list     --registry $REGISTRY -o table
az acr token update   --registry $REGISTRY --name ci-token --status disabled
az acr token delete   --registry $REGISTRY --name ci-token --yes
```

---

## Managed Identity Authentication

```bash
ACR_ID=$(az acr show --name $REGISTRY -g $RG --query id -o tsv)

# AKS — one-liner attach (auto-assigns AcrPull to kubelet identity)
az aks update --name myaks --resource-group $RG --attach-acr $REGISTRY

# Manual role assignment
AKS_IDENTITY=$(az aks show --name myaks -g $RG \
  --query identityProfile.kubeletidentity.objectId -o tsv)
az role assignment create --assignee "$AKS_IDENTITY" --role AcrPull --scope "$ACR_ID"

# App Service
az webapp identity assign -g $RG -n mywebapp
az role assignment create \
  --assignee "$(az webapp identity show -g $RG -n mywebapp --query principalId -o tsv)" \
  --role AcrPull --scope "$ACR_ID"

# Azure Container Instances
az container create -g $RG --name mycontainer \
  --image $REGISTRY.azurecr.io/myapp:latest \
  --acr-identity "$IDENTITY_ID" --assign-identity "$IDENTITY_ID"
```

---

## Private Endpoint & Dedicated Data Endpoint (Premium)

```bash
az acr update --name $REGISTRY --public-network-enabled false

ACR_ID=$(az acr show --name $REGISTRY -g $RG --query id -o tsv)

az network private-endpoint create -g $RG --name acr-pe \
  --vnet-name myVnet --subnet mySubnet \
  --private-connection-resource-id "$ACR_ID" \
  --group-id registry --connection-name acr-conn

az network private-dns zone create -g $RG --name "privatelink.azurecr.io"
az network private-dns link vnet create -g $RG \
  --zone-name "privatelink.azurecr.io" --name acr-dns-link \
  --virtual-network myVnet --registration-enabled false

# Dedicated per-region data endpoints (minimise firewall FQDN count)
az acr update --name $REGISTRY --data-endpoint-enabled true
az acr show-endpoints --name $REGISTRY
```

---

## Retention & Purge Policies

```bash
# Auto-delete untagged manifests after 30 days (Premium)
az acr config retention update --registry $REGISTRY \
  --status enabled --days 30 --type UntaggedManifests

# Manual purge (dry-run first, then for real)
az acr run --registry $REGISTRY \
  --cmd "acr purge --filter 'myapp:.*' --ago 30d --untagged --dry-run" /dev/null
az acr run --registry $REGISTRY \
  --cmd "acr purge --filter 'myapp:.*' --ago 30d --untagged --keep 5" /dev/null

# Scheduled weekly purge
az acr task create --registry $REGISTRY --name weekly-purge \
  --cmd "acr purge --filter 'myapp:.*' --ago 30d --untagged --keep 5" \
  --schedule "0 1 * * 0" --context /dev/null
```

---

## Cache Rules (Pull-Through Cache)

```bash
az acr credential-set create --registry $REGISTRY --name dockerhub-creds \
  --login-server docker.io \
  --username-secret "dockerhub-username" --password-secret "dockerhub-password"

az acr cache create --registry $REGISTRY --name cache-dockerhub \
  --source-repo "docker.io/*" --target-repo "dockerhub/*" --credential-set dockerhub-creds
az acr cache create --registry $REGISTRY --name cache-mcr \
  --source-repo "mcr.microsoft.com/*" --target-repo "mcr/*"

docker pull $REGISTRY.azurecr.io/dockerhub/library/nginx:alpine  # transparent proxy

az acr cache list   --registry $REGISTRY -o table
az acr cache delete --registry $REGISTRY --name cache-dockerhub --yes
```

---

## Connected Registries (IoT Edge / Offline)

```bash
# Create read-only connected registry syncing 'myapp'
az acr connected-registry create --registry $REGISTRY \
  --name edge-registry --repository myapp --mode ReadOnly

# Get deployment settings for IoT Edge
az acr connected-registry get-settings \
  --registry $REGISTRY --name edge-registry --parent-protocol https

az acr connected-registry list   --registry $REGISTRY -o table
az acr connected-registry show   --registry $REGISTRY --name edge-registry
az acr connected-registry update --registry $REGISTRY \
  --name edge-registry --add-repository anotherapp
```

---

## CI/CD Integration

### GitHub Actions (OIDC)

```yaml
# .github/workflows/build-push.yml
name: Build and Push to ACR
on:
  push: { branches: [main] }
jobs:
  build:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: |
          az acr build --registry myregistry \
            --image myapp:${{ github.sha }} --image myapp:latest --file Dockerfile .
```

### Azure DevOps Pipeline

```yaml
trigger: { branches: { include: [main] } }
variables: { tag: $(Build.BuildId) }
stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        pool: { vmImage: ubuntu-latest }
        steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: MyServiceConnection
              scriptType: bash; scriptLocation: inlineScript
              inlineScript: |
                az acr build --registry myregistry \
                  --image myapp:$(tag) --image myapp:latest --file Dockerfile .
```

---

## Bicep & Terraform

### Bicep (`acr.bicep`)

```bicep
param registryName string
param location string = resourceGroup().location

resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: registryName; location: location
  sku: { name: 'Premium' }
  properties: {
    adminUserEnabled: false; publicNetworkAccess: 'Disabled'; zoneRedundancy: 'Enabled'
    policies: {
      retentionPolicy: { days: 30, status: 'enabled' }
      quarantinePolicy: { status: 'enabled' }
    }
  }
}
resource replication 'Microsoft.ContainerRegistry/registries/replications@2023-07-01' = {
  parent: acr; name: 'westeurope'; location: 'westeurope'
  properties: { zoneRedundancy: 'Enabled' }
}
output loginServer string = acr.properties.loginServer
```

```bash
az deployment group create -g $RG --template-file acr.bicep --parameters registryName=$REGISTRY
```

### Terraform

```hcl
resource "azurerm_container_registry" "acr" {
  name                = var.registry_name
  resource_group_name = var.resource_group
  location            = var.location
  sku                 = "Premium"
  admin_enabled       = false
  georeplications     { location = "West Europe"; zone_redundancy_enabled = true }
  retention_policy    { days = 30; enabled = true }
  trust_policy        { enabled = true }
}
resource "azurerm_role_assignment" "aks_pull" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPull"
  principal_id         = var.aks_kubelet_identity_object_id
}
```

---

## Monitoring: Diagnostics, Metrics, Webhooks

```bash
ACR_ID=$(az acr show --name $REGISTRY -g $RG --query id -o tsv)
WORKSPACE_ID=$(az monitor log-analytics workspace show -g $RG -n myworkspace --query id -o tsv)

# Diagnostic logs to Log Analytics
az monitor diagnostic-settings create --resource "$ACR_ID" --name acr-diag \
  --workspace "$WORKSPACE_ID" \
  --logs '[{"category":"ContainerRegistryRepositoryEvents","enabled":true},
           {"category":"ContainerRegistryLoginEvents","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Metrics
az monitor metrics list --resource "$ACR_ID" --metric StorageUsed --interval PT1H
az monitor metrics list --resource "$ACR_ID" --metric SuccessfulPullCount --interval P1D

# KQL — top repos by pull count:
# ContainerRegistryRepositoryEvents
# | where TimeGenerated > ago(24h) and OperationName == "Pull"
# | summarize Pulls=count() by Repository | order by Pulls desc

# Webhooks
az acr webhook create --registry $REGISTRY --name push-notify \
  --uri "https://myserver.example.com/hook" --actions push delete \
  --headers "Authorization=Bearer $TOKEN"
az acr webhook ping --registry $REGISTRY --name push-notify
az acr webhook list --registry $REGISTRY -o table
az acr webhook delete --registry $REGISTRY --name push-notify --yes
```

---

## Security Best Practices

1. **Disable admin account** — `az acr update --name $REGISTRY --admin-enabled false`; use managed identities or scope-map tokens instead.
2. **Disable public network access** + use private endpoints for production (Premium).
3. **Use Managed Identity** for AKS, ACI, App Service — zero stored credentials.
4. **Least-privilege scope-map tokens** — grant only the repos and actions needed.
5. **Enable Defender for Containers** to auto-scan images on push and receive CVE alerts.
6. **Enable quarantine policy** (Premium) to block unscanned/failing images from pull.
7. **Sign images with Notation** and enforce verification in AKS via Azure Policy.
8. **Enable retention policy** to purge untagged manifests automatically.
9. **Use geo-replication** (Premium) for HA and reduced pull latency across regions.
10. **Audit all events** — route `ContainerRegistryLoginEvents` + `RepositoryEvents` to Log Analytics.
11. **Rotate token credentials** with short expiry: `az acr token credential generate --registry $REGISTRY --name ci-token --password1 --expiration-in-days 30`
12. **Use cache rules** for upstream deps to avoid Docker Hub rate limits.
13. **Pin by digest in production** — `myregistry.azurecr.io/myapp@sha256:<digest>` prevents tag mutations.
14. **Enable zone redundancy** on registry + replications for 99.99% SLA.
15. **Use dedicated data endpoints** (Premium) to minimise firewall FQDN allowlist surface.
