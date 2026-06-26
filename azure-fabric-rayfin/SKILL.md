---
name: azure-fabric-rayfin
description: Rayfin — open-source Backend-as-a-Service on Microsoft Fabric; TypeScript-decorator models provision DB, auth, APIs, storage, hosting.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: azure, fabric, microsoft fabric, rayfin, microsoft rayfin, baas, backend-as-a-service, rayfin-cli, rayfin-core, rayfinclient, create-rayfin, onelake, data api builder, decorator entity, fabric baas
  version: 1.0.0
  updated: 2026-06-14
---

# Microsoft Rayfin — Backend-as-a-Service on Fabric

**Rayfin** is Microsoft's open-source **Backend-as-a-Service (BaaS)** built on
**Microsoft Fabric** (preview, announced at Build 2026). You define your data
model as plain **TypeScript classes decorated with `@entity`** and Rayfin
provisions the full backend for you: a relational **database**, **auth**
(Fabric SSO + username/password), auto-generated **GraphQL + REST CRUD APIs**
(powered by **Data API Builder / DAB**), **blob storage**, and **static web
hosting** — all running on Fabric capacity and backed by OneLake.

The mental model: **your decorated classes are the source of truth.** Change a
class → run `rayfin up` → the schema, API, and row-level-security policies are
regenerated. It is the Fabric-native answer to Supabase / Firebase / AWS Amplify.

> **Sources** — grounded in the official Microsoft repo
> `github.com/microsoft/awesome-rayfin` (the `field-technician` template and the
> `rayfin`/`template-gallery` skills), the product page at
> `microsoft.com/.../microsoft-fabric/features/rayfin`, and the published npm
> packages `@microsoft/create-rayfin`, `@microsoft/rayfin-cli`,
> `@microsoft/rayfin-core`, `@microsoft/rayfin-client` (all **preview**, MIT).
> Rayfin is in **preview** — verify package versions and the exact CLI/decorator
> surface against the current docs before relying on it in production.

---

## Prereqs

- **Node.js 20+** and npm.
- A **Microsoft Fabric** tenant + workspace with capacity (F SKU or Power BI
  Premium with Fabric enabled), and permission to create Fabric items.
- The Rayfin CLI (installed transitively by the scaffolder, or globally):
  ```bash
  npm install -g @microsoft/rayfin-cli   # provides the `rayfin` binary
  ```
- For local dev, the data dialect defaults to **`mssql`**; **`postgresql`** is
  also supported. Auth allows `localhost` redirect URIs out of the box.

---

## Scaffold a new app

Use the create-package — it lets you pick a starter template (e.g. the
`field-technician` React/Vite app) and wires up `rayfin/`, the client SDK, and an
MCP server for agent tooling:

```bash
npm create @microsoft/rayfin@latest my-app
cd my-app
npm install
```

Then authenticate and provision:

```bash
rayfin login                 # device-code / browser sign-in to Fabric
rayfin init                  # link the project to a Fabric workspace (writes ids)
rayfin up                    # provision/update ALL services declared in rayfin.yml
```

`rayfin up` is **idempotent and convergent** — run it after every model or
config change. Sub-targets let you push one service at a time:

```bash
rayfin up db apply           # apply data-model (schema) changes to the database
rayfin up staticapp deploy   # build + publish the static frontend
```

---

## Project structure

```
my-app/
├── rayfin/
│   ├── rayfin.yml           # service declaration (auth/data/storage/hosting)
│   └── data/
│       ├── schema.ts        # registers entities into a typed schema
│       ├── Customer.ts      # one @entity class per file
│       ├── Job.ts
│       └── ...
├── src/                     # frontend (React + Vite in the sample template)
│   ├── services/rayfin/RayfinClientService.ts   # RayfinClient singleton
│   └── hooks/               # data + auth hooks
├── .mcp.json                # Rayfin MCP server (lets agents drive the backend)
└── package.json
```

---

## Data model — decorated entities

This is the heart of Rayfin. Each entity is a class from `@microsoft/rayfin-core`.
A clean, minimal entity:

```ts
// rayfin/data/Customer.ts
import { entity, role, text, uuid, email } from '@microsoft/rayfin-core';

@entity()
@role('authenticated', '*')          // any signed-in user gets all ('*') actions
export class Customer {
  @uuid() id!: string;               // server-generated primary key
  @text() name!: string;             // required text column
  @text() phone!: string;
  @email({ optional: true }) email?: string;
  @text({ optional: true }) address?: string;
}
```

A richer entity showing enums, dates, booleans-with-defaults, and relations:

```ts
// rayfin/data/Job.ts
import { entity, role, text, uuid, date, boolean, set, one } from '@microsoft/rayfin-core';
import { Customer } from './Customer.js';
import { Region } from './Region.js';
import { UserProfile } from './UserProfile.js';

export type JobStatus =
  | 'new' | 'scheduled' | 'in-progress' | 'complete' | 'abandoned';

@entity()
@role('authenticated', '*')
export class Job {
  @uuid() id!: string;
  @text() title!: string;
  @text({ optional: true }) description?: string;

  // @set(...) → a constrained enum column
  @set('new', 'scheduled', 'in-progress', 'complete', 'abandoned')
  status!: JobStatus;

  @date({ optional: true }) scheduledAt?: Date;
  @date() createdAt!: Date;
  @date() updatedAt!: Date;

  @boolean({ default: false }) isOnSite!: boolean;
  @boolean({ default: false }) needsHelp!: boolean;

  // Relations — @one(() => Target) is a foreign key to one row
  @one(() => Customer) customer!: Customer;
  @one(() => Region) region!: Region;
  @one(() => UserProfile, { optional: true }) technician?: UserProfile;
  @one(() => UserProfile) createdBy!: UserProfile;
}
```

A pure **join entity** (many-to-many wiring) is just two `@one` relations:

```ts
// rayfin/data/UserRegion.ts
import { entity, role, uuid, one } from '@microsoft/rayfin-core';
import { UserProfile } from './UserProfile.js';
import { Region } from './Region.js';

@entity()
@role('authenticated', '*')
export class UserRegion {
  @uuid() id!: string;
  @one(() => UserProfile) userProfile!: UserProfile;
  @one(() => Region) region!: Region;
}
```

### Decorator cheat-sheet

| Decorator | Purpose |
|---|---|
| `@entity()` | Marks the class as a managed table. |
| `@role(name, actions)` | RLS/authorization, e.g. `@role('authenticated', '*')` or a specific role name; `'*'` = all CRUD actions. |
| `@uuid()` | Server-generated UUID primary key. |
| `@text({ optional })` | String column; `optional: true` → nullable. |
| `@int()` / `@decimal()` | Numeric columns. |
| `@boolean({ default })` | Boolean with optional default. |
| `@date({ optional })` | Date/timestamp column. |
| `@email({ optional })` | Validated email string. |
| `@set(...values)` | Enum constrained to the listed string values. |
| `@one(() => T, { optional })` | Foreign-key relation to one row of `T`. |
| `@many(() => T)` | Inverse / collection side of a relation. |

> Use a lazy `() => Target` in relations so circular imports between entity
> files resolve correctly. Import sibling entities with the `.js` extension
> (ESM), as shown above.

### Register entities into a typed schema

`schema.ts` exports both a **type** (for end-to-end client typing) and a
**runtime array** (so the CLI knows which classes to provision):

```ts
// rayfin/data/schema.ts
import { Customer } from './Customer.js';
import { Job } from './Job.js';
import { Region } from './Region.js';
import { UserProfile } from './UserProfile.js';
import { UserRegion } from './UserRegion.js';

export type FieldTechSchema = {
  Customer: Customer;
  Job: Job;
  Region: Region;
  UserProfile: UserProfile;
  UserRegion: UserRegion;
};

export const schema = [Customer, Job, Region, UserProfile, UserRegion];
```

---

## Service config — `rayfin.yml`

Declares which backend services Rayfin provisions. This is the full surface from
the sample app:

```yaml
# rayfin/rayfin.yml
id: field-technician
name: field-technician
version: 1.0.0
services:
  auth:
    enabled: true
    fabric:                       # Entra/Fabric SSO
      enabled: true
    password:                     # username + password sign-in
      enabled: true
    allowedRedirectUris:
      - http://localhost:5173
      - http://localhost:5173/auth/callback
  data:
    enabled: true
    dialect: mssql                # or: postgresql
  storage:
    enabled: true                 # blob storage service
  staticHosting:
    enabled: true
    folder: dist                  # build output to publish
    buildCommand: npm run build
    indexDocument: index.html
```

Add a redirect URI here for every origin your app runs on (local + deployed),
or Fabric auth will reject the callback.

---

## Client SDK — `@microsoft/rayfin-client`

### Construct the client (once, as a singleton)

```ts
import { RayfinClient } from '@microsoft/rayfin-client';
import type { FieldTechSchema } from '../../../rayfin/data/schema';

const client = new RayfinClient<FieldTechSchema>({
  baseUrl: import.meta.env.VITE_RAYFIN_API_URL,          // e.g. http://localhost:5168/
  publishableKey: import.meta.env.VITE_RAYFIN_PUBLISHABLE_KEY,
  useProxy: false,
  headers: { 'Content-Type': 'application/json', Origin: window.location.origin },
});
```

Passing the `<FieldTechSchema>` generic makes `client.data.<Entity>` fully typed
from your decorated classes. The **publishable key** is required.

### Queries — fluent builder, terminated by `.execute()`

```ts
// SELECT specific columns, filter, order
const open = await client.data.Job
  .select(['id', 'title', 'status', 'createdAt'])
  .where({ status: { eq: 'new' } })       // operators: eq, (and more)
  .orderBy({ createdAt: 'desc' })
  .execute();                              // → Job[]

// Fetch one by id
const rows = await client.data.Job
  .select(['id', 'title', 'status'])
  .where({ id: { eq: jobId } })
  .execute();
const job = rows.length ? rows[0] : null;
```

### Mutations — `create` / `update` / `delete`

```ts
// CREATE — relations are set with id-only stubs: { id }
const job = await client.data.Job.create({
  title: 'Fix HVAC',
  status: 'new',
  createdAt: new Date(),
  updatedAt: new Date(),
  isOnSite: false,
  needsHelp: false,
  customer: { id: customerId },     // @one relation
  region:   { id: regionId },
  createdBy: { id: profileId },
});

// UPDATE — (whereKey, patch)
await client.data.Job.update({ id: job.id }, { status: 'complete', completedAt: new Date() });

// DELETE
await client.data.Equipment.delete({ id: equipmentId });
```

---

## Auth

Rayfin ships an auth service with both **Fabric SSO** and **username/password**.
Typical flows:

```ts
// Email + password
const user = await authService.signIn(email, password);
const result = await authService.signUp(email, password);
await authService.signOut();

// Fabric single sign-on (Entra) — opens the Fabric login handoff
const user = await authService.ensureSignedInWithFabric();

// Restore session on app start (embedded → current user)
const embedded = await authService.initEmbeddedAuth();
const current  = await authService.getCurrentUser();
```

Read the authenticated identity off the client session — useful for stamping
ownership columns server-side:

```ts
const session = client.auth.getSession();
const userId = session.user?.id;
if (!userId) throw new Error('User is not authenticated');
```

`@role('authenticated', '*')` on an entity means only signed-in users can touch
it; switch the role name (e.g. `'anonymous'`) or narrow the action list to
tighten the auto-generated row-level-security policy.

---

## Deploy

```bash
rayfin login
rayfin up                     # converge every service in rayfin.yml
rayfin up db apply            # push only schema changes
rayfin up staticapp deploy    # build (buildCommand) + publish the dist/ frontend
```

`rayfin up` writes Fabric identifiers (workspace/item ids) into your env so the
client and Fabric auth pick them up (e.g. `VITE_FABRIC_WORKSPACE_ID`,
`VITE_FABRIC_ITEM_ID`, `VITE_RAYFIN_PUBLISHABLE_KEY`).

---

## Agent integration (MCP)

Scaffolded apps include a `.mcp.json` registering the **Rayfin MCP server**, so
an AI agent (Claude Code, Copilot, jaaicode, etc.) can inspect the model and
drive `rayfin` commands directly. Load it with your agent's MCP support to let
the agent add entities and run `rayfin up` as part of a task.

---

## Common pitfalls

| Symptom | Fix |
|---|---|
| `VITE_RAYFIN_PUBLISHABLE_KEY ... is required` | Publishable key not set — run `rayfin up` (writes env) or export it before building. |
| Fabric login popup never returns | The callback origin isn't in `allowedRedirectUris` in `rayfin.yml`. Add every origin (local + deployed). |
| New entity not in the API | It isn't in `schema.ts`'s `schema[]` array, or you didn't run `rayfin up db apply`. |
| `User is not authenticated` on writes | Call `signIn` / `ensureSignedInWithFabric` first; entities with `@role('authenticated', …)` reject anonymous calls. |
| Circular import between entities | Use lazy relations `@one(() => Target)` and import siblings with the `.js` extension. |
| Relation won't set on create | Pass an id-only stub `{ id }` for `@one` fields, not the full object. |
| Schema change ignored | `rayfin up` is the apply step — editing a class alone changes nothing until you converge. |

---

## See also

- `github.com/microsoft/awesome-rayfin` — official templates, skills, samples.
- `microsoft.com/.../microsoft-fabric/features/rayfin` — product overview.
- npm: `@microsoft/create-rayfin`, `@microsoft/rayfin-cli`,
  `@microsoft/rayfin-core`, `@microsoft/rayfin-client` (all preview).
- Companion Fabric skills in this library: **`azure-fabric`** (platform),
  **`azure-fabric-data-agents`** (conversational analytics).
