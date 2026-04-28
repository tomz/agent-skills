---
name: azure-fabric-data-agents
description: Microsoft Fabric Data Agents (GA) — conversational AI grounded in OneLake (lakehouses, warehouses, Power BI semantic models, KQL databases, mirrored databases, ontologies, Microsoft Graph). Provisioning via the fabric-data-agent-sdk, identity model, Purview governance, sharing, evaluation, and operational gotchas.
license: MIT
version: 2.0.0
updated: 2026-04-28
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---

# Microsoft Fabric Data Agents

Conversational analytics agents inside Microsoft Fabric — a **generally available**
Fabric workspace item type (`DataAgent`) that grounds an Azure-OpenAI-Assistants-API
agent on data living in OneLake. Accessible from the Fabric portal chat surface,
from Microsoft 365 Copilot / Teams (via sharing), and programmatically via the
**`fabric-data-agent-sdk`** Python package inside Fabric notebooks.

> **Sources & scope** — this skill is grounded in the Microsoft Learn docs at
> `learn.microsoft.com/en-us/fabric/data-science/concept-data-agent`,
> `…/how-to-create-data-agent`, `…/data-agent-sharing`, `…/fabric-data-agent-sdk`,
> `…/data-agent-tenant-settings`, the REST API reference at
> `learn.microsoft.com/en-us/rest/api/fabric/dataagent/items/create-data-agent`,
> and the official sample notebooks at
> `github.com/microsoft/fabric-samples/tree/main/docs-samples/data-science/data-agent-sdk`.
> Where docs and samples disagree (most notably on capacity SKU), this skill
> calls it out explicitly.

For a 20-minute first-agent path, see **`azure-fabric-data-agents-quickstart`**.

---

## Architecture

```
User (chat surface: Fabric portal, M365 Copilot, Teams, embedded app, notebook)
  │
  ▼
Fabric Data Agent (workspace item, type=DataAgent)
  │  ├── instructions          (agent-level persona & rules)
  │  ├── data sources (max 5)  ─┐
  │  │                          │  per-source:
  │  │                          ├── selected tables (whitelist)
  │  │                          ├── datasource instructions
  │  │                          └── few-shot examples (NL → SQL/DAX/KQL)
  │  └── publish state          (Draft → Published)
  ▼
Microsoft-managed Azure OpenAI Assistants API
  │  parses Q → routes to a tool → generates query → validates → executes
  ▼
Per-source query engines  (NL2SQL / NL2DAX / NL2KQL / Microsoft Graph / Ontology)
  │
  └── Auth: caller's Entra ID identity. Read-only. Honors:
       • workspace RBAC + item permissions
       • Power BI model RLS / OLS
       • OneLake item ACLs (incl. shortcuts & cross-tenant shares)
       • Purview DLP (Warehouse, GA), access-restriction policies
         (Warehouse / KQL DB / Fabric SQL DB, preview), sensitivity labels
```

### Three things to understand first

1. **The agent is GA**, but the **Python SDK and several governance features
   are still preview**. Don't conflate them.
2. **It is read-only.** "The Fabric data agent strictly enforces read-only
   access, maintaining read-only data connections to all data sources."
3. **There is no service-account "execution mode."** Every query runs as the
   Entra ID identity of the caller (user OR service principal calling the
   API). Workspace + data permissions of *that identity* gate everything.

---

## Prerequisites

| Item | Required |
|---|---|
| **Capacity** | A paid **F2 or higher** F SKU, or **P1+ Power BI Premium** with [Microsoft Fabric enabled](https://learn.microsoft.com/en-us/fabric/admin/fabric-switch). The official SDK consumption sample notebook recommends **F64+** for end-to-end reliability. |
| **Tenant settings** | Enable **AI skill / Copilot for Fabric** and the **cross-geo processing & cross-geo storing for AI** toggles per the [tenant settings doc](https://learn.microsoft.com/en-us/fabric/data-science/data-agent-tenant-settings). |
| **Workspace role** | **Contributor** or higher (per the REST `Create DataAgent` API: "the caller must have a contributor workspace role"). Not Member, not Admin specifically — Contributor is enough. |
| **Data sources** | At least one of: lakehouse, warehouse, Power BI semantic model, KQL database, mirrored database, ontology. **Read access** required. |
| **For Power BI semantic models** | Only **Read** on the model is required. **Build is not required.** Workspace membership where the model lives is **not required**. (Write is only needed to *modify* the model or use Prep for AI.) |

---

## Provisioning

There are three supported ways to create and configure a Data Agent:

| Method | When to use |
|---|---|
| **Fabric portal UI** | First agent, exploration, business users. `+ New item` → "Fabric data agent" → name → add data sources from the OneLake catalog → check tables → ask. |
| **`fabric-data-agent-sdk`** (preview) | Code-first, repeatable, runs **inside a Fabric notebook only** (not local). The recommended programmatic path. |
| **Fabric REST API** (`POST /v1/workspaces/{ws}/dataAgents`) | CI/CD, automation outside notebooks. The body uses a `definition.parts[]` array of base64-encoded JSON files (`Files/Config/data_agent.json`, `…/draft/lakehouse-<name>/datasource.json`, `…/fewshots.json`, `…/stage_config.json`, `publish_info.json`). Most users should not hand-roll this — let the SDK or portal generate it. |

### SDK path (recommended for code-first)

```python
%pip install fabric-data-agent-sdk     # only inside a Fabric notebook

from fabric.dataagent.client import (
    FabricDataAgentManagement,
    create_data_agent,
    delete_data_agent,
)

# Create new — errors with ConflictName if it already exists
data_agent = create_data_agent("sales-analyst")

# Or attach to existing
data_agent = FabricDataAgentManagement("sales-analyst")

# Inspect current configuration
data_agent.get_configuration()

# Add agent-level instructions
data_agent.update_configuration(instructions="""
You are the Sales Analyst. You answer questions about pipeline, bookings,
quotas, and rep performance using the lakehouse and warehouse attached.

Rules:
- "Booked revenue" = SUM(amount_usd) WHERE stage = 'Closed Won'.
- "Pipeline" = stage NOT IN ('Closed Won','Closed Lost').
- Fiscal year starts February 1 — use fiscal_quarter, not calendar quarter,
  unless the user explicitly says "calendar".
- Always show the executed SQL/DAX/KQL beneath the answer.
- Out of scope: root cause, prediction, why-questions. Refuse and stop.
""")

# Add data sources (up to 5 total per agent, in any combination)
data_agent.add_datasource("sales_lakehouse",  type="lakehouse")
data_agent.add_datasource("sales_warehouse",  type="warehouse")
data_agent.add_datasource("Sales Model",      type="semanticmodel")
data_agent.add_datasource("EventsKQL",        type="kqldatabase")

# Per-source: select tables (NOTHING is selected by default)
ds = data_agent.get_datasources()[0]
ds.pretty_print()                       # tables marked "*" are visible to the agent
ds.select("dbo", "opportunities")       # whitelist
ds.select("dbo", "quotas")
ds.unselect("dbo", "internal_audit")    # explicit removal

# Per-source instructions (query-generation hints)
ds.update_configuration(instructions="""
- For revenue questions, use opportunities.amount_usd (already FX-normalized).
- For quota math, JOIN opportunities to quotas on owner_email AND fiscal_quarter.
""")

# Per-source few-shot examples (NL → query, dialect of the source)
ds.add_fewshots({
    "Q1 2026 booked revenue by region":
        "SELECT region, SUM(amount_usd) AS booked_revenue_usd "
        "FROM dbo.opportunities "
        "WHERE stage = 'Closed Won' "
        "  AND close_date BETWEEN '2026-01-01' AND '2026-03-31' "
        "GROUP BY region ORDER BY booked_revenue_usd DESC",
    "Pipeline coverage for next quarter by rep":
        "SELECT o.owner_email, "
        "       SUM(CASE WHEN o.stage NOT IN ('Closed Won','Closed Lost') THEN o.amount_usd END) AS open_pipe, "
        "       q.quota_usd "
        "FROM dbo.opportunities o "
        "LEFT JOIN dbo.quotas q ON q.owner_email = o.owner_email AND q.quarter = '2026Q2' "
        "WHERE o.close_date BETWEEN '2026-04-01' AND '2026-06-30' "
        "GROUP BY o.owner_email, q.quota_usd",
})
ds.get_fewshots()

# Inspect / delete a few-shot
ds.remove_fewshot("<fewshot-id-from-get_fewshots>")

# Publish — until you publish, only the creator can chat
data_agent.publish()

# Delete (cleanup)
delete_data_agent("sales-analyst")
```

### Portal path (recommended for first agent)

`+ New item` → search **"Fabric data agent"** → name it → the OneLake catalog
opens → **Add** a data source (one at a time; up to 5) → in the left **Explorer**
pane, **check the tables** that the agent should see → start chatting in the
right pane → use **Edit instructions** and **Add example** in the UI.

> Microsoft's tip from the docs: "Use descriptive names for both tables and
> columns. A table named `SalesData` is more meaningful than `TableA`, and
> column names like `ActiveCustomer` or `IsCustomerActive` are clearer than
> `C1` or `ActCu`. Descriptive names help the AI generate more accurate and
> reliable queries."

### REST path (CI/CD)

```bash
FABRIC_TOKEN=$(az account get-access-token \
  --resource https://api.fabric.microsoft.com --query accessToken -o tsv)

# Minimal create — no definition; you configure via SDK or portal afterwards.
curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "displayName": "sales-analyst", "description": "Sales agent" }'
```

The `definition` body, when supplied, is a `parts[]` array of base64-encoded
JSON files at paths like `Files/Config/data_agent.json`,
`Files/Config/draft/lakehouse-<name>/datasource.json`,
`Files/Config/draft/lakehouse-<name>/fewshots.json`,
`Files/Config/draft/stage_config.json`,
`Files/Config/published/...`, and `Files/Config/publish_info.json`. The SDK is
the practical way to produce these payloads — see the
[REST reference](https://learn.microsoft.com/en-us/rest/api/fabric/dataagent/items/create-data-agent)
for the exact shape.

Required Entra scope for the API: **`Item.ReadWrite.All`**. Service principals
and managed identities are supported.

---

## Identity & Access

The agent always runs queries as the **Entra ID identity of the caller** —
there is no "service account execution mode" toggle. The chatting principal's
permissions are the floor.

| Surface | Identity at query time |
|---|---|
| Fabric portal chat | Signed-in user |
| M365 Copilot / Teams plugin | Signed-in user (via SSO) |
| `FabricOpenAI` in a notebook | The notebook's running identity (interactive user, or workspace identity if scheduled) |
| Direct REST automation | Whatever Entra principal owns the bearer token (user or SP) |

### What permissions a chatter actually needs

| To do this | They need |
|---|---|
| See the agent in the workspace | Workspace **Viewer** + (typically also via sharing) |
| Be able to chat | Per agent **share** — see Sharing section |
| Get answers from a **lakehouse / warehouse** source | **Read** on the lakehouse/warehouse SQL endpoint |
| Get answers from a **Power BI semantic model** | **Read** on the model. **Build not required.** Workspace membership where the model lives **not required.** |
| Get answers from a **KQL database** | Standard KQL viewer permissions |

### Cross-tenant data

If your workspace contains tables shared in via **OneLake external data
sharing**, the agent queries them through the OneLake shortcut created at
share-acceptance — no extra credentials, runs under the consumer's Entra ID,
and consumer-tenant governance applies.

---

## Sharing

Per [`data-agent-sharing`](https://learn.microsoft.com/en-us/fabric/data-science/data-agent-sharing),
agents are shared as a first-class operation distinct from workspace RBAC.
Share dialog supports per-user/group permission tiers. Recipients still need
**read on the underlying data sources** (or, for semantic models, just Read on
the model) for queries to actually return data — sharing the agent does not
elevate data access.

> Operational rule: **the agent's share is who can chat; the data sources'
> permissions are who gets answers.** If a user can chat but gets "no access,"
> they're missing data permissions, not agent permissions.

---

## What to put in instructions vs few-shots vs descriptions

The agent has three context surfaces. Use each for what it's good at:

| Surface | What goes here | What does NOT |
|---|---|---|
| **Agent instructions** (`update_configuration(instructions=...)`) | Persona, refusal rules, output format, business definitions ("booked revenue ="), fiscal calendar quirks, scope boundary | Per-table SQL hints (those go on the datasource) |
| **Datasource instructions** (`ds.update_configuration(instructions=...)`) | Which table to use for X, JOIN keys, deprecated columns to avoid, dialect notes | Whole worked SQL examples (those are few-shots) |
| **Few-shots** (`ds.add_fewshots({nl: sql})`) | 5–30 high-value NL→query examples; each example raises the routing prior toward that pattern | More than ~50 (dilutes routing); examples that aren't validated to actually run |
| **Table/column descriptions** (`ALTER TABLE … COMMENT '…'` on the underlying Delta table; warehouse uses T-SQL `sp_addextendedproperty`) | What each column means, units, valid values, how to filter, lineage | Long prose — keep < 200 chars per column |

Always validate few-shots with the **Few-Shot Examples Validator** notebook
(`Fabric-DataAgent-Few-Shot-Examples-Validator-sample.ipynb` in the
`fabric-samples` repo) — it actually runs each example's SQL and checks the
result is non-empty and well-formed.

---

## Consumption (chat from code)

Use the `FabricOpenAI` client bundled with `fabric-data-agent-sdk`. It wraps
the Azure OpenAI Assistants API with the agent as the assistant target.

```python
from fabric.dataagent.client import FabricOpenAI

fabric_client = FabricOpenAI(artifact_name="sales-analyst")

assistant = fabric_client.beta.assistants.create(model="gpt-4o")
thread    = fabric_client.beta.threads.create()

fabric_client.beta.threads.messages.create(
    thread_id=thread.id, role="user",
    content="Q1 2026 booked revenue by region",
)

run = fabric_client.beta.threads.runs.create_and_poll(
    thread_id=thread.id, assistant_id=assistant.id,
)
assert run.status == "completed", run.status

for m in fabric_client.beta.threads.messages.list(thread_id=thread.id).data:
    if m.role == "assistant":
        for part in m.content:
            if part.type == "text":
                print(part.text.value)
        break
```

Multi-turn: reuse `thread.id` across calls. The agent maintains conversational
context within a thread.

> The SDK is **only supported inside Fabric notebooks**. Microsoft explicitly
> says it "isn't supported for local execution." For non-notebook automation
> use a notebook scheduled in a pipeline, a Spark Job Definition, or call the
> REST API directly.

---

## Governance & Safety

### Microsoft Purview integration

Per the concept doc, the following Purview capabilities apply to Data Agents:

| Capability | Status | What it does |
|---|---|---|
| **DLP policies in Fabric Data Warehouse** | GA | Detects/restricts agent access to sensitive warehouse data |
| **Access restriction policies** for Warehouse, KQL DB, Fabric SQL DB | Preview | Prevents agent from accessing/returning sensitive assets |
| **Risk discovery & auditing** | Preview | Prompts/responses subject to Purview audit |
| **DSPM Data Risk Assessments** | Preview | Surface sensitive-data risks in agent's data sources |
| **Insider Risk Management** | Preview | Detect risky AI-usage patterns (e.g., abnormal query volumes) |
| **Audit, eDiscovery, retention** | Per Fabric workload | Agent interactions are logged in supported workloads |

### Outbound access protection

Outbound connections from agents are subject to the tenant's network and
access rules in the Fabric admin portal. Admins can control which external
endpoints agents may reach.

### Cross-region rule (hard fail)

> "The Fabric data agent can't execute queries when the data source's
> workspace capacity is in a different region than the data agent's workspace
> capacity."

Example: agent in *France Central*, lakehouse in *North Europe* → query fails.
Fix: move the agent's workspace to a capacity in the same region as the data,
or move/re-shortcut the data.

### Read-only enforcement

The agent maintains read-only connections to all sources. Even a crafted
prompt cannot get it to issue a write. This is an architectural property, not
a guardrail rule — but verify in your audit logs anyway.

### Out-of-scope question patterns

Per the docs, these are not what the agent does (it's NL→query, not analysis):

- "Why is Q2 productivity lower?"
- "What is the root cause of the sales spike?"
- "Forecast next quarter's revenue."
- Anything requiring causal inference, ML, or external context.

Put a refusal rule in the agent instructions. Better: include a few-shot whose
"answer" is a refusal, so the model learns the pattern.

---

## Evaluation

The SDK ships an evaluation utility plus a sample notebook
(`Fabric-DataAgent-Evaluation-sample.ipynb`). The pattern is the standard
golden-set scoring:

1. Maintain a CSV/Delta table of `(question, expected_query_or_answer)` pairs.
2. For each row, send the question to the agent, capture the executed query
   and the result.
3. Score correctness (numeric tolerance for aggregates, exact match for
   labels), latency, refusal-when-out-of-scope, and SQL similarity (e.g., via
   `sqlglot` AST comparison).
4. Persist results to a Delta table; chart drift over time.

Re-run weekly (or after every agent edit) and alert on regressions. See the
[Microsoft blog post on programmatic evaluation](https://blog.fabric.microsoft.com/en-US/blog/evaluate-your-fabric-data-agents-programmatically-with-the-python-sdk/)
for the canonical pattern.

---

## Common Patterns

### Pattern: agent-per-domain, not one mega-agent

A single agent capped at 5 sources is also capped in instruction-context
budget. For broad analytics, build per-domain agents:

```
sales-analyst       → sales_lakehouse, crm_warehouse, Sales Model
finance-analyst     → finance_warehouse, billing_lakehouse, Finance Model
ops-analyst         → ops_lakehouse, telemetry_kql
exec-summary        → cross-domain *_summary semantic models
```

### Pattern: curated layer is non-negotiable

Don't point an agent at raw landing tables. Build curated `*_curated`
schemas with descriptive names, column comments, partitioning/liquid
clustering, and views that hide unstable internals. The agent is only as good
as the schema it reads.

### Pattern: validate few-shots in CI

Wire the Few-Shot Examples Validator into a scheduled notebook. Any few-shot
whose query returns 0 rows or errors should fail the build — that means
schema drift broke the example, and the agent is now learning from broken
SQL.

### Pattern: snapshot for reproducible answers

If users complain "the agent gave a different number yesterday," it's almost
always because the data changed. For monthly-board-deck reproducibility,
snapshot `*_curated` to `*_curated_snapshot_YYYYMMDD` and bind the agent to
the snapshot.

---

## Gotchas

1. **F2 vs F64 SKU contradiction.** The official prerequisites include says
   F2 is enough; the official SDK consumption sample says F64. Reality: F2
   gets you a functioning agent for the portal chat path; the OpenAI-Assistants
   notebook consumption path is more reliable on F64+. Plan accordingly.

2. **The SDK is preview and notebook-only.** Don't try to `pip install` it on
   your laptop; it expects the Fabric notebook runtime.

3. **The REST `definition` body is base64-encoded JSON parts**, not the
   friendly `{dataSources: [...]}` shape that some unofficial blog posts
   imply. The SDK or portal generates these for you.

4. **Default table selection is empty.** After `add_datasource()`, NO tables
   are visible to the agent until you `ds.select("schema", "table")`. A
   freshly-added datasource that returns "I don't see any tables" usually
   just means you forgot to select.

5. **Lakehouse SQL endpoint is read-only.** Add column descriptions via
   Spark `ALTER TABLE … ALTER COLUMN … COMMENT '…'` on the underlying Delta
   table, not via T-SQL `sp_addextendedproperty` against the SQL endpoint
   (which will fail). Warehouses (writable T-SQL) do accept
   `sp_addextendedproperty`.

6. **Cross-region = hard fail**, not slow. Same-region the agent and all its
   sources, or use shortcuts to bring the data into the agent's region.

7. **Publish or no chat for others.** Drafts work only for the creator.
   Recipients of a `share` see "no access" until you `data_agent.publish()`.

8. **Sharing the agent ≠ granting data access.** Recipient still needs Read
   on each underlying source (except Power BI semantic models, where Read on
   the model is sufficient and workspace membership is not required).

9. **5 source maximum per agent.** Hard cap. Hit this and you need to either
   consolidate sources via shortcuts/views, or split into multiple agents.

10. **Few-shots can override descriptions.** A few-shot that joins
    `opportunities` to `quotas` on `owner_email` will bias the agent toward
    that join even when irrelevant. Keep few-shots narrow and high-quality.

11. **No service-account "mode."** Every query runs as the caller. If a Teams
    bot uses an SP, that SP needs Read on the data sources — and *every user
    of the bot effectively gets the SP's data access*, regardless of their
    own permissions. For mixed-audience bots, build audience filtering at the
    data layer (separate datasets/lakehouses per audience), not "in the bot."

12. **Tenant settings can silently disable agents.** "AI skill", "Copilot for
    Fabric", and "cross-geo processing/storing for AI" all need to be on for
    your security group. Agents in disabled tenants fail with vague auth
    errors.

13. **Renamed columns require re-publish.** Schema changes propagate to the
    agent only on the next publish. Until then it generates SQL using the old
    names.

14. **The agent does not do analytics.** It does NL→SQL/DAX/KQL→answer. Root
    cause / forecasting / "why" questions are explicitly out of scope per the
    Microsoft docs. Refuse them in the agent instructions.

---

## Quick Reference

| Action | SDK call |
|---|---|
| Create | `data_agent = create_data_agent("name")` |
| Attach to existing | `data_agent = FabricDataAgentManagement("name")` |
| Set agent instructions | `data_agent.update_configuration(instructions="…")` |
| Inspect config | `data_agent.get_configuration()` |
| Add data source | `data_agent.add_datasource("item-name", type="lakehouse"\|"warehouse"\|"semanticmodel"\|"kqldatabase")` |
| List sources | `data_agent.get_datasources()` |
| Select tables | `ds.select("schema", "table")` / `ds.unselect(...)` |
| Datasource instructions | `ds.update_configuration(instructions="…")` |
| Add few-shots | `ds.add_fewshots({"NL": "SQL", …})` |
| Inspect/delete few-shots | `ds.get_fewshots()` / `ds.remove_fewshot(id)` |
| Publish | `data_agent.publish()` |
| Delete | `delete_data_agent("name")` |
| Chat from code | `FabricOpenAI(artifact_name="name").beta.threads.…` |

| REST endpoint | Purpose |
|---|---|
| `POST /v1/workspaces/{ws}/dataAgents` | Create (Contributor+, scope `Item.ReadWrite.All`) |
| `GET  /v1/workspaces/{ws}/dataAgents` | List |
| `GET  /v1/workspaces/{ws}/dataAgents/{id}` | Read |
| `PATCH /v1/workspaces/{ws}/dataAgents/{id}` | Update |
| `DELETE /v1/workspaces/{ws}/dataAgents/{id}` | Delete |

| Source type strings (SDK) | Engine |
|---|---|
| `"lakehouse"` | NL2SQL over lakehouse SQL endpoint |
| `"warehouse"` | NL2SQL over Fabric Warehouse |
| `"semanticmodel"` | NL2DAX over Power BI semantic model |
| `"kqldatabase"` | NL2KQL over KQL DB / Eventhouse |

(Per the docs, ontologies, mirrored databases, and Microsoft Graph are also
supported source types — check current SDK release notes for the exact `type=`
string for each, which has been moving.)

---

## See also

- **`azure-fabric-data-agents-quickstart`** — 20-minute first-agent path
- Concept: `learn.microsoft.com/en-us/fabric/data-science/concept-data-agent`
- How-to: `learn.microsoft.com/en-us/fabric/data-science/how-to-create-data-agent`
- SDK: `learn.microsoft.com/en-us/fabric/data-science/fabric-data-agent-sdk`
- Sharing: `learn.microsoft.com/en-us/fabric/data-science/data-agent-sharing`
- Tenant settings: `learn.microsoft.com/en-us/fabric/data-science/data-agent-tenant-settings`
- REST API: `learn.microsoft.com/en-us/rest/api/fabric/dataagent`
- Sample notebooks: `github.com/microsoft/fabric-samples/tree/main/docs-samples/data-science/data-agent-sdk`
- Evaluation blog: `blog.fabric.microsoft.com/en-US/blog/evaluate-your-fabric-data-agents-programmatically-with-the-python-sdk/`
