---
name: cloudflare-terraform
description: Cloudflare IaC — terraform-provider-cloudflare v5 for zones, DNS, WAF, Workers, Pages, Access, account resources. Includes v4 to v5 notes.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare terraform, dns, waf, workers, pages, access, terraform-provider-cloudflare, account-level
  version: 1.1.0
  updated: 2026-06-14
---
# Cloudflare with Terraform

Use the official `cloudflare/cloudflare` provider. **v5 is GA** (since Feb 2025) and is the current major. The provider is now auto-generated from Cloudflare's OpenAPI spec — resource schemas align directly with API endpoints.

## Provider setup

```hcl
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 5"          # v5.19+ has automatic state upgraders
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token   # CLOUDFLARE_API_TOKEN env var also works
}

variable "account_id" { type = string }
variable "zone_id"    { type = string }
```

**Always use API tokens** (scoped) — never the global API key. Typical scopes: `Zone.DNS:Edit`, `Zone.Zone Settings:Edit`, `Zone.WAF:Edit`, `Account.Workers Scripts:Edit`, `Account.Access:Edit`.

## v4 → v5 migration (read this first if you have existing config)

v5 is a **ground-up rewrite**. 40+ resources were renamed and most attribute schemas changed (flat → nested objects in many places).

| v4 name | v5 name |
|---|---|
| `cloudflare_record` | `cloudflare_dns_record` |
| `cloudflare_tunnel` | `cloudflare_zero_trust_tunnel_cloudflared` |
| `cloudflare_access_application` | `cloudflare_zero_trust_access_application` |
| `cloudflare_access_policy` | `cloudflare_zero_trust_access_policy` |
| `cloudflare_workers_script` | `cloudflare_workers_script` (same, but schema changed) |

Recommended path:

```bash
# 1. Pin v4.52.5 first to apply transitional state updates
# 2. Run Cloudflare's tf-migrate CLI:
tf-migrate migrate --source-version v4 --target-version v5 --dry-run
tf-migrate migrate --source-version v4 --target-version v5

# 3. Bump provider to ~> 5
terraform init -upgrade
terraform plan          # state upgraders run automatically (v5.19+)
terraform apply
```

`tf-migrate` rewrites HCL, updates cross-file references, and emits `moved` blocks for renamed resources (requires Terraform ≥ 1.8). Back up state first: `terraform state pull > terraform.tfstate.backup`.

Manual moved-block example:

```hcl
moved {
  from = cloudflare_record.example
  to   = cloudflare_dns_record.example
}
```

## Zone + DNS (v5)

```hcl
resource "cloudflare_zone" "primary" {
  account = { id = var.account_id }   # v5: nested object, not flat account_id
  name    = "example.com"
  type    = "full"
}

resource "cloudflare_zone_setting" "ssl" {
  zone_id    = cloudflare_zone.primary.id
  setting_id = "ssl"
  value      = "strict"
}
# (per-setting resource in v5 — no more single `cloudflare_zone_settings_override`)

locals {
  dns_records = {
    "@"   = { type = "A",     content = "203.0.113.10", proxied = true  }
    "www" = { type = "CNAME", content = "example.com",  proxied = true  }
    "mx"  = { type = "MX",    content = "mx.example.",  proxied = false, priority = 10 }
  }
}

resource "cloudflare_dns_record" "rec" {
  for_each = local.dns_records
  zone_id  = cloudflare_zone.primary.id
  name     = each.key
  type     = each.value.type
  content  = each.value.content
  proxied  = each.value.proxied
  priority = try(each.value.priority, null)
  ttl      = 1
}
```

## WAF custom rules (Rulesets engine)

```hcl
resource "cloudflare_ruleset" "custom_waf" {
  zone_id = cloudflare_zone.primary.id
  name    = "custom waf"
  kind    = "zone"
  phase   = "http_request_firewall_custom"

  rules = [
    {
      action      = "block"
      expression  = "(http.request.uri.path contains \"/admin\" and not ip.src in {203.0.113.0/24})"
      description = "Restrict /admin to office IPs"
      enabled     = true
    },
    {
      action      = "managed_challenge"
      expression  = "(cf.bot_management.score < 30 and not cf.bot_management.verified_bot)"
      description = "Challenge low-score bots"
      enabled     = true
    },
  ]
}
```

> In v5, `rules` is a **list attribute** (`= [ ... ]`), not repeated `rules { }` blocks. This is the most common gotcha when porting v4 modules by hand.

## Rate limiting

```hcl
resource "cloudflare_ruleset" "ratelimit" {
  zone_id = cloudflare_zone.primary.id
  name    = "rate limits"
  kind    = "zone"
  phase   = "http_ratelimit"

  rules = [{
    action      = "block"
    expression  = "(http.request.uri.path eq \"/api/login\")"
    description = "Login brute force"
    ratelimit = {
      characteristics     = ["ip.src", "cf.colo.id"]
      period              = 60
      requests_per_period = 5
      mitigation_timeout  = 600
    }
  }]
}
```

## Workers & Pages

```hcl
resource "cloudflare_workers_script" "api" {
  account_id  = var.account_id
  script_name = "api"
  content     = file("${path.module}/../dist/api.js")
  main_module = "api.js"

  bindings = [
    { name = "CACHE",       type = "kv_namespace", namespace_id = cloudflare_workers_kv_namespace.cache.id },
    { name = "API_VERSION", type = "plain_text",   text         = "v1" },
    { name = "DB_PASSWORD", type = "secret_text",  text         = var.db_password },
  ]
}

resource "cloudflare_workers_route" "api" {
  zone_id     = cloudflare_zone.primary.id
  pattern     = "api.example.com/*"
  script      = cloudflare_workers_script.api.script_name
}

resource "cloudflare_workers_kv_namespace" "cache" {
  account_id = var.account_id
  title      = "cache"
}
```

> v5 unified bindings into a single `bindings = [ ... ]` list with a `type` discriminator. v4's `kv_namespace_binding { ... }`, `plain_text_binding { ... }`, etc. blocks are gone.

## Access application (Zero Trust)

```hcl
resource "cloudflare_zero_trust_access_application" "grafana" {
  zone_id          = cloudflare_zone.primary.id
  name             = "Grafana"
  domain           = "grafana.example.com"
  type             = "self_hosted"
  session_duration = "8h"
}

resource "cloudflare_zero_trust_access_policy" "grafana_staff" {
  application_id = cloudflare_zero_trust_access_application.grafana.id
  zone_id        = cloudflare_zone.primary.id
  name           = "Staff only"
  decision       = "allow"

  include = [{ email_domain = { domain = "example.com" } }]
  require = [{ auth_method  = { auth_method = "mfa" } }]
}
```

## Tips

- Use `for_each` with maps over `count` — stable addresses survive reordering.
- Pull existing zones with `terraform import cloudflare_zone.primary <zone_id>` rather than recreating.
- The provider rate-limits at ~1200 req/5min — chunk huge DNS imports.
- **List vs block syntax**: v5 uses list assignments (`rules = [{...}]`) for most repeated structures. v4 used HCL `rules { }` blocks. Mixing the two is the #1 v4→v5 plan error.
- Pin the provider version (`~> 5`). Major bumps involve resource renames; minor (5.x → 5.y) generally don't.
- **State upgraders** are automatic from v5.19+. If on v5.0–v5.18 you may need `terraform state mv` manually — see the [v5 migration guide](https://github.com/cloudflare/terraform-provider-cloudflare/blob/main/docs/guides/version-5-migration.md).
- `cloudflare_record.value` (v3) → `cloudflare_record.content` (v4) → `cloudflare_dns_record.content` (v5). Each major shuffles this.
