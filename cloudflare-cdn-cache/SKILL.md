---
name: cloudflare-cdn-cache
description: Cloudflare CDN and cache control — Cache Rules, cache keys, tiered caching, Cache API in Workers, purge strategies.
triggers: cloudflare, cloudflare cdn cache, cdn, api, cloudflare cdn, cache rules, page rules, cache api, workers
license: MIT
version: 1.0.0
updated: 2026-05-12
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---
# Cloudflare CDN & Cache

## Default behavior

CF only caches **static file extensions by default** (`.css`, `.js`, `.jpg`, `.png`, `.pdf`, etc.) — see the [default cache list]. HTML and unknown extensions are **not cached** unless you configure it.

Origin headers respected (when matched):

- `Cache-Control: public, max-age=N` — CF edge cache + browser
- `Cache-Control: s-maxage=N` — CF edge only
- `Cache-Control: private` / `no-store` — bypass
- `Cache-Control: stale-while-revalidate=N` — serve stale while refreshing

## Cache Rules (replaces Page Rules)

Dashboard → Caching → Cache Rules. **Page Rules are in legacy mode** — still functional but Cloudflare actively pushes migration to Cache Rules / Configuration Rules / Origin Rules. New work should use Cache Rules.

Example: cache HTML for 5 minutes on `/blog/*`:

```
When:  (http.host eq "example.com" and starts_with(http.request.uri.path, "/blog/"))
Then:
  - Cache eligibility: Eligible for cache
  - Edge TTL: 5 minutes (Override origin)
  - Browser TTL: Respect origin
  - Cache key: Include query string "page,tag"; ignore everything else
```

## Cache key engineering

Most wins come from a tight cache key:

- **Strip tracking params** (`utm_*`, `fbclid`, `gclid`) — otherwise cache fragments per click.
- **Bucket cookies** — include only auth cookies in the key, not session/analytics.
- **Vary by device** if you serve different markup for mobile (rarely needed with responsive design).

## Tiered Cache & Argo

- **Tiered Cache (free, enabled by default for new zones)**: edge PoPs request from regional upper-tier PoP instead of origin → reduces origin load 50–80%.
- **Argo Smart Routing (paid)**: faster origin path over CF backbone.
- **Cache Reserve (paid)**: R2-backed cache for assets that fall out of edge LRU — near-100% offload for long-tail content.

## Purge

```bash
# Single URL
curl -X POST -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  -d '{"files":["https://example.com/path/style.css"]}' \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache"

# By tag (requires Cache-Tag header on origin response; Enterprise)
-d '{"tags":["product-123"]}'

# By prefix (Enterprise)
-d '{"prefixes":["example.com/blog/"]}'

# Everything (use sparingly — origin stampede risk)
-d '{"purge_everything":true}'
```

**Always prefer tag/prefix over purge-all** on busy sites — purge-all causes origin thundering herd.

## Workers Cache API

For custom caching inside a Worker:

```ts
export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    const cache = caches.default;
    const cacheKey = new Request(new URL(req.url).toString(), req);

    let res = await cache.match(cacheKey);
    if (res) return res;

    res = await fetch('https://origin/' + new URL(req.url).pathname);
    res = new Response(res.body, res);
    res.headers.set('Cache-Control', 'public, max-age=3600');
    ctx.waitUntil(cache.put(cacheKey, res.clone()));
    return res;
  },
};
```

## Debugging

Headers to inspect (`curl -I` or DevTools):

| Header | Meaning |
|---|---|
| `cf-cache-status: HIT` | served from edge |
| `MISS` | fetched from origin, now cached |
| `EXPIRED` | stale → revalidated |
| `BYPASS` | rule said don't cache |
| `DYNAMIC` | extension/content-type not in default cache list, no rule overrode |
| `REVALIDATED` | conditional GET hit origin, no body |

`DYNAMIC` on HTML is the #1 "why isn't my site cached?" → add a Cache Rule.
