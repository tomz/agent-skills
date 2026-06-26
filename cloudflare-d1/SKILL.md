---
name: cloudflare-d1
description: Cloudflare D1 — serverless SQLite at the edge. Schema/migrations, prepared statements, batching, read replication, and Worker integration.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare d1, d1, sqlite, worker
  version: 1.0.0
  updated: 2026-06-26
---
# Cloudflare D1

Serverless SQLite, replicated globally. Primary lives in one region; read replicas auto-spawn near users.

## Setup

```bash
wrangler d1 create my-db
# → copy database_id into wrangler.toml

wrangler d1 execute my-db --command="SELECT 1"
wrangler d1 execute my-db --file=./schema.sql
wrangler d1 execute my-db --remote --file=./schema.sql   # prod
```

```toml
[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "…"
```

## Migrations

```bash
wrangler d1 migrations create my-db add_users_table
# → creates migrations/0001_add_users_table.sql
wrangler d1 migrations apply my-db --local
wrangler d1 migrations apply my-db --remote
wrangler d1 migrations list my-db
```

Example migration:

```sql
-- migrations/0001_init.sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT NOT NULL UNIQUE,
  created_at INTEGER DEFAULT (unixepoch())
);
CREATE INDEX idx_users_email ON users(email);
```

## Query from Worker

```ts
// SELECT one
const user = await env.DB.prepare('SELECT * FROM users WHERE id = ?')
  .bind(id).first<User>();

// SELECT many
const { results } = await env.DB.prepare('SELECT * FROM users WHERE created_at > ?')
  .bind(since).all<User>();

// INSERT with returning
const inserted = await env.DB.prepare(
  'INSERT INTO users (email) VALUES (?) RETURNING id, created_at',
).bind(email).first();

// Batch (atomic, single round-trip)
await env.DB.batch([
  env.DB.prepare('INSERT INTO accounts (id) VALUES (?)').bind(aid),
  env.DB.prepare('INSERT INTO members (account_id, user_id) VALUES (?, ?)').bind(aid, uid),
]);
```

**Always use `.bind()` for parameters.** String interpolation = SQL injection.

## Transactions

D1 does **not** support interactive transactions across multiple requests. Use `batch()` for atomic multi-statement work — it runs in an implicit transaction.

## Read replication (Sessions API)

For read-heavy workloads, use the Sessions API to pin reads to a replica while preserving sequential consistency:

```ts
const session = env.DB.withSession('first-unconstrained');
const { results } = await session.prepare('SELECT * FROM posts').all();
// Subsequent reads on `session` won't go back in time.
ctx.waitUntil(/* persist bookmark in cookie for next request */);
```

## Limits (verified 2026-04)

- **Database size**: 500 MB Free / 10 GB Paid (hard cap — design for horizontal sharding).
- **Databases per account**: 10 Free / 50,000 Paid (raisable on Enterprise).
- **Storage per account**: 5 GB Free / 1 TB Paid.
- **Queries per Worker invocation**: 50 Free / 1,000 Paid.
- **Max row / BLOB / string**: 2 MB.
- **Max SQL statement**: 100 KB.
- **Max bound parameters**: 100 per query.
- **Max columns per table**: 100.
- **Max query duration**: 30 seconds.
- **Concurrent D1 connections per Worker invocation**: 6.
- Each DB is backed by a single Durable Object → **single-threaded**; throughput ≈ 1000 qps for sub-ms queries.
- No stored procedures; no triggers calling external code.

## Local dev

`wrangler dev` uses a local SQLite file at `.wrangler/state/v3/d1/`. Add to `.gitignore`. Use `--persist-to` to share state across runs.

## Backups

```bash
wrangler d1 export my-db --remote --output=./backup.sql
wrangler d1 export my-db --remote --no-data --output=./schema.sql
```

Time Travel: point-in-time recovery up to 30 days back — `wrangler d1 time-travel restore`.
