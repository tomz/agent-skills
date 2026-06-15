---
name: cloudflare-wrangler
description: Cloudflare Wrangler CLI — unified tool for Workers, Pages, R2, KV, D1, Queues, secrets, tail logs, and local dev with Miniflare.
triggers: cloudflare, cloudflare wrangler, cli, r2, kv, d1, cloudflare wrangler cli, workers, pages, queues, miniflare, command-line
license: MIT
version: 1.0.0
updated: 2026-06-14
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---
# Wrangler CLI

The single CLI for every Cloudflare developer-platform product. Wraps the CF API, manages `wrangler.toml` / `wrangler.jsonc`, and runs a local Miniflare-backed dev environment.

**Current major: Wrangler v4** (GA March 2025). v3 receives bug fixes until Q1 2026, security-only until Q1 2027. Migration from v3 → v4 is small; most workflows are unchanged.

## Install & auth

```bash
npm install -D wrangler            # project-local (recommended)
npx wrangler --version
# or globally:
npm install -g wrangler

wrangler login                     # browser OAuth, writes ~/.wrangler/config/default.toml
wrangler whoami
wrangler logout
```

**CI / non-interactive auth:** set `CLOUDFLARE_API_TOKEN` (scoped token, never the global key). Optional `CLOUDFLARE_ACCOUNT_ID` to skip the picker when you belong to multiple accounts.

```bash
export CLOUDFLARE_API_TOKEN="…"
export CLOUDFLARE_ACCOUNT_ID="…"
wrangler deploy
```

## Project bootstrap

```bash
npm create cloudflare@latest my-app    # interactive scaffold (Workers/Pages, TS, framework)
cd my-app && npm install
wrangler dev
```

## `wrangler.toml` essentials

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2025-01-01"
compatibility_flags = ["nodejs_compat"]
workers_dev = true                    # *.workers.dev subdomain
account_id = "…"                      # optional; uses env/login otherwise

[vars]
LOG_LEVEL = "info"

[[kv_namespaces]]
binding = "CACHE"
id = "prod-id"
preview_id = "preview-id"

[[r2_buckets]]
binding = "ASSETS"
bucket_name = "my-assets"

[[d1_databases]]
binding = "DB"
database_name = "prod"
database_id = "…"

[triggers]
crons = ["*/5 * * * *"]

# Per-environment overrides
[env.staging]
name = "my-worker-staging"
vars = { LOG_LEVEL = "debug" }

[env.production]
routes = [{ pattern = "api.example.com/*", zone_name = "example.com" }]
```

Validate before deploy:

```bash
wrangler types                     # generate env types into worker-configuration.d.ts
wrangler deploy --dry-run --outdir=dist
```

## Local development

```bash
wrangler dev                       # localhost:8787, Miniflare-backed, live reload
wrangler dev --remote              # run on real CF edge (real bindings, ~slower iteration)
wrangler dev --port 3000 --ip 0.0.0.0
wrangler dev --persist-to=.wrangler/state    # persist KV/R2/D1 state across runs
wrangler dev --test-scheduled      # POST /__scheduled to trigger crons manually
wrangler dev --env staging         # use [env.staging] bindings
```

## Deploy

```bash
wrangler deploy                    # production
wrangler deploy --env staging
wrangler deploy --minify
wrangler deploy --dry-run --outdir=dist
wrangler deployments list          # last 10 versions
wrangler rollback                  # interactive — pick a version to roll back to
wrangler rollback <version-id>
```

**Gradual deploys / versioning:**

```bash
wrangler versions upload           # upload but don't route traffic
wrangler versions deploy           # route % of traffic — supports 10/50/100 splits
```

## Secrets vs vars

- `[vars]` in `wrangler.toml` → **plaintext**, committed to git. Use for non-sensitive config.
- Secrets → encrypted, set out-of-band:

```bash
wrangler secret put STRIPE_KEY                # prompts for value
wrangler secret put STRIPE_KEY --env staging
echo "$VAL" | wrangler secret put STRIPE_KEY  # from stdin (CI)
wrangler secret list
wrangler secret delete STRIPE_KEY
wrangler secret bulk secrets.json             # { "K1": "v1", "K2": "v2" }
```

## Logs (`tail`)

```bash
wrangler tail                                    # stream live logs
wrangler tail --format=pretty
wrangler tail --status=error                     # only failures
wrangler tail --search "user-123"                # substring filter
wrangler tail --sampling-rate 0.1                # 10% sample for high-traffic Workers
wrangler tail --ip-address self                  # only requests from your IP
```

For persistent logs, configure **Logpush** (R2/S3/Datadog) in the dashboard — `tail` is ephemeral.

## KV

```bash
wrangler kv namespace create CACHE
wrangler kv namespace create CACHE --preview
wrangler kv namespace list

wrangler kv key put --binding=CACHE foo bar
wrangler kv key put --binding=CACHE foo bar --ttl=3600
wrangler kv key put --binding=CACHE foo @./payload.json --path
wrangler kv key get --binding=CACHE foo
wrangler kv key list --binding=CACHE --prefix=user:
wrangler kv key delete --binding=CACHE foo

wrangler kv bulk put --binding=CACHE pairs.json   # [{"key":"k","value":"v","expiration_ttl":60}]
wrangler kv bulk delete --binding=CACHE keys.json
```

Add `--local` to operate on the local Miniflare store; `--remote` for production.

## R2

```bash
wrangler r2 bucket create my-bucket
wrangler r2 bucket list
wrangler r2 bucket delete my-bucket

wrangler r2 object put my-bucket/key.jpg --file=./local.jpg
wrangler r2 object get my-bucket/key.jpg --file=./out.jpg
wrangler r2 object delete my-bucket/key.jpg

wrangler r2 bucket dev-url enable my-bucket       # public r2.dev URL (dev only)
wrangler r2 bucket cors set my-bucket --rules=cors.json
wrangler r2 bucket lifecycle add my-bucket --id=expire-tmp --prefix=tmp/ --expire-days=7
```

## D1

```bash
wrangler d1 create my-db
wrangler d1 list
wrangler d1 info my-db

wrangler d1 execute my-db --command="SELECT 1"
wrangler d1 execute my-db --file=./schema.sql --local
wrangler d1 execute my-db --file=./schema.sql --remote

wrangler d1 migrations create my-db add_users
wrangler d1 migrations apply my-db --local
wrangler d1 migrations apply my-db --remote
wrangler d1 migrations list my-db

wrangler d1 export my-db --remote --output=backup.sql
wrangler d1 export my-db --remote --no-data --output=schema.sql
wrangler d1 time-travel restore my-db --timestamp=2025-01-01T00:00:00Z
```

## Pages

```bash
wrangler pages project create my-site --production-branch=main
wrangler pages project list

wrangler pages deploy ./dist                     # ad-hoc deploy
wrangler pages deploy ./dist --project-name=my-site --branch=preview
wrangler pages dev ./dist                        # local preview
wrangler pages dev -- npm run dev                # proxy a framework dev server
wrangler pages deployment list --project-name=my-site
wrangler pages deployment tail --project-name=my-site
```

## Queues

```bash
wrangler queues create my-queue
wrangler queues list
wrangler queues consumer add my-queue my-worker
wrangler queues delete my-queue
```

```toml
[[queues.producers]]
binding = "QUEUE"
queue   = "my-queue"

[[queues.consumers]]
queue          = "my-queue"
max_batch_size = 10
max_retries    = 3
dead_letter_queue = "my-queue-dlq"
```

## Useful flags & env vars

| Flag / env | Effect |
|---|---|
| `--env <name>` | Apply `[env.<name>]` block from `wrangler.toml` |
| `--config path/to/wrangler.toml` | Use non-default config |
| `--var KEY:VALUE` | One-shot var override at deploy time |
| `--compatibility-date YYYY-MM-DD` | Override runtime date |
| `CLOUDFLARE_API_TOKEN` | Auth (CI) |
| `CLOUDFLARE_ACCOUNT_ID` | Skip account picker |
| `WRANGLER_LOG=debug` | Verbose CLI logging |
| `NO_D1_WARNING=true` | Silence D1 beta warnings in CI |

## Gotchas

- **`wrangler dev` is not `wrangler dev --remote`.** Local uses Miniflare (workerd) — fast but emulated. Remote uses real edge with real data — slower iteration but correct semantics.
- **`name` in `wrangler.toml` is the deployment identity.** Renaming creates a new Worker and orphans the old one (incl. its bindings/secrets).
- **Bindings differ per environment.** `[env.staging]` does **not** inherit `[[kv_namespaces]]` from the top level — you must repeat them. (Fixed in newer wrangler versions with `inheritance = "extend"`, but legacy behavior bites.)
- **Compatibility date matters.** Don't bump blindly — it can switch default runtime behaviors. Read the changelog.
- **`kv:namespace` → `kv namespace`** (colon dropped) was the v2 → v3 syntax change. v4 keeps the v3 form. The examples in this doc are v3/v4-compatible.
- **Account selection:** if you're in multiple CF accounts and `CLOUDFLARE_ACCOUNT_ID` is unset, wrangler picks the first one silently in non-TTY contexts. Always set it in CI.
- **Logs lose context fast.** `wrangler tail` shows only ~10 min of recent events; for postmortems use Logpush.

## CI example (GitHub Actions)

```yaml
- uses: actions/setup-node@v4
  with: { node-version: 20 }
- run: npm ci
- run: npx wrangler deploy --env production
  env:
    CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
    CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
```

Token scopes needed for a typical deploy: `Account.Workers Scripts:Edit`, `Account.Workers KV Storage:Edit`, `Account.Workers R2 Storage:Edit`, `Account.D1:Edit`, `Zone.Workers Routes:Edit` (for custom routes).
