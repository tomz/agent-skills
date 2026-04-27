---
name: sf-devops
description: Salesforce DevOps — scratch org model, unlocked packages (1GP/2GP), CI/CD with GitHub Actions, DevOps Center, metadata types, source tracking, and .forceignore
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

# Salesforce DevOps Skill

## Overview

Salesforce has two primary development/deployment models:
1. **Org development model** — change sets, direct org edits, limited source tracking
2. **Package development model** — source-driven, version-controlled, unlocked/managed packages

The package model is strongly preferred for teams. Scratch orgs + unlocked packages + CI/CD
is the modern Salesforce DevOps stack.

---

## Development Models Compared

| | Org Development | Package Development |
|---|---|---|
| Source of truth | Org metadata | Git repository |
| Deployment unit | Change set or full deploy | Package version |
| Environment | Sandbox | Scratch org |
| Source tracking | Manual | Automatic (scratch orgs) |
| Rollback | Manual | Install previous version |
| Dependencies | Implicit | Explicit (in sfdx-project.json) |

---

## Scratch Org Development Flow

```bash
# 1. Create feature branch
git checkout -b feature/account-enhancements

# 2. Create scratch org for the feature
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias feature-scratch \
  --duration-days 7 \
  --set-default

# 3. Push current source to scratch org
sf project deploy start --target-org feature-scratch

# 4. Install package dependencies (if any)
sf package install \
  --package 04t... \
  --target-org feature-scratch \
  --wait 10

# 5. Develop in org (using Setup UI, Developer Console, etc.)

# 6. Pull changes back to local
sf project retrieve start --target-org feature-scratch

# 7. Review, commit, push to GitHub
git add force-app/
git commit -m "feat: add account enhancement fields"
git push origin feature/account-enhancements

# 8. Delete scratch org when done
sf org delete scratch --target-org feature-scratch --no-prompt
```

---

## Unlocked Packages (2nd Generation Packaging — 2GP)

Unlocked packages are the recommended package type for internal apps. They are:
- Upgradeable and versioned
- Subscriber-org-upgradeable (unlike change sets)
- Source-tracked and Git-friendly

### One-time setup
```bash
# Enable Dev Hub in your production org (Setup → Dev Hub)
sf config set target-dev-hub devhub@mycompany.com

# Create the package (one time per project)
sf package create \
  --name "MyCompanyCore" \
  --package-type Unlocked \
  --path force-app \
  --target-dev-hub devhub@mycompany.com

# This adds the package to sfdx-project.json:
# "package": "MyCompanyCore", "packageAlias": "MyCompanyCore: 0Ho..."
```

### Create a package version
```bash
# Create version (NEXT auto-increments patch number)
sf package version create \
  --package "MyCompanyCore" \
  --installation-key-bypass \
  --wait 15 \
  --target-dev-hub devhub@mycompany.com

# Create with installation key (for controlled installs)
sf package version create \
  --package "MyCompanyCore" \
  --installation-key "mySecretKey123" \
  --wait 15 \
  --target-dev-hub devhub@mycompany.com

# List versions
sf package version list \
  --packages "MyCompanyCore" \
  --target-dev-hub devhub@mycompany.com
```

### Promote and install
```bash
# Promote version to "released" (required before installing in production)
sf package version promote \
  --package "MyCompanyCore@1.2.0-1" \
  --target-dev-hub devhub@mycompany.com

# Install in target org
sf package install \
  --package "MyCompanyCore@1.2.0-1" \
  --target-org staging \
  --installation-key "mySecretKey123" \
  --security-type AllUsers \
  --wait 20

# Upgrade (install newer version over existing)
sf package install \
  --package "MyCompanyCore@1.3.0-1" \
  --target-org production \
  --upgrade-type DeprecateOnly \  # or Mixed (removes unused components)
  --wait 30

# Check install status
sf package install report --request-id 0Hf... --target-org staging
```

### sfdx-project.json with package deps
```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true,
      "package": "MyCompanyCore",
      "versionName": "Spring '24",
      "versionNumber": "1.3.0.NEXT",
      "dependencies": [
        { "package": "Salesforce Labs Flow Actions", "versionNumber": "1.6.0.1" },
        { "package": "0Ho..." }
      ]
    }
  ],
  "packageAliases": {
    "MyCompanyCore": "0Ho...",
    "MyCompanyCore@1.2.0-1": "04t...",
    "Salesforce Labs Flow Actions": "0Ho..."
  }
}
```

---

## CI/CD with GitHub Actions

### Directory structure
```
.github/
└── workflows/
    ├── validate-pr.yml      # PR validation (check-only deploy + tests)
    ├── deploy-staging.yml   # Auto-deploy to staging on merge to develop
    └── deploy-prod.yml      # Manual deploy to production
```

### validate-pr.yml — PR validation
```yaml
name: Validate Pull Request

on:
  pull_request:
    branches: [main, develop]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SF CLI
        run: npm install -g @salesforce/cli

      - name: Authenticate to Dev Hub (JWT)
        run: |
          echo "${{ secrets.SERVER_KEY }}" > server.key
          sf org login jwt \
            --username ${{ secrets.SF_USERNAME }} \
            --jwt-key-file server.key \
            --client-id ${{ secrets.SF_CLIENT_ID }} \
            --instance-url ${{ secrets.SF_INSTANCE_URL }} \
            --alias target-org

      - name: Create scratch org
        run: |
          sf org create scratch \
            --definition-file config/project-scratch-def.json \
            --alias ci-scratch \
            --duration-days 1 \
            --set-default \
            --target-dev-hub target-org

      - name: Push source
        run: sf project deploy start --target-org ci-scratch

      - name: Run tests
        run: |
          sf apex test run \
            --target-org ci-scratch \
            --test-level RunLocalTests \
            --code-coverage \
            --result-format human \
            --wait 30

      - name: Delete scratch org
        if: always()
        run: sf org delete scratch --target-org ci-scratch --no-prompt
```

### deploy-staging.yml — Sandbox deploy
```yaml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SF CLI
        run: npm install -g @salesforce/cli

      - name: Auth to Staging Sandbox (JWT)
        run: |
          echo "${{ secrets.STAGING_SERVER_KEY }}" > server.key
          sf org login jwt \
            --username ${{ secrets.STAGING_USERNAME }} \
            --jwt-key-file server.key \
            --client-id ${{ secrets.STAGING_CLIENT_ID }} \
            --instance-url https://test.salesforce.com \
            --alias staging

      - name: Deploy to staging
        run: |
          sf project deploy start \
            --target-org staging \
            --test-level RunLocalTests \
            --wait 30

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"✅ Deployed to staging successfully"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**GitHub Secrets to configure:**
- `SERVER_KEY` — private key for JWT (PEM format, multi-line)
- `SF_USERNAME` — the Salesforce user for CI
- `SF_CLIENT_ID` — Connected App consumer key
- `SF_INSTANCE_URL` — `https://login.salesforce.com` or custom domain

---

## DevOps Center

Salesforce's native DevOps tool (built on GitHub). Use for:
- Work item tracking linked to Git branches
- Pipeline-based promotion (Dev → QA → UAT → Prod)
- No CLI required for simple orgs

Setup: **Setup → DevOps Center → Enable → Connect GitHub org**

Limitations vs custom CI/CD:
- Less flexible test configuration
- No custom pre/post deployment scripts
- Requires GitHub.com (not GitHub Enterprise on-prem)

---

## .forceignore Patterns

```
# .forceignore — exclude from push/pull/deploy
.forceignore
.gitignore
README.md

# Package.xml is for mdapi workflows, not source tracking
manifest/

# Profiles are hard to manage — use Permission Sets instead
**/profiles/**

# These are org-specific and shouldn't be packaged
**/permissionsets/Admin.permissionset-meta.xml

# Stale or duplicate files
**/*.dup
**/.eslintrc.json
**/jsconfig.json

# Reports/dashboards (retrieve manually when needed)
**/reports/**
**/dashboards/**
```

---

## Common Metadata Types Reference

| Metadata Type | File Suffix | Notes |
|---|---|---|
| ApexClass | `.cls` | + `.cls-meta.xml` |
| ApexTrigger | `.trigger` | + `.trigger-meta.xml` |
| LightningComponentBundle | dir + meta.xml | LWC |
| AuraDefinitionBundle | dir | Aura (legacy) |
| CustomObject | `.object-meta.xml` | In `objects/` dir |
| CustomField | `.field-meta.xml` | In `objects/ObjectName/fields/` |
| Flow | `.flow-meta.xml` | Screen/auto-launched flows |
| PermissionSet | `.permissionset-meta.xml` | |
| Profile | `.profile-meta.xml` | Avoid in packages |
| CustomLabel | `.labels-meta.xml` | |
| StaticResource | `.resource` + meta | Binary + text files |
| Layout | `.layout-meta.xml` | |
| ValidationRule | `.validationRule-meta.xml` | In object dir |

---

## Common Gotchas

- **Package namespace lock**: Once you create a managed package with a namespace, it's
  permanent for that Dev Hub. Unlocked packages can be namespace-free (recommended for internal apps).
- **Version number `NEXT`**: The `NEXT` keyword in `versionNumber` auto-increments the patch
  number when creating a version. Use `NEXT` in dev; use explicit numbers for promoted releases.
- **JWT private key in CI**: Store as a multi-line GitHub Secret. Use `echo "$SECRET" > server.key`
  (with double quotes to preserve newlines).
- **Scratch org limits**: Each Dev Hub has a limit on active scratch orgs (default 3–40 depending
  on edition). CI jobs that don't clean up scratch orgs will exhaust the limit.
- **`sf` vs `sfdx` in CI**: Use `sf` (v2+). `sfdx` commands still work but some flags differ.
  Pin the CLI version in CI: `npm install -g @salesforce/cli@latest` or a specific version.
- **Org-dependent packages**: Some metadata (e.g., managed package extensions) requires
  `"orgDependencies": [{"package": "04t..."}]` in package definition instead of standard deps.
- **Deploy order**: Metadata with dependencies must be deployed in order. If a class references
  a custom object, the object must be deployed first. `sf project deploy start` handles this
  automatically for source-format deploys.
