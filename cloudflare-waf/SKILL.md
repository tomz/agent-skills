---
name: cloudflare-waf
description: Cloudflare WAF and security — Custom Rules, Managed Rules, rate limiting, bot management, Turnstile CAPTCHA, and incident response workflow.
triggers: cloudflare, cloudflare waf, waf, custom rules, managed rules, turnstile captcha
license: MIT
version: 1.0.0
updated: 2026-05-12
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---
# Cloudflare WAF & Security

## Rule families (evaluation order)

1. **IP Access Rules** (Security → WAF → Tools): blanket allow/block by IP/ASN/country.
2. **Custom Rules**: your business logic, written in CF's expression language.
3. **Rate Limiting Rules**: throttle by counter (IP, header, cookie).
4. **Managed Rules**: CF-maintained signatures (OWASP, Cloudflare Managed Ruleset, exposed-credential check).
5. **Bot Management**: ML-driven bot score (1=bot, 99=human).

## Custom Rule examples

```
# Block path traversal attempts to /admin
Expression: (http.request.uri.path contains "/admin" and http.request.uri.query contains "../")
Action:     Block

# Challenge requests with no User-Agent
Expression: (http.user_agent eq "")
Action:     Managed Challenge

# Allow office IPs to bypass all WAF
Expression: (ip.src in {203.0.113.0/24 198.51.100.42})
Action:     Skip → all remaining custom rules + managed rulesets

# Geo-block but allow CDN origins
Expression: (ip.geoip.country in {"CN" "RU" "KP"} and not cf.client.bot)
Action:     Block
```

Fields cheat-sheet: `http.request.method`, `http.host`, `http.request.uri.path`, `http.request.uri.query`, `http.user_agent`, `http.request.headers["x-foo"]`, `ip.src`, `ip.geoip.country`, `ip.geoip.asnum`, `ssl`, `cf.threat_score`, `cf.bot_management.score`, `cf.bot_management.verified_bot`.

## Rate Limiting

```
Expression: (http.request.uri.path eq "/api/login")
Counting:   characteristic = IP, period = 60s
Threshold:  5 requests
Action:     Block for 600s
# Optional: response code 429 with custom JSON body
```

For logged-in throttling, count by `http.request.headers["authorization"]` or a session cookie instead of IP.

## Managed Rules

Enable the **Cloudflare Managed Ruleset** + **OWASP Core Ruleset**. Start in `Log` action sitewide for 24–48h, then triage false positives via the Security Events log → exclude specific rules per path:

```
When: (http.request.uri.path eq "/webhook")
Action: Skip → managed rule id 100001 (or set sensitivity = Low for that path)
```

## Bot Management

- **Free tier**: "Bot Fight Mode" (blunt; can break legitimate API clients).
- **Super Bot Fight Mode** (Pro): tiered actions for definitely-automated vs likely-automated.
- **Bot Management** (Enterprise): per-request ML score, JS detections, mobile SDK.

```
Expression: (cf.bot_management.score < 30 and not cf.bot_management.verified_bot)
Action:     Managed Challenge
```

`verified_bot` = Googlebot, Bingbot, etc. — never block these unless intentional.

## Turnstile (CAPTCHA replacement)

Free, privacy-friendly. Drop-in for forms:

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
<form method="POST">
  <div class="cf-turnstile" data-sitekey="0x4AAA…"></div>
  <button>Submit</button>
</form>
```

Server-side verify:

```ts
const form = await req.formData();
const token = form.get('cf-turnstile-response');
const r = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
  method: 'POST',
  body: new URLSearchParams({ secret: env.TURNSTILE_SECRET, response: token, remoteip: req.headers.get('cf-connecting-ip') ?? '' }),
});
const { success } = await r.json();
if (!success) return new Response('CAPTCHA failed', { status: 403 });
```

## Incident response playbook

1. **Identify** the attack pattern via Security → Events (filter by action=Block/Challenge, sort by Top N).
2. **Mitigate immediately**: "Under Attack Mode" (Security → Settings) → JS challenge on every request. Brutal but buys time.
3. **Narrow the rule**: write a Custom Rule targeting the exact pattern (URI, ASN, UA fingerprint).
4. **Disable Under Attack Mode** once the targeted rule is catching the traffic.
5. **Postmortem**: export logs (Logpush → R2/S3), refine managed-ruleset exclusions.

## Headers reference

Every proxied request to your origin includes:

- `CF-Connecting-IP`: real client IP — **trust this, not `X-Forwarded-For`**.
- `CF-IPCountry`: ISO-3166-1 alpha-2.
- `CF-Ray`: unique request ID; quote this in support tickets.
- `CF-Worker`: present if a Worker handled it.
