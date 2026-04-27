---
name: do-cli
description: DigitalOcean doctl CLI — authentication, contexts, output formats, and core subcommand patterns
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

# DigitalOcean CLI (doctl) Skill

## Overview

`doctl` is the official DigitalOcean CLI. It wraps the DO API and supports every resource type:
compute, databases, Kubernetes, networking, App Platform, and more. Install once, authenticate,
and you can script anything you'd do in the control panel.

---

## Installation

```bash
# macOS
brew install doctl

# Linux (amd64)
curl -sL https://github.com/digitalocean/doctl/releases/latest/download/doctl-$(curl -s https://api.github.com/repos/digitalocean/doctl/releases/latest | grep tag_name | cut -d'"' -f4 | tr -d v)-linux-amd64.tar.gz | tar xz
sudo mv doctl /usr/local/bin/

# Verify
doctl version
```

---

## Authentication

### Basic: single account

```bash
# Interactive — opens browser or prompts for token
doctl auth init

# Non-interactive (CI/CD, scripts)
doctl auth init --access-token $DO_TOKEN
```

### Multiple accounts: named contexts

```bash
# Create a context for each account
doctl auth init --context personal
doctl auth init --context work --access-token $WORK_DO_TOKEN

# List contexts
doctl auth list

# Switch active context
doctl auth switch --context work

# One-off command in a different context
doctl compute droplet list --context personal
```

### Token management

- Tokens are stored in `~/.config/doctl/config.yaml`
- Create tokens at: https://cloud.digitalocean.com/account/api/tokens
- Scopes: `read` (safe for monitoring) vs `write` (full control)
- For CI/CD: set `DIGITALOCEAN_ACCESS_TOKEN` env var (doctl picks it up automatically)

```bash
# Verify your token works
doctl account get
```

---

## Output Formats

doctl defaults to human-readable table output. Change it with `--output`:

```bash
# Default table
doctl compute droplet list

# JSON — best for scripting, jq, automation
doctl compute droplet list --output json

# YAML — good for config files
doctl compute droplet list --output yaml

# Specific columns in table mode
doctl compute droplet list --format ID,Name,PublicIPv4,Status,Region,Size

# No header row (pipe-friendly)
doctl compute droplet list --no-header

# Combine: just IPs, no header
doctl compute droplet list --format PublicIPv4 --no-header
```

### Useful jq patterns with doctl

```bash
# Get all droplet IDs
doctl compute droplet list --output json | jq '.[].id'

# Find droplets by tag
doctl compute droplet list --tag-name production --output json | jq '.[].name'

# Get a droplet's IP by name
doctl compute droplet list --output json | jq -r '.[] | select(.name=="web-01") | .networks.v4[] | select(.type=="public") | .ip_address'
```

---

## Core Subcommand Tree

```
doctl
├── account          — account info, rate limits, team
├── auth             — authentication and contexts
├── compute          — Droplets, volumes, firewalls, snapshots, SSH keys
│   ├── droplet
│   ├── droplet-action
│   ├── volume
│   ├── volume-action
│   ├── firewall
│   ├── image
│   ├── region
│   ├── size
│   ├── ssh-key
│   ├── snapshot
│   ├── reserved-ip
│   └── tag
├── databases        — managed PostgreSQL, MySQL, Redis, MongoDB, Kafka
├── kubernetes       — DOKS clusters, node pools, kubeconfig
│   └── cluster
│       └── node-pool
├── registry         — DigitalOcean Container Registry (DOCR)
├── apps             — App Platform apps, deployments, logs
├── networking       — VPC, load balancers, domains, CDN
│   ├── vpc
│   ├── load-balancer
│   ├── domain
│   └── cdn
└── monitoring       — alert policies, notification destinations
```

---

## Common Patterns

### Scripting with doctl

```bash
# Delete all droplets matching a pattern (dry-run first!)
doctl compute droplet list --output json | \
  jq -r '.[] | select(.name | startswith("test-")) | .id' | \
  xargs -I{} echo "Would delete: {}"  # remove echo to execute

# Create and wait for a droplet to be active
DROPLET_ID=$(doctl compute droplet create web-01 \
  --image ubuntu-24-04-x64 --size s-1vcpu-1gb --region nyc3 \
  --output json --no-wait | jq -r '.[0].id')
doctl compute droplet-action wait $DROPLET_ID --action-type create
```

### Pagination

doctl handles pagination automatically — you always get all results, not just the first page.
Use `--output json` for large lists to avoid truncation in table mode.

### Rate limits

```bash
# Check your current rate limit status
doctl account get --output json | jq '.rate_limit_remaining, .rate_limit_limit'
```

---

## Config File

`~/.config/doctl/config.yaml` — edit with caution:

```yaml
access-token: dop_v1_...
context: default
contexts:
  default:
    access-token: dop_v1_...
  work:
    access-token: dop_v1_...
```

---

## Guardrails

- **Never commit tokens** — use environment variables or secret managers
- **Test with `--no-wait`** and check before waiting — creation can fail
- **Use `--output json` in scripts** — table format changes between versions
- **Tag resources** — makes bulk operations safe (`--tag-name env:prod`)
- **`doctl compute region list`** to see valid region slugs before scripting
- **`doctl compute size list`** to see valid size slugs — prices are included
- **`doctl compute image list --public`** to find OS image slugs

---

## Quick Reference

```bash
# Account info
doctl account get

# List all regions
doctl compute region list

# List all sizes with pricing
doctl compute size list

# List public images (OS options)
doctl compute image list --public | grep ubuntu

# Get help for any subcommand
doctl compute droplet create --help
doctl databases --help
```
