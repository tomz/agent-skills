---
name: cloudflare-kv
description: Cloudflare Workers KV — eventually-consistent global key-value store. When to use vs D1/DO/R2, TTL/metadata, list pagination, consistency.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare kv, kv, d1, r2, ttl, cloudflare workers kv, eventually-consistent, key-value
  version: 1.1.0
  updated: 2026-06-26
---
# Workers KV

Global, low-latency, **eventually consistent** key-value store. Optimized for read-heavy workloads.

## When to use what

| Need | Use |
|---|---|
| Config / feature flags / sessions read 1000× per write | **KV** |
| Relational queries, joins, indexes | **D1** |
| Strong consistency, per-key coordination | **Durable Objects** |
| Files / blobs >25 MB | **R2** |

**KV writes take up to 60s to propagate globally.** Don't use for primary state.

## Setup

```bash
wrangler kv namespace create CACHE
wrangler kv namespace create CACHE --preview
```

```toml
[[kv_namespaces]]
binding = "CACHE"
id = "prod-id"
preview_id = "preview-id"
```

## CRUD

```ts
// PUT
await env.CACHE.put('user:123', JSON.stringify(user), {
  expirationTtl: 3600,                          // seconds; min 60
  metadata: { tier: 'premium', v: 2 },          // ≤1024 bytes, returned with list/get
});

// GET
const raw = await env.CACHE.get('user:123');
const obj = await env.CACHE.get('user:123', 'json');
const { value, metadata } = await env.CACHE.getWithMetadata<User, Meta>('user:123', 'json');

// DELETE
await env.CACHE.delete('user:123');

// LIST (paginated)
let cursor: string | undefined;
do {
  const page = await env.CACHE.list({ prefix: 'user:', cursor, limit: 1000 });
  for (const k of page.keys) {
    // k.name, k.expiration, k.metadata
  }
  cursor = page.list_complete ? undefined : page.cursor;
} while (cursor);
```

## CLI

```bash
wrangler kv key put --binding=CACHE foo bar
wrangler kv key put --binding=CACHE foo bar --ttl=600
wrangler kv key get --binding=CACHE foo
wrangler kv key list --binding=CACHE --prefix=user:
wrangler kv bulk put --binding=CACHE ./pairs.json
```

## Limits

- Value: **25 MiB** max
- Key: 512 bytes max
- Metadata: 1024 bytes max
- 1 write/sec to the **same key** (others throttled)
- Free tier: 100k reads/day, 1k writes/day, 1 GB storage. Paid: unlimited reads/writes, unlimited storage.
- 1,000 KV ops per Worker invocation (shared with other external services)
- Reads: unlimited from cache; cold reads ~50 ms, hot reads <10 ms

## Patterns

### Cache-aside

```ts
async function getUser(id: string, env: Env) {
  const cached = await env.CACHE.get(`user:${id}`, 'json');
  if (cached) return cached;
  const fresh = await env.DB.prepare('SELECT * FROM users WHERE id=?').bind(id).first();
  ctx.waitUntil(env.CACHE.put(`user:${id}`, JSON.stringify(fresh), { expirationTtl: 300 }));
  return fresh;
}
```

### Feature flags

KV is ideal: 1 write when ops toggles a flag, millions of reads from Workers — eventual consistency is fine for flags.

### Anti-patterns

- **Counter / leaderboard** → use Durable Object (KV writes lose updates).
- **User session that must be invalidated immediately on logout** → use DO or short TTL.
- **Anything requiring read-your-writes** → KV is the wrong tool.
