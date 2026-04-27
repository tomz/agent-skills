---
name: do-app-platform
description: DigitalOcean App Platform — app specs, components, deploys, scaling, env vars, custom domains, and deploy hooks
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

# DigitalOcean App Platform Skill

## Overview

App Platform is DO's PaaS. You push code (or a Docker image) and DO handles the build,
deploy, TLS, scaling, and routing. No server management. Supports GitHub, GitLab, and
Docker Hub as sources.

**Component types:**
| Type | Persistent | Runs | Billed |
|------|-----------|------|--------|
| **Service** | No | Continuously (HTTP) | Always |
| **Worker** | No | Continuously (no HTTP) | Always |
| **Job** | No | On demand / cron | Per run |
| **Static Site** | No | CDN-served | Free tier available |
| **Function** | No | On invocation | Per invocation |

---

## Quick Start

```bash
# List apps
doctl apps list

# Create app from a spec file (preferred — declarative, version-controlled)
doctl apps create --spec app.yaml

# Create interactively from a GitHub repo
doctl apps create \
  --spec - << 'EOF'
name: my-app
region: nyc
services:
  - name: web
    github:
      repo: myorg/my-app
      branch: main
      deploy_on_push: true
    run_command: gunicorn app:app
    http_port: 8080
    instance_size_slug: apps-s-1vcpu-0.5gb
    instance_count: 1
EOF
```

---

## App Spec (YAML)

The app spec is the single source of truth. Keep it in your repo.

```yaml
# app.yaml — comprehensive example
name: my-saas
region: nyc   # nyc, ams, sfo, sgp, lon, fra, tor, syd

# Environment variables shared across all components
envs:
  - key: ENVIRONMENT
    value: production
    scope: RUN_AND_BUILD_TIME
  - key: DATABASE_URL
    value: ${my-db.DATABASE_URL}   # reference a managed DB component
    scope: RUN_TIME
  - key: SECRET_KEY
    type: SECRET                   # encrypted at rest, masked in logs
    value: "my-secret-value"
    scope: RUN_TIME

services:
  - name: api
    github:
      repo: myorg/my-saas
      branch: main
      deploy_on_push: true
    build_command: pip install -r requirements.txt
    run_command: gunicorn --workers 4 --bind 0.0.0.0:8080 app:app
    http_port: 8080
    instance_size_slug: apps-s-1vcpu-0.5gb
    instance_count: 2
    health_check:
      http_path: /health
      initial_delay_seconds: 10
      period_seconds: 10
      timeout_seconds: 5
      success_threshold: 1
      failure_threshold: 3
    routes:
      - path: /api
    envs:
      - key: WORKER_THREADS
        value: "4"
        scope: RUN_TIME
    cors:
      allow_origins:
        - exact: "https://example.com"
      allow_methods:
        - GET
        - POST
      allow_headers:
        - Authorization
      allow_credentials: true
    alert_policies:
      - rule: CPU_UTILIZATION
        operator: GREATER_THAN
        value: 80
        window: FIVE_MINUTES
      - rule: RESTART_COUNT
        operator: GREATER_THAN
        value: 5
        window: FIVE_MINUTES

  - name: frontend
    github:
      repo: myorg/my-saas-frontend
      branch: main
      deploy_on_push: true
    build_command: npm ci && npm run build
    run_command: node server.js
    http_port: 3000
    instance_size_slug: apps-s-1vcpu-0.5gb
    instance_count: 1
    routes:
      - path: /

workers:
  - name: queue-worker
    github:
      repo: myorg/my-saas
      branch: main
      deploy_on_push: true
    run_command: python worker.py
    instance_size_slug: apps-s-1vcpu-0.5gb
    instance_count: 1

jobs:
  - name: db-migrate
    github:
      repo: myorg/my-saas
      branch: main
    run_command: python manage.py migrate
    instance_size_slug: apps-s-1vcpu-0.5gb
    kind: PRE_DEPLOY   # runs before each deploy (POST_DEPLOY also available)

static_sites:
  - name: docs
    github:
      repo: myorg/my-saas-docs
      branch: main
      deploy_on_push: true
    build_command: mkdocs build
    output_dir: site/
    routes:
      - path: /docs

databases:
  - name: my-db
    engine: PG
    version: "16"
    size: db-s-1vcpu-1gb
    num_nodes: 1

domains:
  - domain: example.com
    type: PRIMARY
    zone: example.com
  - domain: www.example.com
    type: ALIAS
```

---

## Deploying from Docker

```yaml
services:
  - name: web
    image:
      registry_type: DOCR           # DigitalOcean Container Registry
      registry: my-registry
      repository: my-app
      tag: latest
      # For private DockerHub:
      # registry_type: DOCKER_HUB
      # registry: myuser
      # repository: my-app
      # tag: v1.2.3
    http_port: 8080
    instance_size_slug: apps-s-1vcpu-0.5gb
    instance_count: 1
```

---

## Managing Apps with doctl

```bash
# Create from spec file
doctl apps create --spec app.yaml

# Update spec (redeploys automatically)
doctl apps update $APP_ID --spec app.yaml

# Get current spec (export for editing)
doctl apps spec get $APP_ID > current-spec.yaml

# Validate a spec without deploying
doctl apps spec validate app.yaml

# Get app details and public URL
doctl apps get $APP_ID

# List deployments
doctl apps list-deployments $APP_ID

# Get deployment logs (build + runtime)
doctl apps logs $APP_ID --type BUILD
doctl apps logs $APP_ID --type DEPLOY
doctl apps logs $APP_ID --type RUN
doctl apps logs $APP_ID --type RUN --follow  # tail -f style

# Trigger a manual deploy
doctl apps create-deployment $APP_ID

# Force rebuild (clears build cache)
doctl apps create-deployment $APP_ID --force-rebuild

# Get deployment status
doctl apps get-deployment $APP_ID $DEPLOYMENT_ID

# Cancel a deployment in progress
doctl apps cancel-deployment $APP_ID $DEPLOYMENT_ID

# Delete app
doctl apps delete $APP_ID
```

---

## Environment Variables

```bash
# List env vars for an app
doctl apps spec get $APP_ID | yq '.envs'

# The right way: edit app.yaml and re-apply
# App Platform doesn't have a one-liner for adding envs outside the spec

# For secrets, use type: SECRET in the spec.
# The value is write-only after creation — you can't read it back via API.
```

Environment variable scopes:

| Scope | Build | Run |
|-------|-------|-----|
| `BUILD_TIME` | ✓ | ✗ |
| `RUN_TIME` | ✗ | ✓ |
| `RUN_AND_BUILD_TIME` | ✓ | ✓ |

---

## Custom Domains

```bash
# Add a domain via spec (preferred):
# domains:
#   - domain: example.com
#     type: PRIMARY

# Or add after creation:
doctl apps update $APP_ID --spec - << EOF
$(doctl apps spec get $APP_ID)
domains:
  - domain: example.com
    type: PRIMARY
EOF

# Point your DNS to DO:
# CNAME www.example.com → <app-name>.ondigitalocean.app
# For apex (@) domain: use DO DNS with ALIAS record or CNAME flattening
```

---

## Deploy Hooks

Deploy hooks trigger a new deployment via a secret URL — useful for external CI/CD or
webhooks from your registry.

```bash
# List deploy hooks
doctl apps list-deployments $APP_ID  # hooks are shown in app details

# Create a deploy hook (via control panel UI):
# App → Settings → Deploy Hooks → Add Hook
# Returns a URL like: https://api.digitalocean.com/v2/apps/$APP_ID/deployments?token=...

# Trigger via curl
curl -X POST "https://api.digitalocean.com/v2/apps/$APP_ID/deployments" \
  -H "Authorization: Bearer $DO_TOKEN" \
  -H "Content-Type: application/json"
```

---

## Scaling

```bash
# Scale horizontally (update instance_count in spec)
doctl apps spec get $APP_ID | \
  python3 -c "
import sys, yaml
spec = yaml.safe_load(sys.stdin)
spec['services'][0]['instance_count'] = 5
print(yaml.dump(spec))
" | doctl apps update $APP_ID --spec -

# Scale vertically (change instance_size_slug)
# Sizes: apps-s-1vcpu-0.5gb ($5), apps-s-1vcpu-1gb ($10),
#        apps-s-2vcpu-4gb ($25), apps-p-1vcpu-1gb (pro, dedicated)
```

---

## Alerts

```yaml
# In the service spec:
alert_policies:
  - rule: CPU_UTILIZATION
    operator: GREATER_THAN
    value: 80
    window: FIVE_MINUTES
  - rule: MEM_UTILIZATION
    operator: GREATER_THAN
    value: 90
    window: FIVE_MINUTES
  - rule: RESTART_COUNT
    operator: GREATER_THAN
    value: 3
    window: FIVE_MINUTES
  - rule: DEPLOYMENT_FAILED
  - rule: DEPLOYMENT_LIVE
```

---

## Gotchas and Guardrails

- **`deploy_on_push: true`** means every push to the branch triggers a build — use feature branches
- **`kind: PRE_DEPLOY` jobs** block the deploy if they exit non-zero — good for migrations
- **Static sites are free** on the Starter tier — great for docs, landing pages
- **Logs are ephemeral** — App Platform doesn't retain logs; pipe to Papertrail or Logtail
- **Secret env vars** can't be read back via API after setting — save them elsewhere
- **Health check path must return 2xx** — if missing, all traffic routes to the component anyway but restarts on failure
- **Instance count = 1** means a few seconds of downtime during deploys — set to 2+ for zero-downtime rolling
- **Build cache** is reused by default — `--force-rebuild` if your dependencies seem stale
- **App bandwidth is metered** — check egress pricing for high-traffic apps; CDN reduces it
- **`doctl apps spec validate`** before applying to catch schema errors
- **Managed databases in app spec** are provisioned and billed separately from the app itself
