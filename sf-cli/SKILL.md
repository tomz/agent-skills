---
name: sf-cli
description: Salesforce CLI (sf command) — auth, scratch orgs, sandboxes, deployments, metadata API, project structure, and package development
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

# Salesforce CLI (sf) Skill

## Overview

The `sf` CLI (formerly `sfdx`) is the primary tool for Salesforce development. It replaced
`sfdx` as the unified CLI in v2+. Most `sfdx` commands still work as aliases but use `sf`
for new work. Install via npm: `npm install -g @salesforce/cli`

Check version: `sf version` — expect `@salesforce/cli/2.x.x`

---

## Authentication

### Web-based login (interactive)
```bash
# Login to production/developer org
sf org login web --alias myorg --instance-url https://login.salesforce.com

# Login to sandbox
sf org login web --alias mysandbox --instance-url https://test.salesforce.com

# Login to a custom domain
sf org login web --alias myorg --instance-url https://mydomain.my.salesforce.com
```

### JWT Bearer Flow (CI/CD, non-interactive)
```bash
# Requires: Connected App with JWT, server.key (private key), consumer key
sf org login jwt \
  --username ci@myorg.com \
  --jwt-key-file server.key \
  --client-id 3MVG9... \
  --alias ci-org \
  --instance-url https://login.salesforce.com
```

### List and manage orgs
```bash
sf org list                          # All authenticated orgs
sf org display --target-org myorg   # Show org details (access token, instance URL)
sf org logout --target-org myorg    # Revoke auth
sf config set target-org myorg      # Set default org for project
```

---

## Project Structure

A valid Salesforce DX project requires `sfdx-project.json` in the root:

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true,
      "package": "MyPackage",
      "versionName": "Summer '24",
      "versionNumber": "1.2.0.NEXT"
    }
  ],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "61.0"
}
```

### Typical directory layout
```
my-project/
├── sfdx-project.json
├── .forceignore
├── .gitignore
├── config/
│   └── project-scratch-def.json    # Scratch org definition
└── force-app/
    └── main/
        └── default/
            ├── classes/            # Apex classes (.cls + .cls-meta.xml)
            ├── triggers/           # Apex triggers
            ├── lwc/                # Lightning Web Components
            ├── aura/               # Aura components (legacy)
            ├── objects/            # Custom objects + fields
            ├── layouts/            # Page layouts
            ├── permissionsets/     # Permission sets
            ├── profiles/           # Profiles (avoid in packages)
            ├── flows/              # Flow definitions
            └── staticresources/    # Static resources
```

### .forceignore (exclude from push/pull/deploy)
```
.forceignore
.gitignore
**/profiles/**          # Often excluded from packages
**/*.dup
**/jsconfig.json
**/.eslintrc.json
```

---

## Scratch Orgs

Scratch orgs are disposable, ephemeral Salesforce environments for development.

### Scratch org definition (config/project-scratch-def.json)
```json
{
  "orgName": "My Dev Org",
  "edition": "Developer",
  "features": ["EnableSetPasswordInApi", "Communities", "ServiceCloud"],
  "settings": {
    "lightningExperienceSettings": { "enableS1DesktopEnabled": true },
    "securitySettings": { "passwordPolicies": { "enableSetPasswordInApi": true } }
  }
}
```

### Create and use scratch orgs
```bash
# Create (default 7 days, max 30)
sf org create scratch \
  --definition-file config/project-scratch-def.json \
  --alias mydev \
  --duration-days 7 \
  --set-default

# Push local source → scratch org
sf project deploy start --target-org mydev

# Pull changes made in org → local
sf project retrieve start --target-org mydev

# Open in browser
sf org open --target-org mydev

# Delete when done
sf org delete scratch --target-org mydev --no-prompt
```

---

## Deployment (Source vs Metadata API)

### Source deploy (recommended — source-tracked orgs)
```bash
# Deploy entire project
sf project deploy start --target-org myorg

# Deploy specific directory or file
sf project deploy start --source-dir force-app/main/default/classes --target-org myorg

# Deploy with test run
sf project deploy start \
  --target-org myorg \
  --test-level RunLocalTests

# Check-only (validate without saving)
sf project deploy start \
  --target-org myorg \
  --dry-run \
  --test-level RunLocalTests

# Deploy specific metadata
sf project deploy start \
  --metadata ApexClass:MyClass,CustomObject:Account__c \
  --target-org myorg
```

### Retrieve metadata from org
```bash
# Retrieve by metadata type
sf project retrieve start \
  --metadata ApexClass,CustomObject \
  --target-org myorg

# Retrieve specific component
sf project retrieve start \
  --metadata "ApexClass:AccountService" \
  --target-org myorg

# Retrieve everything (careful with large orgs)
sf project retrieve start --target-org myorg
```

### Deploy status and async
```bash
# Deploy async (get job ID)
sf project deploy start --target-org myorg --async

# Check status of async deploy
sf project deploy report --job-id 0Af...

# Cancel async deploy
sf project deploy cancel --job-id 0Af...
```

---

## Metadata API (mdapi style)

```bash
# Convert source format → mdapi format
sf project convert source --output-dir mdapi-output

# Deploy mdapi package
sf project deploy start --metadata-dir mdapi-output --target-org myorg

# Retrieve as mdapi package
sf project retrieve start \
  --target-org myorg \
  --manifest package.xml \
  --output-dir mdapi-output
```

### package.xml example
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
  <types>
    <members>*</members>
    <name>ApexClass</name>
  </types>
  <types>
    <members>Account</members>
    <name>CustomObject</name>
  </types>
  <version>61.0</version>
</Package>
```

---

## Running Apex & SOQL from CLI

```bash
# Execute anonymous Apex
sf apex run --file scripts/setup.apex --target-org myorg

# Interactive Apex REPL
sf apex run --target-org myorg

# Run SOQL query
sf data query --query "SELECT Id, Name FROM Account LIMIT 10" --target-org myorg

# Query with CSV output
sf data query \
  --query "SELECT Id, Name, Email FROM Contact" \
  --result-format csv \
  --target-org myorg > contacts.csv
```

---

## Apex Tests
```bash
# Run all local tests
sf apex test run --target-org myorg --test-level RunLocalTests --synchronous

# Run specific class
sf apex test run --class-names MyClassTest --target-org myorg --synchronous

# Run specific method
sf apex test run \
  --tests "MyClassTest.testPositiveCase,MyClassTest.testBulk" \
  --target-org myorg \
  --synchronous

# Get code coverage report
sf apex test run \
  --target-org myorg \
  --test-level RunLocalTests \
  --code-coverage \
  --result-format human
```

---

## Debug Logs
```bash
# Tail real-time debug logs
sf apex tail log --target-org myorg

# Get recent logs
sf apex log list --target-org myorg

# Get specific log
sf apex log get --log-id 07L... --target-org myorg

# Set trace flag for user
sf apex log tail --target-org myorg --debug-level SFDC_DevConsole
```

---

## Common Gotchas

- **API version drift**: `sourceApiVersion` in `sfdx-project.json` must be ≤ org API version.
  Mismatch causes deploy errors. Check org version: `sf org display`.
- **Profiles vs Permission Sets**: Profiles are notoriously hard to manage in source control.
  Prefer Permission Sets. Add `**/profiles/**` to `.forceignore` for packages.
- **Scratch org limits**: Dev Hub limits vary by edition (e.g., 6 active scratch orgs for
  Developer Edition). Check: `sf org list --all`.
- **Push vs Deploy**: `sf project deploy start` works on all org types. In scratch orgs,
  use source tracking (push/pull) for change detection. On sandboxes/prod, always deploy.
- **Metadata not in source format**: Some metadata types (e.g., Reports, Dashboards) must
  be retrieved before they appear locally. They are not auto-tracked.
- **JWT auth in CI**: The private key (`server.key`) must never be committed. Use
  environment variables or secrets managers. The Connected App must have the user's profile
  pre-authorized.
