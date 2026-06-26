---
name: cloudflare-dns
description: Cloudflare DNS — record types, proxied vs DNS-only, CNAME flattening, DNSSEC, API/Terraform automation, migration from other registrars.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare dns, dns, cname, dnssec, api, terraform
  version: 1.0.0
  updated: 2026-06-14
---
# Cloudflare DNS

Authoritative DNS for ~20% of the internet. Anycast, free, sub-10ms global resolution.

## The orange-cloud toggle

- 🟠 **Proxied**: traffic routes through CF (CDN, WAF, DDoS, SSL, Workers Routes). Returns CF anycast IPs in DNS. **Only HTTP/HTTPS and standard web ports.**
- ⚫ **DNS-only** (grey cloud): CF answers DNS but traffic goes straight to your origin. Use for SSH, SMTP, FTP, custom ports, mail records.

Rule of thumb: **A/AAAA/CNAME for web → proxied. Everything else → DNS-only.**

## Record types & gotchas

| Type | Notes |
|---|---|
| A / AAAA | Standard. Proxy-able. |
| CNAME | **CNAME flattening** at apex is automatic — you can CNAME `example.com` (not just `www`). |
| MX | Cannot be proxied. Set proxy off. |
| TXT | SPF, DKIM, DMARC, domain verification. |
| SRV | Service records (e.g. `_sip._tcp`). |
| CAA | Restrict which CAs can issue certs for your domain. |
| NS | Subdomain delegation only — root NS is managed by CF. |

## API (most common ops)

```bash
# List records
curl -H "Authorization: Bearer $CF_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?type=A"

# Create
curl -X POST -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"A","name":"api","content":"203.0.113.10","ttl":1,"proxied":true}' \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records"

# Update (PATCH for partial)
curl -X PATCH … -d '{"content":"203.0.113.20"}' \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID"

# Delete
curl -X DELETE … "…/dns_records/$RECORD_ID"
```

`ttl: 1` = "automatic" (CF chooses, typically 300s when proxied is irrelevant).

## Terraform

```hcl
resource "cloudflare_record" "api" {
  zone_id = var.zone_id
  name    = "api"
  content = "203.0.113.10"
  type    = "A"
  ttl     = 1
  proxied = true
}

resource "cloudflare_record" "mx" {
  zone_id  = var.zone_id
  name     = "@"
  type     = "MX"
  content  = "mx.improvmx.com"
  priority = 10
  proxied  = false        # NEVER proxy MX
}
```

## DNSSEC

1. Dashboard → DNS → Settings → Enable DNSSEC.
2. Copy DS record to your **registrar** (not CF).
3. Verify with `dig +dnssec example.com`.

Don't enable until DS is published at registrar — domain will go dark.

## Migrating from another registrar

1. Add site at CF → import zone (auto-detects most records).
2. **Verify every record** (especially MX, TXT, custom ports) is grey-cloud unless web traffic.
3. Lower TTLs at old provider 24h ahead.
4. Change NS at registrar to CF's assigned nameservers.
5. Wait for propagation (`dig NS example.com`) — usually <1h, up to 48h.
6. Don't disable old provider for ~72h in case of rollback.

## Bulk edits

API tokens with `Zone.DNS:Edit` scope. For bulk imports use **BIND zone file** upload in dashboard or terraform-managed records.
