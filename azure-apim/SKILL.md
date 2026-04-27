---
name: azure-apim
description: Azure API Management — APIs, products, subscriptions, policies, developer portal, gateway, OAuth, rate limiting, caching, versioning
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure API Management (APIM) Skill

## Overview

Azure API Management is a fully managed service that acts as a gateway, proxy, and
developer portal for APIs. It enables publishing, securing, transforming, and monitoring
APIs across backends (Azure Functions, App Service, Logic Apps, Kubernetes, on-prem).

---

## APIM Tiers

| Tier | Use Case | SLA | VNet | Scale Units | Self-hosted GW |
|------|----------|-----|------|-------------|----------------|
| **Consumption** | Serverless, dev/test, per-call billing | 99.95% | No | Auto (0→N) | No |
| **Developer** | Non-production, full features | None | Yes (external) | 1 fixed | Yes |
| **Basic** | Low-traffic production | 99.95% | No | 1–2 | No |
| **Standard** | Mid-scale production | 99.95% | No | 1–4 | Yes |
| **Premium** | Enterprise, multi-region | 99.99% | Yes (internal/external) | 1–31 per region | Yes |
| **Basic v2** | Simplified v2 stack, fast provision | 99.95% | Yes (injection) | 1–10 | No |
| **Standard v2** | v2 with VNet injection | 99.95% | Yes (injection) | 1–10 | Yes |

**v2 tiers** provision in under 10 minutes (vs. 30–45 min for classic) and support
subnet injection with no dedicated subnet ownership requirement.

Choose **Premium** for multi-region active-active, internal VNet, availability zones,
and custom gateway capacity. Choose **Consumption** for event-driven/serverless APIs
billed per million calls with zero idle cost.

---

## Creating an APIM Instance

```bash
# Create resource group
az group create --name rg-apim-prod --location eastus

# Create APIM instance (Standard tier)
az apim create \
  --name myapim-prod \
  --resource-group rg-apim-prod \
  --location eastus \
  --sku-name Standard \
  --sku-capacity 1 \
  --publisher-email admin@contoso.com \
  --publisher-name "Contoso API Team"

# Create Consumption tier (no capacity needed)
az apim create \
  --name myapim-dev \
  --resource-group rg-apim-dev \
  --location eastus \
  --sku-name Consumption \
  --publisher-email dev@contoso.com \
  --publisher-name "Contoso Dev"

# Check provisioning state
az apim show --name myapim-prod --resource-group rg-apim-prod \
  --query provisioningState -o tsv
```

---

## Importing and Managing APIs

### Import from OpenAPI (Swagger)

```bash
# Import from URL
az apim api import \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id petstore-api \
  --path petstore \
  --specification-format OpenApi \
  --specification-url https://petstore3.swagger.io/api/v3/openapi.json \
  --display-name "Pet Store API" \
  --protocols https

# Import from local file
az apim api import \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id orders-api \
  --path orders \
  --specification-format OpenApiJson \
  --specification-path ./openapi.json
```

### Import from Azure Function App

```bash
az apim api import \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id func-api \
  --path functions \
  --specification-format OpenApiJson \
  --display-name "Function API"
# Then link the Function App backend via portal or ARM
```

### Import from WSDL (SOAP)

```bash
az apim api import \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id soap-api \
  --path soap \
  --specification-format Wsdl \
  --specification-url https://example.com/service.wsdl \
  --soap-api-type soap  # or 'http' for SOAP-to-REST passthrough
```

### Create a blank REST API

```bash
az apim api create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id inventory-api \
  --path inventory \
  --display-name "Inventory API" \
  --protocols https \
  --service-url https://inventory-backend.azurewebsites.net

# Add an operation
az apim api operation create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id inventory-api \
  --operation-id get-items \
  --display-name "Get Items" \
  --method GET \
  --url-template /items
```

---

## Products and Subscriptions

Products group APIs and define access policies. Subscriptions grant callers access keys.

```bash
# Create a product
az apim product create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --product-id starter \
  --product-name "Starter Plan" \
  --description "Limited access tier" \
  --subscription-required true \
  --approval-required false \
  --state published

# Link an API to a product
az apim product api add \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --product-id starter \
  --api-id petstore-api

# List subscriptions (includes keys)
az apim subscription list \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --output table

# Get primary key for a subscription
az apim subscription keys list \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --subscription-id <sub-id> \
  --query primaryKey -o tsv

# Regenerate a subscription key
az apim subscription regenerate-key \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --subscription-id <sub-id> \
  --key-kind primary
```

**Calling an API with a subscription key:**
```bash
curl -H "Ocp-Apim-Subscription-Key: <key>" \
     https://myapim-prod.azure-api.net/petstore/pets
```

---

## Policies

Policies are XML-based transformation/enforcement rules applied at four lifecycle points:
- **`<inbound>`** — process the request before sending to backend
- **`<backend>`** — control communication with the backend
- **`<outbound>`** — process the response before returning to caller
- **`<on-error>`** — handle errors at any stage

Policies apply at scopes: **Global** > **Product** > **API** > **Operation**.
Use `<base />` to inherit the parent scope.

### Rate Limiting and Quotas

```xml
<policies>
  <inbound>
    <base />
    <!-- Rate limit: 10 calls per 60 seconds per subscription key -->
    <rate-limit calls="10" renewal-period="60" />

    <!-- Rate limit by key (e.g. per IP) -->
    <rate-limit-by-key calls="5" renewal-period="30"
      counter-key="@(context.Request.IpAddress)"
      increment-condition="@(true)" />

    <!-- Quota: 1000 calls / 1 MB bandwidth per week per subscription -->
    <quota calls="1000" bandwidth="1048576" renewal-period="604800" />

    <!-- Quota by key per user -->
    <quota-by-key calls="500" renewal-period="86400"
      counter-key="@(context.Subscription?.Id ?? context.Request.IpAddress)" />
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
```

### Caching

```xml
<policies>
  <inbound>
    <base />
    <!-- Cache GET responses for 300 seconds, vary by query param -->
    <cache-lookup vary-by-developer="false"
                  vary-by-developer-groups="false"
                  downstream-caching-type="none">
      <vary-by-query-parameter>category</vary-by-query-parameter>
      <vary-by-header>Accept</vary-by-header>
    </cache-lookup>
  </inbound>
  <outbound>
    <base />
    <cache-store duration="300" />
  </outbound>
</policies>
```

### JWT Validation (OAuth / AAD)

```xml
<policies>
  <inbound>
    <base />
    <validate-jwt header-name="Authorization"
                  failed-validation-httpcode="401"
                  failed-validation-error-message="Unauthorized"
                  require-expiration-time="true"
                  require-scheme="Bearer"
                  require-signed-tokens="true">
      <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
      <audiences>
        <audience>api://my-api-app-id</audience>
      </audiences>
      <issuers>
        <issuer>https://sts.windows.net/{tenant-id}/</issuer>
      </issuers>
      <required-claims>
        <claim name="roles" match="any">
          <value>API.Read</value>
          <value>API.Write</value>
        </claim>
      </required-claims>
    </validate-jwt>
  </inbound>
</policies>
```

### CORS

```xml
<policies>
  <inbound>
    <base />
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://app.contoso.com</origin>
        <origin>https://dev.contoso.com</origin>
      </allowed-origins>
      <allowed-methods preflight-result-max-age="300">
        <method>GET</method>
        <method>POST</method>
        <method>PUT</method>
        <method>DELETE</method>
        <method>OPTIONS</method>
      </allowed-methods>
      <allowed-headers>
        <header>Content-Type</header>
        <header>Authorization</header>
        <header>Ocp-Apim-Subscription-Key</header>
      </allowed-headers>
    </cors>
  </inbound>
</policies>
```

### URL Rewriting and Header Manipulation

```xml
<policies>
  <inbound>
    <base />
    <!-- Rewrite path: /api/v2/users → /users -->
    <rewrite-uri template="/users" copy-unmatched-params="true" />

    <!-- Set backend URL dynamically -->
    <set-backend-service base-url="https://new-backend.azurewebsites.net" />

    <!-- Add/override headers -->
    <set-header name="X-Forwarded-For" exists-action="override">
      <value>@(context.Request.IpAddress)</value>
    </set-header>
    <set-header name="X-Api-Version" exists-action="skip">
      <value>2.0</value>
    </set-header>

    <!-- Remove a header -->
    <set-header name="X-Internal-Secret" exists-action="delete" />

    <!-- Set query parameter -->
    <set-query-parameter name="api-version" exists-action="override">
      <value>2024-01-01</value>
    </set-query-parameter>
  </inbound>
  <outbound>
    <base />
    <!-- Strip internal headers from response -->
    <set-header name="X-Powered-By" exists-action="delete" />
    <set-header name="Server" exists-action="delete" />
  </outbound>
</policies>
```

### IP Filtering

```xml
<policies>
  <inbound>
    <base />
    <ip-filter action="allow">
      <address>10.0.0.0/8</address>
      <address>192.168.1.100</address>
      <address-range from="203.0.113.0" to="203.0.113.255" />
    </ip-filter>
  </inbound>
</policies>
```

### Mock Response

```xml
<policies>
  <inbound><base /></inbound>
  <backend>
    <!-- Skip backend entirely and return mock -->
    <mock-response status-code="200" content-type="application/json" />
  </backend>
  <outbound><base /></outbound>
</policies>
```

### JSON/XML Transformation

```xml
<policies>
  <outbound>
    <base />
    <!-- Convert XML backend response to JSON -->
    <xml-to-json kind="direct" apply="always" consider-accept-header="false" />

    <!-- Modify JSON body using JPath -->
    <set-body>@{
      var body = context.Response.Body.As<JObject>();
      body["source"] = "apim-gateway";
      body.Remove("internalId");
      return body.ToString();
    }</set-body>
  </outbound>
</policies>
```

### Send Request (Aggregation / Orchestration)

```xml
<policies>
  <inbound>
    <base />
    <!-- Call an auth service before forwarding -->
    <send-request mode="new" response-variable-name="authResponse" timeout="10" ignore-error="false">
      <set-url>https://auth.contoso.com/validate</set-url>
      <set-method>POST</set-method>
      <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
      </set-header>
      <set-body>@("{\"token\":\"" + context.Request.Headers["Authorization"] + "\"}")</set-body>
    </send-request>
    <choose>
      <when condition="@(((IResponse)context.Variables["authResponse"]).StatusCode != 200)">
        <return-response>
          <set-status code="403" reason="Forbidden" />
        </return-response>
      </when>
    </choose>
  </inbound>
</policies>
```

---

## Policy Expressions (C# Inline)

Policy expressions use C# 7 syntax within `@(...)` (single expression) or `@{...}` (statement block).

Key `context` object properties:

```csharp
context.Api.Id                          // API identifier
context.Api.Name                        // API display name
context.Operation.Id                    // Operation identifier
context.Product.Id                      // Product identifier (if subscribed)
context.Subscription.Id                 // Subscription identifier
context.Subscription.Key                // Subscription key
context.User.Id                         // User identifier
context.Request.IpAddress               // Caller IP
context.Request.Method                  // HTTP method
context.Request.Url.Path                // Request path
context.Request.Url.Query["param"]      // Query string value
context.Request.Headers["name"]         // Request header
context.Request.Body.As<JObject>()      // Parse JSON request body
context.Response.StatusCode             // Backend response code
context.Response.Body.As<JObject>()     // Parse JSON response body
context.Variables["key"]               // Named variables set by set-variable
context.Timestamp                       // Current UTC DateTime
context.Elapsed                         // TimeSpan since request started
context.Deployment.Region               // APIM gateway region
context.Deployment.ServiceName          // APIM service name
```

---

## Named Values (Secrets and Configuration)

Named values store constants and secrets (including Key Vault references) for use in policies.

```bash
# Create a plain named value
az apim nv create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --named-value-id backend-api-key \
  --display-name "Backend API Key" \
  --value "supersecretkey123" \
  --secret true

# Create a Key Vault-backed named value
az apim nv create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --named-value-id kv-backend-key \
  --display-name "KV Backend Key" \
  --key-vault-secret-identifier https://myvault.vault.azure.net/secrets/backend-key
```

Reference in policy:
```xml
<set-header name="X-Backend-Key" exists-action="override">
  <value>{{backend-api-key}}</value>
</set-header>
```

---

## Backends

Backends abstract the downstream service URL and credentials.

```bash
# Create a backend
az apim backend create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --backend-id orders-backend \
  --url https://orders.azurewebsites.net/api \
  --protocol http \
  --title "Orders Service"
```

Policy usage:
```xml
<set-backend-service backend-id="orders-backend" />
```

---

## Versioning and Revisions

**Revisions** — non-breaking changes to an existing API (one revision is current at a time).
**Versions** — breaking changes exposed as separate URL paths or headers.

```bash
# Create a new revision
az apim api revision create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id petstore-api \
  --api-revision 2 \
  --api-revision-description "Add pagination support"

# Make a revision current
az apim api release create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id petstore-api \
  --api-revision 2 \
  --release-id rel-v2 \
  --notes "Released pagination support"

# Create a versioned API (path-based)
az apim api create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id petstore-v2 \
  --path petstore/v2 \
  --display-name "Pet Store API v2" \
  --api-version v2 \
  --api-version-scheme Segment
```

---

## OAuth 2.0 / OpenID Connect Integration

### Register Authorization Server

```bash
# Via ARM REST or portal — configure in APIM Developer Portal
# Authorization endpoint: https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
# Token endpoint:         https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
# Client credentials:     Register app in AAD, grant API permissions
```

### Validate JWT from AAD in Policy

```xml
<validate-jwt header-name="Authorization" failed-validation-httpcode="401">
  <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
  <audiences><audience>api://{app-id}</audience></audiences>
</validate-jwt>
```

### Extract Claims as Variables

```xml
<set-variable name="userId"
  value="@(context.Request.Headers["Authorization"]
    .Split(' ')[1]
    .AsJwt()?.Claims["oid"].FirstOrDefault())" />
```

---

## Self-Hosted Gateway

The self-hosted gateway is a containerized APIM gateway for on-premises or other clouds.

### Deploy to Kubernetes

```bash
# Create a gateway resource in APIM
az apim gateway create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --gateway-id onprem-gateway \
  --location-data name="On-Prem DC"

# Get the deployment config (YAML)
az apim gateway show \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --gateway-id onprem-gateway
```

**Kubernetes deployment (gateway.yaml):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apim-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apim-gateway
  template:
    metadata:
      labels:
        app: apim-gateway
    spec:
      containers:
      - name: apim-gateway
        image: mcr.microsoft.com/azure-api-management/gateway:2.6.0
        env:
        - name: config.service.endpoint
          value: "https://myapim-prod.configuration.azure-api.net"
        - name: config.service.auth
          valueFrom:
            secretKeyRef:
              name: apim-gateway-token
              key: value
        ports:
        - containerPort: 8080   # HTTP
        - containerPort: 8081   # HTTPS
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: apim-gateway
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8081
  selector:
    app: apim-gateway
```

### Docker (local testing)

```bash
docker run -d \
  -p 8080:8080 \
  -e config.service.endpoint="https://myapim-prod.configuration.azure-api.net" \
  -e config.service.auth="GatewayKey <token>" \
  mcr.microsoft.com/azure-api-management/gateway:2.6.0
```

---

## Synthetic GraphQL, WebSocket, gRPC Passthrough

### Synthetic GraphQL (schema import)

APIM can front a REST backend with a GraphQL façade — import a `.graphql` schema,
map each resolver to a REST backend via resolver policies.

```bash
az apim api import \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --api-id graphql-api \
  --path graphql \
  --specification-format GraphQL \
  --specification-url https://example.com/schema.graphql
```

Resolver policy (HTTP data source):
```xml
<http-data-source>
  <http-request>
    <set-method>GET</set-method>
    <set-url>https://backend/products/@(context.GraphQL.Arguments["id"])</set-url>
  </http-request>
</http-data-source>
```

### WebSocket Passthrough

Import a WebSocket API with `--service-url wss://backend` — APIM passes frames through,
policies can inspect the initial HTTP upgrade request.

### gRPC Passthrough

Import a `.proto` file; APIM terminates HTTP/2, translates to backend gRPC.

---

## Developer Portal

The developer portal is an auto-generated, customizable website for API consumers.

```bash
# Publish the developer portal (after customizations)
az apim portal-revision create \
  --resource-group rg-apim-prod \
  --service-name myapim-prod \
  --portal-revision-id rev1 \
  --description "Initial publish"

# Get the developer portal URL
az apim show --name myapim-prod --resource-group rg-apim-prod \
  --query developerPortalUrl -o tsv
```

**Key customization points:**
- Branding via portal editor (logo, colors, fonts — no code required)
- Custom pages with HTML/CSS widgets
- Custom domain: `developer.contoso.com`
- AAD B2C or AAD identity provider for sign-in
- Email notification templates (sign-up, subscription approval, etc.)

---

## Monitoring and Diagnostics

### Application Insights Integration

```bash
# Create App Insights
az monitor app-insights component create \
  --app apim-insights \
  --resource-group rg-apim-prod \
  --location eastus

# Link to APIM (portal: APIM → APIs → select API → Diagnostic Logs → Application Insights)
# Or via ARM: set diagnostics on the API with instrumentationKey
```

Policy to emit custom events:
```xml
<outbound>
  <base />
  <trace source="my-api" severity="information">
    <message>@("Response: " + context.Response.StatusCode)</message>
    <metadata name="userId" value="@(context.User?.Id)" />
  </trace>
</outbound>
```

### Diagnostic Logs (Azure Monitor)

```bash
az monitor diagnostic-settings create \
  --resource $(az apim show --name myapim-prod --resource-group rg-apim-prod --query id -o tsv) \
  --name apim-diag \
  --logs '[{"category":"GatewayLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --workspace $(az monitor log-analytics workspace show \
      --resource-group rg-apim-prod --workspace-name apim-law --query id -o tsv)
```

**Key metrics:** `TotalRequests`, `SuccessfulRequests`, `UnauthorizedRequests`,
`FailedRequests`, `BackendDuration`, `Duration`, `Capacity`.

**KQL query — error rate by API:**
```kql
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| summarize Total=count(), Errors=countif(ResponseCode >= 500)
    by ApiId
| extend ErrorRate = round(100.0 * Errors / Total, 2)
| order by ErrorRate desc
```

---

## Bicep Provisioning

```bicep
param location string = resourceGroup().location
param apimName string = 'myapim-prod'
param publisherEmail string = 'admin@contoso.com'
param publisherName string = 'Contoso'

resource apim 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: apimName
  location: location
  sku: {
    name: 'Standard'
    capacity: 1
  }
  properties: {
    publisherEmail: publisherEmail
    publisherName: publisherName
    customProperties: {
      'Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10': 'False'
      'Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11': 'False'
    }
  }
}

resource api 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apim
  name: 'petstore'
  properties: {
    displayName: 'Pet Store API'
    path: 'petstore'
    protocols: ['https']
    serviceUrl: 'https://petstore3.swagger.io/api/v3'
    format: 'openapi+json-link'
    value: 'https://petstore3.swagger.io/api/v3/openapi.json'
  }
}

resource globalPolicy 'Microsoft.ApiManagement/service/policies@2023-03-01-preview' = {
  parent: apim
  name: 'policy'
  properties: {
    format: 'xml'
    value: '''
<policies>
  <inbound>
    <rate-limit-by-key calls="100" renewal-period="60"
      counter-key="@(context.Request.IpAddress)" />
    <cors><allowed-origins><origin>*</origin></allowed-origins></cors>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>'''
  }
}
```

---

## Terraform Provisioning

```hcl
resource "azurerm_api_management" "apim" {
  name                = "myapim-prod"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  publisher_name      = "Contoso"
  publisher_email     = "admin@contoso.com"
  sku_name            = "Standard_1"

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_api_management_api" "petstore" {
  name                = "petstore"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "1"
  display_name        = "Pet Store API"
  path                = "petstore"
  protocols           = ["https"]

  import {
    content_format = "openapi-link"
    content_value  = "https://petstore3.swagger.io/api/v3/openapi.json"
  }
}

resource "azurerm_api_management_api_policy" "petstore_policy" {
  api_name            = azurerm_api_management_api.petstore.name
  api_management_name = azurerm_api_management.apim.name
  resource_group_name = azurerm_resource_group.rg.name

  xml_content = <<XML
<policies>
  <inbound>
    <base />
    <rate-limit calls="50" renewal-period="60" />
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
XML
}
```

---

## CI/CD with Azure DevOps / GitHub Actions (APIOps)

**APIOps** pattern: API definitions + policies stored in git, deployed via pipeline.

### GitHub Actions — Deploy APIM Policy

```yaml
# .github/workflows/deploy-apim.yml
name: Deploy APIM Configuration

on:
  push:
    branches: [main]
    paths:
      - 'apim/**'

env:
  RESOURCE_GROUP: rg-apim-prod
  APIM_NAME: myapim-prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Import API from OpenAPI spec
      run: |
        az apim api import \
          --resource-group $RESOURCE_GROUP \
          --service-name $APIM_NAME \
          --api-id orders-api \
          --path orders \
          --specification-format OpenApiJson \
          --specification-path apim/apis/orders/openapi.json

    - name: Apply API Policy
      run: |
        az apim api policy create \
          --resource-group $RESOURCE_GROUP \
          --service-name $APIM_NAME \
          --api-id orders-api \
          --xml-file apim/apis/orders/policy.xml

    - name: Apply Global Policy
      run: |
        az apim policy create \
          --resource-group $RESOURCE_GROUP \
          --service-name $APIM_NAME \
          --xml-file apim/policies/global.xml
```

### Azure DevOps Pipeline

```yaml
trigger:
  branches:
    include: [main]
  paths:
    include: [apim/*]

variables:
  resourceGroup: rg-apim-prod
  apimName: myapim-prod

stages:
- stage: Validate
  jobs:
  - job: ValidatePolicies
    steps:
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'APIM-ServiceConnection'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          # Dry-run: check API definition validity
          az apim api show --resource-group $(resourceGroup) \
            --service-name $(apimName) --api-id orders-api

- stage: Deploy
  dependsOn: Validate
  jobs:
  - deployment: DeployAPIM
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            displayName: 'Deploy Policies'
            inputs:
              azureSubscription: 'APIM-ServiceConnection'
              scriptType: bash
              scriptLocation: scriptPath
              scriptPath: scripts/deploy-apim.sh
```

---

## Security Best Practices

1. **Disable HTTP** — enforce HTTPS only on all APIs and the developer portal.
   ```bash
   az apim update --name myapim-prod --resource-group rg-apim-prod \
     --set properties.customProperties."Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10"=False
   ```

2. **Remove subscription key passthrough to backend** — strip the key header:
   ```xml
   <set-header name="Ocp-Apim-Subscription-Key" exists-action="delete" />
   ```

3. **Validate JWT on every API** — never rely solely on subscription keys for auth.

4. **Use Named Values with Key Vault** — never hardcode secrets in policies.

5. **Enable Defender for APIs** — detects OWASP API Top 10 threats via Microsoft Defender.
   ```bash
   az security api-collection onboard \
     --resource-group rg-apim-prod \
     --service-name myapim-prod \
     --api-id orders-api
   ```

6. **Internal VNet (Premium)** — deploy APIM inside a VNet so backends are never publicly
   reachable; place an Application Gateway in front for WAF.

7. **Managed Identity for backends** — use APIM system-assigned identity to call
   Azure services without credentials:
   ```xml
   <authentication-managed-identity resource="https://cognitiveservices.azure.com/" />
   ```

8. **IP allowlist at global scope** — restrict inbound to known CIDR ranges.

9. **Rotate subscription keys** — use key rotation schedule; alert on unused keys.

10. **Audit logs** — enable GatewayLogs to Log Analytics; alert on 401/403 spikes.

---

## Quick Reference — Common az apim Commands

```bash
# List all APIM instances in subscription
az apim list -o table

# List APIs
az apim api list --resource-group rg --service-name myapim -o table

# List operations in an API
az apim api operation list --resource-group rg --service-name myapim --api-id orders-api -o table

# Show current global policy
az apim policy show --resource-group rg --service-name myapim

# Show API-level policy
az apim api policy show --resource-group rg --service-name myapim --api-id orders-api

# Delete an API
az apim api delete --resource-group rg --service-name myapim --api-id old-api --delete-revisions true

# Scale out (add a unit)
az apim update --name myapim --resource-group rg --sku-capacity 2

# Backup APIM to storage
az apim backup \
  --name myapim --resource-group rg \
  --backup-name backup-$(date +%Y%m%d) \
  --storage-account-name mystorageacct \
  --storage-account-container apim-backups \
  --storage-account-key <key>

# Restore from backup
az apim restore \
  --name myapim --resource-group rg \
  --backup-name backup-20240101 \
  --storage-account-name mystorageacct \
  --storage-account-container apim-backups \
  --storage-account-key <key>
```
