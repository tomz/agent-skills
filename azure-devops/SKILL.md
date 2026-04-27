---
name: azure-devops
description: Azure DevOps Pipelines (YAML), Repos, Artifacts, GitHub Actions for Azure, ACR, blue-green/canary deployments, environments and approvals
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

# Azure DevOps Skills

## Azure Pipelines — YAML Structure

```yaml
# azure-pipelines.yml — full structure reference
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - docs/**
      - '**/*.md'

pr:
  branches:
    include:
      - main

variables:
  - group: my-variable-group          # from Pipelines > Library
  - name: IMAGE_TAG
    value: $(Build.BuildId)
  - name: ACR_NAME
    value: myregistry

pool:
  vmImage: ubuntu-latest              # Microsoft-hosted; or use self-hosted

stages:
  - stage: Build
    displayName: Build & Test
    jobs:
      - job: BuildJob
        steps:
          - checkout: self
            fetchDepth: 0            # full history for semantic versioning

          - task: UsePythonVersion@0
            inputs:
              versionSpec: '3.11'

          - script: |
              pip install -r requirements.txt
              pytest tests/ --junitxml=results.xml
            displayName: Run Tests

          - task: PublishTestResults@2
            inputs:
              testResultsFiles: results.xml
            condition: succeededOrFailed()

          - task: Docker@2
            displayName: Build and Push Docker Image
            inputs:
              command: buildAndPush
              repository: $(ACR_NAME).azurecr.io/myapp
              dockerfile: Dockerfile
              tags: |
                $(IMAGE_TAG)
                latest
              containerRegistry: my-acr-service-connection

  - stage: DeployStaging
    displayName: Deploy to Staging
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployStaging
        displayName: Deploy to Staging Slot
        environment: staging                  # links to Environment in Azure Pipelines
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: my-azure-service-connection
                    appType: webAppLinux
                    appName: myapp-unique
                    deployToSlotOrASE: true
                    slotName: staging
                    package: $(Pipeline.Workspace)/drop/*.zip

  - stage: DeployProd
    displayName: Deploy to Production
    dependsOn: DeployStaging
    jobs:
      - deployment: SwapSlots
        environment: production               # has approval gate configured
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureAppServiceManage@0
                  inputs:
                    azureSubscription: my-azure-service-connection
                    Action: Swap Slots
                    WebAppName: myapp-unique
                    ResourceGroupName: myRG
                    SourceSlot: staging
```

## Pipeline Templates (Reusable)

```yaml
# templates/docker-build.yml
parameters:
  - name: imageName
    type: string
  - name: dockerfile
    type: string
    default: Dockerfile
  - name: buildContext
    type: string
    default: .

steps:
  - task: Docker@2
    displayName: Build Image
    inputs:
      command: build
      repository: ${{ parameters.imageName }}
      dockerfile: ${{ parameters.dockerfile }}
      buildContext: ${{ parameters.buildContext }}
      tags: $(Build.BuildId)

# azure-pipelines.yml — consuming the template
steps:
  - template: templates/docker-build.yml
    parameters:
      imageName: myregistry.azurecr.io/myapp
      dockerfile: src/Dockerfile
```

## Service Connections

Service connections authenticate Azure Pipelines to Azure resources.

```bash
# Create via CLI (OIDC federated — no secret rotation needed)
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id <app-id> \
  --azure-rm-subscription-id <sub-id> \
  --azure-rm-subscription-name "My Subscription" \
  --azure-rm-tenant-id <tenant-id> \
  --name my-azure-service-connection \
  --org https://dev.azure.com/myorg \
  --project myproject

# Workload identity federation (preferred — no secrets)
az devops service-endpoint azurerm create \
  --azure-rm-workload-identity-federation \  
  --azure-rm-subscription-id <sub-id> \
  --name my-oidc-connection \
  --org https://dev.azure.com/myorg \
  --project myproject
```

## Pipeline Variables & Secrets

```yaml
# Inline variable
variables:
  MY_VAR: "value"

# Secret variable (masked in logs)
# Set in pipeline UI > Variables > mark as Secret
# Reference the same way:
- script: echo "$(MY_SECRET)"  # will show ***

# Runtime variable (set during pipeline run)
- script: |
    echo "##vso[task.setvariable variable=COMPUTED_VALUE;isOutput=true]$(date +%Y%m%d)"
  name: setVar

# Reference output variable from another job
- script: echo "$(jobs.BuildJob.outputs['setVar.COMPUTED_VALUE'])"
```

## Azure Artifacts

```bash
# Create feed
az artifacts universal publish \
  --organization https://dev.azure.com/myorg \
  --project myproject \
  --scope project \
  --feed myfeed \
  --name mypackage \
  --version 1.0.0 \
  --path ./dist/

# Publish Python package to Artifacts feed
# In azure-pipelines.yml:
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: myproject/myfeed

- script: |
    pip install twine
    twine upload -r myfeed --config-file $(PYPIRC_PATH) dist/*
  displayName: Publish to Artifacts

# Install from Artifacts feed
- task: PipAuthenticate@1
  inputs:
    artifactFeeds: myproject/myfeed

- script: pip install mypackage --index-url $(PIP_EXTRA_INDEX_URL)
```

## GitHub Actions for Azure

### Authentication with OIDC (Recommended — No Secrets)
```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-unique
          slot-name: production
          package: .
```

### Common GitHub Actions for Azure
```yaml
# Build and push to ACR
- name: Build and Push to ACR
  uses: azure/docker-login@v2
  with:
    login-server: myregistry.azurecr.io
    username: ${{ secrets.ACR_USERNAME }}
    password: ${{ secrets.ACR_PASSWORD }}

- run: |
    docker build -t myregistry.azurecr.io/myapp:${{ github.sha }} .
    docker push myregistry.azurecr.io/myapp:${{ github.sha }}

# Deploy to AKS
- name: Set AKS context
  uses: azure/aks-set-context@v4
  with:
    resource-group: myRG
    cluster-name: myAKS

- name: Deploy to AKS
  run: kubectl apply -f k8s/

# Deploy Bicep
- name: Deploy Bicep
  uses: azure/arm-deploy@v2
  with:
    resourceGroupName: myRG
    template: ./infra/main.bicep
    parameters: ./infra/main.parameters.json

# Run Azure CLI commands
- name: Azure CLI script
  uses: azure/cli@v2
  with:
    azcliversion: latest
    inlineScript: |
      az vm list -g myRG --output table
      az storage account show -n mystorageacct -g myRG
```

### Reusable Workflows
```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      app-name:
        required: true
        type: string
    secrets:
      AZURE_CLIENT_ID:
        required: true
      AZURE_TENANT_ID:
        required: true
      AZURE_SUBSCRIPTION_ID:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ inputs.app-name }}

# Calling workflow:
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      app-name: myapp-staging
    secrets: inherit
```

## Azure Container Registry (ACR)

```bash
# Create ACR (Premium for geo-replication, private endpoints)
az acr create \
  --name myregistry \
  --resource-group myRG \
  --sku Premium \
  --admin-enabled false \
  --location eastus

# Enable geo-replication
az acr replication create \
  --registry myregistry \
  --location westeurope

# Login (for local dev)
az acr login --name myregistry

# Build image in ACR (no local Docker needed)
az acr build \
  --registry myregistry \
  --image myapp:$(git rev-parse --short HEAD) \
  --file Dockerfile \
  .

# Import image from Docker Hub
az acr import \
  --name myregistry \
  --source docker.io/library/nginx:alpine \
  --image nginx:alpine

# List repositories and tags
az acr repository list --name myregistry --output table
az acr repository show-tags --name myregistry --repository myapp --output table

# Delete old tags (keep last 5)
az acr repository show-manifests \
  --name myregistry \
  --repository myapp \
  --orderby time_asc \
  --query "[:-5].digest" \
  --output tsv | \
  xargs -I {} az acr repository delete \
    --name myregistry \
    --image myapp@{} \
    --yes

# Attach ACR to AKS (grants AcrPull role to kubelet identity)
az aks update \
  --resource-group myRG \
  --name myAKS \
  --attach-acr myregistry

# Enable Tasks (scheduled image scanning, base image updates)
az acr task create \
  --registry myregistry \
  --name buildOnCommit \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git \
  --file Dockerfile \
  --git-access-token <token>
```

## Deployment Strategies

### Blue-Green (App Service Slots)
```bash
# Current production is "blue" (production slot)
# Deploy new version to "green" (staging slot)
az webapp deployment slot swap \
  --resource-group myRG \
  --name myapp \
  --slot staging \
  --target-slot production
# Swap is near-instant; rollback = swap back
```

### Canary (weighted traffic split)
```yaml
# Azure Front Door — route 10% to new version
# Configure in Bicep:
resource originGroup 'Microsoft.Cdn/profiles/originGroups@2023-05-01' = {
  ...
}
# Set weights: old=900, new=100 (10% canary)
```

### Rolling (AKS)
```yaml
# k8s Deployment — rolling update is default
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # one extra pod during update
      maxUnavailable: 0   # no downtime
  template:
    spec:
      containers:
        - name: myapp
          image: myregistry.azurecr.io/myapp:v2.0
```

## Environments & Approvals

### Azure Pipelines
```bash
# Create environment via CLI
az devops environment create \
  --name production \
  --project myproject \
  --org https://dev.azure.com/myorg

# Approvals are configured in the UI:
# Project Settings > Pipelines > Environments > production > Approvals and Checks
# Add "Approvals" check, set approvers, timeout, instructions
```

### GitHub Actions
```yaml
# github.com > Settings > Environments > production
# Add required reviewers, wait timer, restrict to main branch

jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.azurewebsites.net
    # Pipeline pauses here until approved
    steps:
      - name: Deploy
        run: echo "Deploying to production..."
```

## Gotchas

- **OIDC vs client secret**: Federated credentials (OIDC) don't expire. Client secrets expire (1-2 years) and rotation is manual. Always prefer OIDC for CI/CD.
- **Service connection scope**: Grant service connections to resource groups, not subscriptions, when possible.
- **ACR admin account**: Disable admin account (`--admin-enabled false`). Use service principals or managed identity for CI/CD and AKS.
- **Slot warm-up**: App Service swap performs a warm-up request before swap. Configure `applicationInitialization` in web.config or app settings for custom health endpoints.
- **Pipeline YAML `condition`**: `succeeded()` only checks the immediately preceding step. Use `and(succeeded('StepName'), ...)` for specific step dependencies.
- **GitHub Actions OIDC**: The federated credential subject must exactly match the workflow context (`repo:org/repo:ref:refs/heads/main` for main branch, `repo:org/repo:environment:production` for environments).
- **ACR task limits**: Free tier allows 100 builds/month. Use Basic+ for production.
- **`isOutput=true` in pipeline variables**: Required for cross-job variable passing. Without it, the variable is only available within the same job.
