---
name: cloudflare-workers
description: Cloudflare Workers — serverless edge compute on V8 isolates. wrangler, fetch handlers, bindings, Durable Objects, KV/R2/D1, cron triggers.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare workers, v8, cli, kv, r2, d1, durable objects
  version: 1.0.0
  updated: 2026-06-14
---
# Cloudflare Workers

Serverless functions running on V8 isolates at 300+ edge PoPs. Cold-start ~0ms.

## Project layout

```
my-worker/
├── src/index.ts
├── wrangler.toml
└── package.json
```

`wrangler.toml`:

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2025-01-01"
compatibility_flags = ["nodejs_compat"]

[vars]
API_VERSION = "v1"

[[kv_namespaces]]
binding = "CACHE"
id = "abc123…"

[[r2_buckets]]
binding = "ASSETS"
bucket_name = "my-assets"

[[d1_databases]]
binding = "DB"
database_name = "prod"
database_id = "…"

[triggers]
crons = ["0 * * * *"]
```

## Fetch handler (TypeScript)

```ts
export interface Env {
  CACHE: KVNamespace;
  ASSETS: R2Bucket;
  DB: D1Database;
  API_VERSION: string;
  SECRET_KEY: string;   // wrangler secret put SECRET_KEY
}

export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(req.url);

    // Cache lookup
    const cached = await env.CACHE.get(url.pathname);
    if (cached) return new Response(cached, { headers: { 'cf-cache': 'HIT' } });

    const result = await env.DB.prepare('SELECT * FROM items WHERE slug = ?')
      .bind(url.pathname).first();

    const body = JSON.stringify(result);
    // Background write — don't block response
    ctx.waitUntil(env.CACHE.put(url.pathname, body, { expirationTtl: 300 }));
    return new Response(body, { headers: { 'content-type': 'application/json' } });
  },

  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    ctx.waitUntil(rebuildSitemap(env));
  },
};
```

## Commands

```bash
npm create cloudflare@latest          # scaffold
wrangler dev                          # local dev with miniflare
wrangler dev --remote                 # run against real CF edge
wrangler deploy                       # ship to prod
wrangler tail                         # live logs
wrangler secret put API_KEY           # encrypted env var
wrangler kv:key put --binding=CACHE foo bar
```

## Limits & gotchas

- **CPU**: 10 ms Free / 30 s default on Paid, raisable to 5 min via `[limits] cpu_ms = 300000`. `Date.now()` only advances on I/O.
- **Memory**: 128 MB per isolate (both tiers).
- **Subrequests**: 50 Free / 10,000 Paid per request (also called "outgoing fetches").
- **Worker size**: 3 MB Free / 10 MB Paid (compressed).
- **No Node APIs** without `nodejs_compat` flag — no `fs`, no native sockets (use `connect()` from `cloudflare:sockets`).
- **Global scope is reused** across requests in the same isolate — never store user data there.
- Use `ctx.waitUntil()` for fire-and-forget work after responding.

## Durable Objects (stateful)

For coordination/strong consistency. One instance per ID globally.

```ts
export class Counter {
  constructor(private state: DurableObjectState, private env: Env) {}
  async fetch(req: Request) {
    let n = (await this.state.storage.get<number>('n')) ?? 0;
    n++;
    await this.state.storage.put('n', n);
    return new Response(String(n));
  }
}
```

```toml
[[durable_objects.bindings]]
name = "COUNTER"
class_name = "Counter"

[[migrations]]
tag = "v1"
new_classes = ["Counter"]
```

## Testing

- `vitest` + `@cloudflare/vitest-pool-workers` runs tests inside miniflare with real bindings.
- Avoid `wrangler dev` for unit tests — too slow. Use pool-workers for fast iteration.
