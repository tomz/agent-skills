---
name: azure-static-web-apps
description: Azure Static Web Apps — JAMstack hosting, API backends, authentication, custom domains, staging environments, CLI, framework presets, database connections
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Static Web Apps (SWA) — Comprehensive Reference

Azure Static Web Apps (SWA) is a fully managed hosting service for static frontends
with integrated serverless API backends, built-in auth, global CDN distribution,
staging environments from pull requests, and zero-config SSL.

---

## Tier Comparison: Free vs Standard

| Feature                        | Free                        | Standard                         |
|--------------------------------|-----------------------------|----------------------------------|
| Price                          | $0/month                    | ~$9/month per app                |
| Custom domains                 | 2                           | 5                                |
| SSL certificates               | Free (managed)              | Free (managed) + bring your own  |
| Max app size                   | 250 MB                      | 500 MB                           |
| Auth providers                 | Built-in only               | Built-in + custom OIDC           |
| Staging environments           | 3                           | 10                               |
| Private endpoints              | No                          | Yes                              |
| SLA                            | None                        | 99.95%                           |
| Password protection            | No                          | Yes                              |
| Split traffic                  | No                          | Yes                              |
| Managed functions              | Managed (limited regions)   | Managed or bring-your-own        |
| Database connections (DAB)     | No                          | Yes                              |
| Enterprise edge                | No                          | Yes (add-on)                     |

---

## Supported Frameworks & Presets

SWA auto-detects build settings for these frameworks:

| Framework       | `app_build_command`          | `output_location`   |
|-----------------|------------------------------|---------------------|
| React (CRA)     | `npm run build`              | `build`             |
| React (Vite)    | `npm run build`              | `dist`              |
| Angular         | `ng build`                   | `dist/<project>`    |
| Vue (Vite)      | `npm run build`              | `dist`              |
| Next.js         | `next build`                 | `.next`             |
| Nuxt 3          | `nuxt generate`              | `.output/public`    |
| Gatsby          | `gatsby build`               | `public`            |
| Hugo            | `hugo`                       | `public`            |
| Jekyll          | `jekyll build`               | `_site`             |
| Svelte          | `npm run build`              | `public`            |
| Blazor WASM     | `dotnet publish`             | `wwwroot`           |
| Astro           | `npm run build`              | `dist`              |
| Eleventy        | `npx @11ty/eleventy`         | `_site`             |
| VitePress       | `vitepress build`            | `.vitepress/dist`   |

For Next.js and Nuxt with SSR, SWA uses hybrid rendering via managed functions.
Set `"outputType": "server"` in build config for SSR mode.

---

## staticwebapp.config.json

This file lives in the root of your deployed app (or `app_location`) and controls
routing, headers, auth, and networking.

### Full example

```json
{
  "routes": [
    {
      "route": "/api/*",
      "allowedRoles": ["authenticated"]
    },
    {
      "route": "/admin/*",
      "allowedRoles": ["administrator"]
    },
    {
      "route": "/login",
      "redirect": "/.auth/login/github",
      "statusCode": 302
    },
    {
      "route": "/logout",
      "redirect": "/.auth/logout",
      "statusCode": 302
    },
    {
      "route": "/old-page",
      "redirect": "/new-page",
      "statusCode": 301
    },
    {
      "route": "/products/:id",
      "rewrite": "/products/detail"
    },
    {
      "route": "/*.js",
      "headers": {
        "Cache-Control": "public, max-age=31536000, immutable"
      }
    },
    {
      "route": "/*.css",
      "headers": {
        "Cache-Control": "public, max-age=31536000, immutable"
      }
    }
  ],
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/images/*.{png,jpg,gif,ico,webp}", "/css/*", "/js/*", "/api/*"]
  },
  "responseOverrides": {
    "400": { "rewrite": "/400.html", "statusCode": 400 },
    "401": { "redirect": "/login", "statusCode": 302 },
    "403": { "rewrite": "/403.html", "statusCode": 403 },
    "404": { "rewrite": "/404.html", "statusCode": 404 }
  },
  "globalHeaders": {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "X-XSS-Protection": "1; mode=block",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "geolocation=(), microphone=()",
    "Content-Security-Policy": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
  },
  "mimeTypes": {
    ".json": "text/json",
    ".woff2": "font/woff2",
    ".wasm": "application/wasm"
  },
  "auth": {
    "identityProviders": {
      "github": {
        "registration": {
          "clientIdSettingName": "GITHUB_CLIENT_ID",
          "clientSecretSettingName": "GITHUB_CLIENT_SECRET"
        }
      },
      "azureActiveDirectory": {
        "registration": {
          "openIdIssuer": "https://login.microsoftonline.com/<tenant-id>/v2.0",
          "clientIdSettingName": "AAD_CLIENT_ID",
          "clientSecretSettingName": "AAD_CLIENT_SECRET"
        },
        "userDetailsClaim": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"
      },
      "customOpenIdConnectProviders": {
        "auth0": {
          "registration": {
            "clientIdSettingName": "AUTH0_CLIENT_ID",
            "clientSecretSettingName": "AUTH0_CLIENT_SECRET",
            "openIdConnectConfiguration": {
              "wellKnownOpenIdConfiguration": "https://<your-domain>.auth0.com/.well-known/openid-configuration"
            }
          },
          "login": {
            "nameClaimType": "name",
            "scopes": ["openid", "profile", "email"]
          }
        }
      }
    }
  },
  "networking": {
    "allowedIpRanges": ["10.0.0.0/8", "192.168.0.0/16"]
  },
  "forwardingGateway": {
    "requiredHeaders": {
      "X-Azure-FDID": "<frontdoor-id>"
    }
  },
  "trailingSlash": "auto",
  "platform": {
    "apiRuntime": "node:18"
  }
}
```

### Key config fields

- **`routes`** — ordered list; first match wins. Supports wildcards (`*`), named
  segments (`:param`), and glob extensions (`*.js`).
- **`navigationFallback`** — SPA client-side routing fallback. `exclude` prevents
  API calls and static assets from falling through.
- **`responseOverrides`** — intercept HTTP status codes for custom error pages or
  auth redirects.
- **`globalHeaders`** — applied to every response. Ideal for security headers.
- **`platform.apiRuntime`** — sets Node.js/Python/.NET version for managed functions.
  Values: `node:16`, `node:18`, `node:20`, `python:3.8`, `python:3.9`, `python:3.10`,
  `dotnet:6`, `dotnet:8`.

---

## API Backends

### Managed Azure Functions (built-in)

Place functions in an `api/` folder at the repo root. SWA deploys them automatically
as Azure Functions with the same lifecycle as the frontend.

```
my-app/
├── src/               # frontend source
├── api/               # managed functions
│   ├── host.json
│   ├── package.json
│   └── GetProducts/
│       ├── function.json
│       └── index.js
└── staticwebapp.config.json
```

`api/GetProducts/function.json`:
```json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"]
    },
    { "type": "http", "direction": "out", "name": "res" }
  ]
}
```

Managed functions are available at `/api/*` with no CORS config needed — SWA
handles CORS between the frontend and its own functions automatically.

### Bring-Your-Own Backend (Linked Backends)

Link an existing Azure resource as the API backend (Standard tier only):

```bash
# Link an existing Function App
az staticwebapp backends link \
  --name my-swa \
  --resource-group my-rg \
  --backend-resource-id "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Web/sites/<func-app>" \
  --backend-region eastus

# Link an API Management instance
az staticwebapp backends link \
  --name my-swa \
  --resource-group my-rg \
  --backend-resource-id "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ApiManagement/service/<apim>"

# Unlink
az staticwebapp backends unlink --name my-swa --resource-group my-rg
```

Linked backends receive a managed identity token from SWA — no CORS config or
API keys needed for the `/api/*` proxy.

---

## Database Connections (Data API Builder)

Connects a database directly to `/data-api/*` REST and GraphQL endpoints.
**Standard tier only.**

```bash
# Enable database connection
az staticwebapp dbconnection create \
  --name my-swa \
  --resource-group my-rg \
  --db-resource-id "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Sql/servers/<srv>/databases/<db>" \
  --db-type mssql
```

Supported databases: `mssql` (Azure SQL), `cosmosdb_nosql`, `cosmosdb_gremlin`,
`mysql`, `postgresql`.

`staticwebapp.database.config.json` (in repo root):
```json
{
  "$schema": "https://github.com/Azure/data-api-builder/releases/latest/download/dab.draft.schema.json",
  "data-source": {
    "database-type": "mssql",
    "connection-string": "@env('DATABASE_CONNECTION_STRING')"
  },
  "runtime": {
    "rest": { "enabled": true, "path": "/rest" },
    "graphql": { "enabled": true, "path": "/graphql", "allow-introspection": true }
  },
  "entities": {
    "Product": {
      "source": "dbo.Products",
      "permissions": [
        { "role": "anonymous", "actions": ["read"] },
        { "role": "authenticated", "actions": ["read", "create", "update"] },
        { "role": "administrator", "actions": ["*"] }
      ]
    }
  }
}
```

---

## Authentication & Authorization

### Built-in Providers (Free + Standard)

| Provider   | Login URL                          |
|------------|-------------------------------------|
| GitHub     | `/.auth/login/github`               |
| AAD/Entra  | `/.auth/login/aad`                  |
| Twitter    | `/.auth/login/twitter`              |
| Google     | `/.auth/login/google`               |

All providers: logout at `/.auth/logout`, user info at `/.auth/me`.

`/.auth/me` response:
```json
{
  "clientPrincipal": {
    "identityProvider": "github",
    "userId": "abc123",
    "userDetails": "octocat",
    "userRoles": ["anonymous", "authenticated", "administrator"],
    "claims": []
  }
}
```

### Role Assignment via Invitations

```bash
# Create an invitation link (assigns 'administrator' role)
az staticwebapp users invite \
  --name my-swa \
  --resource-group my-rg \
  --authentication-provider GitHub \
  --user-details octocat \
  --role administrator \
  --invitation-expiration-in-hours 24
```

### Role-Based Access in Config

```json
{
  "routes": [
    { "route": "/admin/*", "allowedRoles": ["administrator"] },
    { "route": "/api/write/*", "allowedRoles": ["editor", "administrator"] },
    { "route": "/public/*", "allowedRoles": ["anonymous"] }
  ],
  "responseOverrides": {
    "401": { "redirect": "/.auth/login/github?post_login_redirect_uri=/admin", "statusCode": 302 }
  }
}
```

### Custom OIDC (Standard tier)

Supports any OIDC-compliant provider (Auth0, Okta, Keycloak, Azure AD B2C).
Register client ID/secret as app settings and reference via `*SettingName` keys
in `staticwebapp.config.json` (shown in full example above).

---

## Staging Environments

SWA auto-creates preview environments for every pull request.

- Preview URL pattern: `https://<swa-name>-<hash>-<pr-number>.<region>.azurestaticapps.net`
- Environments are destroyed when the PR is closed/merged.
- Named environments (Standard): up to 10 concurrent named staging slots.

```bash
# List environments
az staticwebapp environment list --name my-swa --resource-group my-rg

# Delete a specific environment
az staticwebapp environment delete \
  --name my-swa \
  --resource-group my-rg \
  --environment-name pr-42
```

---

## Custom Domains & SSL

```bash
# Add custom domain (auto-provisions free TLS cert via Let's Encrypt)
az staticwebapp hostname set \
  --name my-swa \
  --resource-group my-rg \
  --hostname www.example.com

# Add apex domain (requires ALIAS/ANAME or Azure DNS ALIAS record)
az staticwebapp hostname set \
  --name my-swa \
  --resource-group my-rg \
  --hostname example.com

# List hostnames
az staticwebapp hostname list --name my-swa --resource-group my-rg

# Remove hostname
az staticwebapp hostname delete \
  --name my-swa \
  --resource-group my-rg \
  --hostname www.example.com
```

DNS configuration:
- **Subdomain (`www`):** CNAME → `<swa-name>.<hash>.<region>.azurestaticapps.net`
- **Apex (`example.com`):** ALIAS/ANAME or Azure DNS ALIAS record → same target

---

## SWA CLI — Local Development

```bash
# Install
npm install -g @azure/static-web-apps-cli

# Initialize project (creates swa-cli.config.json)
swa init

# Start local dev server (serves built files + proxies /api/*)
swa start

# Start with live dev server (Vite, CRA, etc.)
swa start http://localhost:3000 --api-location ./api

# Start with specific config entry
swa start my-app

# Login to Azure (stores credentials locally)
swa login --subscription-id <sub> --resource-group my-rg --app-name my-swa

# Deploy to Azure
swa deploy ./build --deployment-token $SWA_TOKEN

# Deploy using config
swa deploy --env production

# Build + deploy
swa deploy --app-location ./src --output-location ./dist --api-location ./api
```

`swa-cli.config.json` (generated by `swa init`):
```json
{
  "$schema": "https://aka.ms/azure/static-web-apps-cli/schema",
  "configurations": {
    "my-app": {
      "appLocation": ".",
      "apiLocation": "api",
      "outputLocation": "dist",
      "appBuildCommand": "npm run build",
      "apiBuildCommand": "npm run build --prefix api",
      "run": "npm run dev",
      "appDevserverUrl": "http://localhost:5173",
      "env": "production",
      "resourceGroup": "my-rg",
      "appName": "my-swa"
    }
  }
}
```

Local auth emulation: SWA CLI emulates `/.auth/*` endpoints locally. Customize
the mock user via the browser popup at `http://localhost:4280/.auth/login/github`.

---

## az staticwebapp CLI Commands

```bash
# Create SWA resource
az staticwebapp create \
  --name my-swa \
  --resource-group my-rg \
  --location eastus2 \
  --sku Standard \
  --source https://github.com/myorg/myrepo \
  --branch main \
  --app-location "/" \
  --output-location "dist" \
  --api-location "api" \
  --login-with-github

# Show SWA details
az staticwebapp show --name my-swa --resource-group my-rg

# List all SWAs in subscription
az staticwebapp list --output table

# Get deployment token (for CI/CD)
az staticwebapp secrets list --name my-swa --resource-group my-rg

# Reset deployment token
az staticwebapp secrets reset-api-key --name my-swa --resource-group my-rg

# Set application settings (env vars)
az staticwebapp appsettings set \
  --name my-swa \
  --resource-group my-rg \
  --setting-names "API_KEY=abc123" "FEATURE_FLAG=true"

# List application settings
az staticwebapp appsettings list --name my-swa --resource-group my-rg

# Delete application settings
az staticwebapp appsettings delete \
  --name my-swa \
  --resource-group my-rg \
  --setting-names "OLD_KEY"

# Delete SWA
az staticwebapp delete --name my-swa --resource-group my-rg --yes
```

---

## GitHub Actions Workflow

SWA auto-generates this workflow when created via the portal or `az staticwebapp create`.

`.github/workflows/azure-static-web-apps.yml`:
```yaml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches: [main]

jobs:
  build_and_deploy:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          lfs: false

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install and build
        run: |
          npm ci
          npm run build

      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: upload
          app_location: "/"
          api_location: "api"
          output_location: "dist"
          skip_app_build: true   # we built above

  close_pull_request:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request
    steps:
      - uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          action: close
```

Key `Azure/static-web-apps-deploy@v1` inputs:
- `app_location` — root of your frontend source
- `output_location` — build output folder (relative to `app_location`)
- `api_location` — folder with Azure Functions (leave empty if none)
- `skip_app_build: true` — skip the action's internal build (use if you build yourself)
- `deployment_environment` — target named environment (omit for production)

---

## Azure DevOps Pipeline

```yaml
trigger:
  branches:
    include: [main]

pr:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

variables:
  SWA_TOKEN: $(AZURE_STATIC_WEB_APPS_API_TOKEN)

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  - script: npm ci && npm run build
    displayName: Install and Build

  - task: AzureStaticWebApp@0
    inputs:
      app_location: '/'
      api_location: 'api'
      output_location: 'dist'
      skip_app_build: true
      azure_static_web_apps_api_token: $(SWA_TOKEN)
    displayName: Deploy to SWA
```

---

## Enterprise Features (Standard Tier)

### Private Endpoints

```bash
az network private-endpoint create \
  --name my-swa-pe \
  --resource-group my-rg \
  --vnet-name my-vnet \
  --subnet pe-subnet \
  --private-connection-resource-id $(az staticwebapp show --name my-swa --resource-group my-rg --query id -o tsv) \
  --group-id staticSites \
  --connection-name my-swa-connection
```

### Password Protection

```bash
az staticwebapp environment set-http-auth \
  --name my-swa \
  --resource-group my-rg \
  --environment-name default \
  --password mysecretpassword \
  --secret-url "" 
```

Or via portal: SWA → Environments → Select environment → Password protection.

### Split Traffic (A/B Testing)

```bash
# Route 20% of traffic to a staging environment
az staticwebapp environment traffic-weight set \
  --name my-swa \
  --resource-group my-rg \
  --environment-name staging \
  --weight 20
```

---

## Environment Variables & Application Settings

App settings are available to managed Azure Functions as environment variables.
They are NOT exposed to the frontend (never put secrets in `staticwebapp.config.json`).

```bash
# Set via CLI
az staticwebapp appsettings set \
  --name my-swa \
  --resource-group my-rg \
  --setting-names \
    "DATABASE_URL=Server=tcp:..." \
    "SENDGRID_KEY=SG.xxx" \
    "FEATURE_DARK_MODE=true"
```

Frontend environment variables must be baked in at **build time** via your
framework's `.env` / `VITE_*` / `NEXT_PUBLIC_*` conventions, or passed as
GitHub Actions secrets:

```yaml
- name: Build
  env:
    VITE_API_BASE: ${{ vars.API_BASE_URL }}
    VITE_GA_ID: ${{ secrets.GA_MEASUREMENT_ID }}
  run: npm run build
```

---

## Monitoring & Application Insights

```bash
# Enable Application Insights on a SWA (links existing AI resource)
az monitor app-insights component connect-webapp \
  --resource-group my-rg \
  --app my-insights \
  --web-app my-swa \
  --enable-profiler false \
  --enable-snapshot-debugger false
```

In your frontend:
```js
// vite.config.js / at app entry
import { ApplicationInsights } from '@microsoft/applicationinsights-web';
const appInsights = new ApplicationInsights({
  config: { connectionString: import.meta.env.VITE_APPINSIGHTS_CONNECTION_STRING }
});
appInsights.loadAppInsights();
appInsights.trackPageView();
```

SWA built-in metrics (Azure Portal → SWA → Metrics):
- `Requests` — total HTTP requests
- `BytesSent` — egress bandwidth
- `Errors` — 4xx/5xx count
- `RequestDuration` — response latency (P50/P95/P99)

---

## Bicep Example

```bicep
param appName string = 'my-swa'
param location string = resourceGroup().location
param repoUrl string
param branch string = 'main'
param githubToken string

resource swa 'Microsoft.Web/staticSites@2023-01-01' = {
  name: appName
  location: location
  sku: {
    name: 'Standard'
    tier: 'Standard'
  }
  properties: {
    repositoryUrl: repoUrl
    branch: branch
    repositoryToken: githubToken
    buildProperties: {
      appLocation: '/'
      apiLocation: 'api'
      outputLocation: 'dist'
    }
  }
}

resource swaSettings 'Microsoft.Web/staticSites/config@2023-01-01' = {
  parent: swa
  name: 'appsettings'
  properties: {
    DATABASE_URL: 'Server=tcp:...'
    FEATURE_FLAG: 'true'
  }
}

resource customDomain 'Microsoft.Web/staticSites/customDomains@2023-01-01' = {
  parent: swa
  name: 'www.example.com'
  properties: {}
}

output defaultHostname string = swa.properties.defaultHostname
output deploymentToken string = listSecrets(swa.id, swa.apiVersion).properties.apiKey
```

## Terraform Example

```hcl
resource "azurerm_static_web_app" "main" {
  name                = "my-swa"
  resource_group_name = azurerm_resource_group.main.name
  location            = "eastus2"
  sku_tier            = "Standard"
  sku_size            = "Standard"
}

resource "azurerm_static_web_app_custom_domain" "www" {
  static_web_app_id = azurerm_static_web_app.main.id
  domain_name       = "www.example.com"
  validation_type   = "cname-delegation"
}

output "api_key" {
  value     = azurerm_static_web_app.main.api_key
  sensitive = true
}
```

---

## Best Practices & Performance

### Security
- Always set `X-Content-Type-Options`, `X-Frame-Options`, and CSP in `globalHeaders`.
- Use `allowedRoles` on all API routes — never rely on obscurity.
- Store secrets only in app settings, never in `staticwebapp.config.json` or frontend code.
- For custom OIDC, always reference secrets via `*SettingName` indirection.
- Enable private endpoints for internal-only apps (Standard tier).

### Performance
- Set immutable cache headers (`Cache-Control: max-age=31536000, immutable`) for
  hashed assets (JS/CSS chunks from Vite/webpack).
- Set short `Cache-Control` for `index.html` (`no-cache` or `max-age=0`) so
  deploys propagate instantly.
- Use `navigationFallback.exclude` to prevent API calls from hitting the SPA fallback.
- Enable brotli compression automatically (SWA CDN handles this).
- Co-locate SWA region with your backend database/functions to minimize latency.

### CI/CD
- Use `skip_app_build: true` and build in a separate step to leverage caching
  (`actions/cache` for `node_modules`).
- Use environments and secrets per environment in GitHub Actions for staging vs production.
- Rotate deployment tokens periodically via `az staticwebapp secrets reset-api-key`.

### Cost
- Free tier is sufficient for personal projects and internal tools with < 3 staging envs.
- Standard tier is required for production apps needing custom OIDC, private endpoints,
  database connections, or SLAs.
- Egress costs apply beyond the free tier's included bandwidth — use CDN-friendly
  immutable cache headers to minimize re-fetches.

---

## Quick-Reference Cheat Sheet

```bash
# Create
az staticwebapp create -n my-swa -g my-rg -l eastus2 --sku Standard

# Get deployment token
az staticwebapp secrets list -n my-swa -g my-rg --query 'properties.apiKey' -o tsv

# Set env vars
az staticwebapp appsettings set -n my-swa -g my-rg --setting-names KEY=val

# Add custom domain
az staticwebapp hostname set -n my-swa -g my-rg --hostname www.example.com

# List environments
az staticwebapp environment list -n my-swa -g my-rg

# SWA CLI local dev
swa start http://localhost:5173 --api-location ./api

# SWA CLI deploy
swa deploy --deployment-token $TOKEN --output-location dist
```
