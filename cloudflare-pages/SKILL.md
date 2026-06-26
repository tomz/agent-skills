---
name: cloudflare-pages
description: Cloudflare Pages — JAMstack hosting with git-based deploys, Pages Functions, preview URLs, custom domains, integration with Workers, KV, R2.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare pages, kv, r2, d1, jamstack, pages functions, workers, git-based
  version: 1.0.0
  updated: 2026-06-14
---
# Cloudflare Pages

Git-driven static + serverless hosting. Every push → preview URL; main → prod.

## When to use Pages vs Workers

- **Pages**: static sites, SSG (Next.js export, Astro, Hugo), apps with light dynamic needs.
- **Workers**: pure API / heavy edge logic / Durable Objects / queues.
- Modern recommendation (2024+): use **Workers with Static Assets** for new projects; Pages remains supported.

## Project structure

```
my-site/
├── public/                  # or dist/, out/, build/ depending on framework
├── functions/               # Pages Functions (file-routed)
│   ├── api/
│   │   └── hello.ts         # → /api/hello
│   └── [[catchall]].ts
└── wrangler.toml
```

## Pages Function

```ts
// functions/api/users/[id].ts
export const onRequestGet: PagesFunction<Env> = async ({ params, env }) => {
  const user = await env.DB.prepare('SELECT * FROM users WHERE id=?')
    .bind(params.id).first();
  return Response.json(user);
};

export const onRequestPost: PagesFunction<Env> = async ({ request, env }) => {
  const body = await request.json();
  // …
  return new Response(null, { status: 201 });
};
```

Middleware (`functions/_middleware.ts`):

```ts
export const onRequest: PagesFunction = async ({ request, next }) => {
  const res = await next();
  res.headers.set('x-frame-options', 'DENY');
  return res;
};
```

## Build configuration

Dashboard → Pages → project → Settings → Builds.

| Framework | Build cmd | Output dir |
|---|---|---|
| Next.js (static export) | `next build` | `out` |
| Astro | `astro build` | `dist` |
| SvelteKit | `npm run build` | `.svelte-kit/cloudflare` (use adapter-cloudflare) |
| Hugo | `hugo` | `public` |
| Vite | `vite build` | `dist` |

## Deploy via CLI

```bash
wrangler pages deploy ./dist --project-name=my-site
wrangler pages dev ./dist                # local preview
wrangler pages deployment list
```

## Preview deployments

- Every PR / branch gets `<commit>.<project>.pages.dev`.
- Use preview-only env vars (set in dashboard) to point at staging APIs.
- Pages does **not** auto-purge old previews — they remain accessible.

## Headers & redirects

`public/_headers`:

```
/*
  X-Frame-Options: DENY
  Strict-Transport-Security: max-age=31536000

/api/*
  Cache-Control: no-store
```

`public/_redirects`:

```
/old/*  /new/:splat  301
/app/*  /spa/index.html  200    # SPA fallback
```

## Gotchas

- Functions run as Workers — same CPU/subrequest limits apply.
- `process.env` is **not** populated; use `env` parameter.
- Output dir must be **relative**, no leading `/`.
- Custom domains need a CF-managed zone, or you use saas-style fronting.
