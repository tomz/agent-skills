---
name: cloudflare-zero-trust
description: Cloudflare Zero Trust — Access (identity-aware proxy), Tunnel (cloudflared), Gateway (DNS/HTTP filtering), WARP client, Service Tokens.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare zero trust, dns, warp, access, tunnel, service tokens, identity-aware, machine-to-machine
  version: 1.0.0
  updated: 2026-06-26
---
# Cloudflare Zero Trust

Replace VPN + bastion + ACLs with identity-aware proxying. Free for ≤50 users.

## The pieces

| Product | Role |
|---|---|
| **Access** | Identity-aware proxy — gates apps behind SSO/MFA, no inbound ports. |
| **Tunnel** (`cloudflared`) | Outbound-only daemon connecting an origin to CF. No public IP needed. |
| **WARP** | Client agent — routes user traffic through CF for filtering/posture. |
| **Gateway** | DNS/HTTP/network egress filter for WARP users. |
| **Service Tokens** | Machine-to-machine auth bypassing browser SSO. |

## Tunnel — replace your VPN/bastion

```bash
# On the private host
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared

cloudflared tunnel login              # browser SSO to CF
cloudflared tunnel create homelab     # writes ~/.cloudflared/<UUID>.json
cloudflared tunnel route dns homelab grafana.example.com
```

`~/.cloudflared/config.yml`:

```yaml
tunnel: <UUID>
credentials-file: /etc/cloudflared/<UUID>.json
ingress:
  - hostname: grafana.example.com
    service: http://localhost:3000
  - hostname: ssh.example.com
    service: ssh://localhost:22
  - service: http_status:404
```

```bash
cloudflared service install         # systemd unit
systemctl enable --now cloudflared
```

The host now has **zero open inbound ports** and is reachable via CF anycast.

## Access policies

Dashboard → Zero Trust → Access → Applications → Add.

Policy example (allow staff + require MFA + device posture):

```
Action: Allow
Include:  Emails ending in @example.com
Require:  Auth method = MFA
Require:  Device posture: WARP installed AND disk encrypted
Session: 8 hours
```

The app sits behind `https://grafana.example.com` — users get an SSO page first.

## Service Tokens (CI/scripts)

```bash
# Create in dashboard, then:
curl https://grafana.example.com/api/health \
  -H "CF-Access-Client-Id: <id>.access" \
  -H "CF-Access-Client-Secret: <secret>"
```

In your Access policy, add `Include: Service Token = <name>`.

## SSH over Access (no public key needed)

```
Host prod
  HostName prod.example.com
  ProxyCommand cloudflared access ssh --hostname %h
```

With short-lived certs enabled, CF mints SSH certs scoped to identity — no `authorized_keys` management.

## Gateway DNS filter

For employee laptops with WARP:

- Block categories (gambling, malware, …)
- Allow/block lists per identity
- Decrypted HTTP inspection (requires CF root cert installed on device)

## Common pitfalls

- **Don't proxy** the tunnel hostname in DNS — `cloudflared tunnel route dns` handles it.
- WARP + corporate VPN often conflict; route only needed CIDRs through WARP.
- Access JWT is in `Cf-Access-Jwt-Assertion` header — verify it server-side too (defense in depth) using JWKS at `https://<team>.cloudflareaccess.com/cdn-cgi/access/certs`.
