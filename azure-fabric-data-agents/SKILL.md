---
name: azure-fabric-data-agents
description: Microsoft Fabric Data Agents — Copilot-grounded conversational AI over OneLake (lakehouses, warehouses, semantic models, KQL). Provisioning, grounding sources, AI Skills, identity, governance, evaluation, and embedding patterns.
license: MIT
version: 1.0.0
updated: 2026-04-27
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---
# Microsoft Fabric Data Agents Skill

Conversational analytics agents inside Microsoft Fabric, grounded in OneLake
items (lakehouses, warehouses, semantic models, KQL databases), governed by
workspace RBAC + OneLake ACLs + Power BI RLS, and accessible via the Fabric
chat UI, the Data Agent REST API, Copilot in Power BI, Teams, and embedded
chat surfaces.

A Fabric Data Agent (also surfaced as **AI Skills** in some Fabric UIs) is a
workspace item that bundles:
- one or more **grounding data sources** (lakehouse SQL endpoints, warehouses,
  KQL databases, semantic models)
- **example questions** (NL → expected output, the Fabric equivalent of "golden queries")
- **instructions** (persona, scope, refusal policy, formatting)
- **published context** (tables/columns/measures the agent is allowed to expose)
- **identity** (run-as user vs. fixed identity for embedded scenarios)

> Naming note: Microsoft has used **AI Skills**, **Data Agents**, and
> **Copilot for Fabric** somewhat interchangeably across the 2025–2026 product
> evolution. The workspace item type as of early 2026 is `DataAgent`
> (REST: `/v1/workspaces/{ws}/dataAgents/{id}`). The chat UI calls them
> "Data Agents" in BI/Lakehouse experiences.

---

## Architecture

```
User (chat surface: Fabric portal, Power BI Copilot, Teams, embedded app)
  │
  ▼
Fabric Data Agent item (workspace-scoped)
  │  ├── instructions (markdown)
  │  ├── example questions (NL → expected SQL/DAX/KQL)
  │  ├── data sources:
  │  │     • lakehouse SQL endpoints  (T-SQL → Delta on OneLake)
  │  │     • warehouses               (T-SQL native)
  │  │     • semantic models          (DAX → Direct Lake or Import)
  │  │     • KQL databases            (KQL → Eventhouse / RTI)
  │  └── publish state (Draft / Published)
  ▼
Copilot orchestrator (Azure OpenAI in tenant region)
  │  NL → routing → SQL/DAX/KQL plan → execution → narrated answer
  ▼
Underlying engines (warehouse SQL, lakehouse SQL endpoint,
                    semantic model VertiPaq/Direct Lake, Kusto)
  │
  └── enforces: workspace RBAC, OneLake item ACLs, T-SQL row-level security,
                Power BI RLS/OLS, KQL row/column policies
```

Three answer modes the agent picks between:
1. **SQL over lakehouse/warehouse** — NL → T-SQL → result table → narrated answer
2. **DAX over semantic model** — NL → DAX → measure values → narrated answer
3. **KQL over Eventhouse** — NL → KQL → time-series chart + narrated answer

---

## Provisioning a Data Agent

> Fabric exposes most agent operations through the workspace UI and the Fabric
> REST API. There is no first-class `az fabric` CLI for agents as of early
> 2026 — automate via REST + a service principal, or via Fabric Deployment
> Pipelines for Dev → Test → Prod promotion.

### Get a Fabric token

```bash
# As the signed-in user (works for interactive scripts)
FABRIC_TOKEN=$(az account get-access-token \
  --resource https://api.fabric.microsoft.com \
  --query accessToken -o tsv)

# As a service principal (CI/CD)
az login --service-principal \
  --username $SP_APP_ID --password $SP_SECRET --tenant $TENANT_ID
FABRIC_TOKEN=$(az account get-access-token \
  --resource https://api.fabric.microsoft.com \
  --query accessToken -o tsv)
```

The service principal must be:
- a member of the workspace with `Member` or `Admin` role (not `Viewer`/`Contributor`),
- in a security group enabled under tenant setting *"Service principals can use Fabric APIs"*,
- granted `Build` permission on each semantic model the agent grounds on.

### Create the agent

```bash
WS_ID=<workspace-guid>

curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Sales Analyst",
    "description": "Answers revenue, pipeline, and quota questions for the sales org.",
    "definition": {
      "instructions": "<see instructions/sales.md below>",
      "dataSources": [
        {
          "type": "LakehouseSqlEndpoint",
          "workspaceId": "'"$WS_ID"'",
          "itemId": "<lakehouse-guid>",
          "schemas": ["dbo"],
          "tables": ["sales_curated.orders","sales_curated.opportunities","sales_curated.quotas"]
        },
        {
          "type": "SemanticModel",
          "workspaceId": "'"$WS_ID"'",
          "itemId": "<semantic-model-guid>"
        }
      ]
    }
  }'
```

### Add example questions (the highest-leverage tuning lever)

```bash
AGENT_ID=<agent-guid>
curl -sX PATCH "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d @example_questions.json
```

```json
// example_questions.json
{
  "definition": {
    "exampleQuestions": [
      {
        "question": "Q1 2026 booked revenue by region",
        "answerKind": "SQL",
        "dataSourceRef": "<lakehouse-guid>",
        "sql": "SELECT region, SUM(amount_usd) AS booked_revenue_usd FROM sales_curated.opportunities WHERE stage='Closed Won' AND close_date BETWEEN '2026-01-01' AND '2026-03-31' GROUP BY region ORDER BY booked_revenue_usd DESC"
      },
      {
        "question": "Pipeline coverage for next quarter by rep",
        "answerKind": "SQL",
        "dataSourceRef": "<lakehouse-guid>",
        "sql": "SELECT o.owner_email, SUM(CASE WHEN o.stage NOT IN ('Closed Won','Closed Lost') THEN o.amount_usd END) AS open_pipeline, q.quota_usd, SUM(CASE WHEN o.stage NOT IN ('Closed Won','Closed Lost') THEN o.amount_usd END) / NULLIF(q.quota_usd,0) AS coverage_ratio FROM sales_curated.opportunities o LEFT JOIN sales_curated.quotas q ON q.owner_email=o.owner_email AND q.quarter='2026Q2' WHERE o.close_date BETWEEN '2026-04-01' AND '2026-06-30' GROUP BY o.owner_email, q.quota_usd"
      },
      {
        "question": "Total revenue last fiscal year",
        "answerKind": "DAX",
        "dataSourceRef": "<semantic-model-guid>",
        "dax": "EVALUATE { CALCULATE([Total Revenue], DATESYTD(LASTDATE('Date'[Date]), \"01-31\")) }"
      }
    ]
  }
}
```

Provide **5–30 high-value** examples. More than ~50 dilutes routing quality;
fewer than 5 underspecifies the domain.

### Publish

Agents must be published before non-creators can chat with them:

```bash
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID/publish" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

Publishing snapshots the current definition into a "published version." Edits
go to the draft; only published agents serve chat traffic to viewers.

---

## Identity & Access (the part that bites you)

Fabric Data Agents have **two identities** to reason about — the chat caller
and the data-execution principal.

| Identity | What it does | Where it's enforced | Risk if wrong |
|---|---|---|---|
| **Caller** (end user / SP) | Auth to the *agent* and gates chat | Workspace RBAC (`Viewer`+ at minimum) on the agent's workspace | Misconfigured users get a "you don't have access" message but can't see data |
| **Execution principal** | Runs SQL/DAX/KQL on the underlying items | OneLake ACLs, T-SQL RLS, Power BI RLS, KQL policies | Agent runs as fixed SP → bypasses per-user RLS — over-permission |

### Two execution modes

**(A) Run-as-user (default for human chat — *strongly recommended*)**

The agent forwards the chatting user's identity to each data source.
T-SQL `SESSION_USER`, Power BI RLS roles based on `USERPRINCIPALNAME()`, and
KQL `current_principal()` all evaluate per user. Each user sees only what they
would see if they wrote the query themselves.

For this to work, every chatting user needs:
- `Viewer` (or higher) on the agent's workspace
- Read permission on each grounded item (lakehouse SQL endpoint, warehouse,
  semantic model `Read+Build`, KQL database `Viewer`)

**(B) Fixed-identity (service principal) — for embedded apps, Teams bots, public dashboards**

The agent runs every query as a single service principal. RLS evaluates against
that SP. Useful when:
- The chatting user has no Fabric license / isn't in the tenant
- A public-facing app needs deterministic data scope per agent

```bash
# Bind a service principal as the execution identity
curl -sX PATCH "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "definition": {
      "executionIdentity": {
        "type": "ServicePrincipal",
        "applicationId": "<sp-app-guid>"
      }
    }
  }'
```

> ⚠ **In fixed-identity mode, T-SQL RLS, Power BI RLS, OLS, and KQL row policies
> evaluate against the SP — *not* the chatting user.** Every user sees the
> union of what the SP can see. Either:
> - bind the SP to a *purpose-built workspace* with only audience-appropriate items, OR
> - implement audience filtering in the *agent instructions* (defense in depth, not security), OR
> - filter at the data layer (separate per-audience lakehouses/datasets).

### Recommended workspace layout

```
ws-curated-data         ← raw + curated items, locked to data engineers
  └── lakehouse_sales_curated, warehouse_finance, ...

ws-sales-agent          ← agent + Power BI semantic model, audience: sales
  ├── DataAgent: sales-analyst
  ├── SemanticModel: Sales Star (Direct Lake → ws-curated-data via shortcut)
  └── Members: group:sales-team@example.com (Viewer)

ws-finance-agent        ← separate workspace for finance
  └── ...
```

The agent never lives in the same workspace as raw data. Audience-scoped
workspaces give you a clean "workspace = audience" mental model that survives
fixed-identity mode without leaking data across audiences.

---

## Grounding Source Engineering (where 80% of quality comes from)

The base model is generic; quality lives in **what you ground on** and **how
the underlying items are described.**

### 1. Build a curated layer first

Don't ground on bronze/raw lakehouse tables. Build a `*_curated` lakehouse with:
- denormalized, business-named columns (`booked_revenue_usd`, not `amt_norm`)
- table & column descriptions (the agent reads these via `INFORMATION_SCHEMA`)
- liquid clustering on common group-by columns
- row-access policies / RLS enforcing audience boundaries
- a `vw_*` view layer hiding nullable/unstable internals

Without this, the agent will pick the wrong table or the wrong column on
ambiguous questions ("revenue" → which of 4 amount columns?).

### 2. Annotate tables and columns with descriptions

The agent reads SQL `COMMENT`s and Power BI model descriptions when planning.

**Lakehouse SQL endpoint / Warehouse (T-SQL):**
```sql
-- Add extended properties on tables and columns (Warehouse)
EXEC sys.sp_addextendedproperty
    @name = N'MS_Description',
    @value = N'One row per Salesforce opportunity, snapshotted nightly at 02:00 UTC. Use stage=''Closed Won'' for booked revenue.',
    @level0type = N'SCHEMA', @level0name = 'dbo',
    @level1type = N'TABLE',  @level1name = 'opportunities';

EXEC sys.sp_addextendedproperty
    @name = N'MS_Description',
    @value = N'Deal amount in USD, normalized at close_date FX rate. NULL until past Prospecting. Use SUM(amount_usd) WHERE stage=''Closed Won'' for booked revenue.',
    @level0type = N'SCHEMA', @level0name = 'dbo',
    @level1type = N'TABLE',  @level1name = 'opportunities',
    @level2type = N'COLUMN', @level2name = 'amount_usd';
```

**Lakehouse Delta tables (Spark — propagates to SQL endpoint):**
```python
spark.sql("""
ALTER TABLE sales_curated.opportunities
SET TBLPROPERTIES (
  'comment' = 'One row per opportunity, snapshotted nightly. Use stage=Closed Won for booked revenue.'
)
""")
spark.sql("ALTER TABLE sales_curated.opportunities ALTER COLUMN amount_usd COMMENT 'Deal amount in USD, normalized at close_date FX. SUM where stage=Closed Won for booked revenue.'")
```

**Semantic model (Tabular Editor / TMSL / Power BI Desktop):**
- Set table description, column description, and **measure description** in
  the model. The agent prefers measures over raw columns when answering
  aggregations — well-described measures are gold.
- Mark sensitive columns as `IsHidden=true` so the agent never picks them up.

### 3. Define measures, don't make the agent rederive them

```dax
// In the Sales Star semantic model
Booked Revenue =
    CALCULATE(
        SUM('Opportunities'[amount_usd]),
        'Opportunities'[stage] = "Closed Won"
    )

Pipeline Coverage =
    DIVIDE(
        CALCULATE(
            SUM('Opportunities'[amount_usd]),
            NOT 'Opportunities'[stage] IN { "Closed Won", "Closed Lost" }
        ),
        SELECTEDVALUE('Quotas'[quota_usd])
    )
```

A user asking "what's our pipeline coverage for Q2?" will get a deterministic
answer when there's a `[Pipeline Coverage]` measure with a clear description.
Without it, the agent must reinvent the formula every time and may pick the
wrong filter.

### 4. Instructions (persona + guardrails)

Stored as markdown in the agent's `definition.instructions`:

```markdown
You are the Sales Analyst for Acme Corp. You answer questions about pipeline,
bookings, quotas, and rep performance using the `sales_curated` lakehouse and
the `Sales Star` semantic model only.

## Routing
- For aggregations on standard measures (revenue, pipeline, coverage), prefer
  the `Sales Star` semantic model and use existing measures.
- For row-level / opportunity-detail / freeform queries, use the `sales_curated`
  lakehouse SQL endpoint.
- Never join across data sources unless explicitly asked.

## Scope
- Use ONLY datasets attached to this agent. Never reference other workspaces.
- For "revenue" without qualifier, use `[Booked Revenue]` (stage=Closed Won).
- "Pipeline" excludes Closed Won and Closed Lost.
- All amounts are USD. Do not use `amount_local`.
- Fiscal year starts February 1. Use the `fiscal_quarter` column unless the
  user says "calendar quarter".

## Refusal policy
- If a question requires PII (email, phone, address) outside the audience's
  scope, refuse and direct them to the data owner.
- If a question requires data from a source not attached, say so explicitly:
  "I can only see sales_curated and Sales Star. For X, ask the Y agent."
- If a SQL plan would scan more than 1 TB, ask the user to narrow the time window.

## Output format
- Always show the executed SQL/DAX below the answer in a fenced code block.
- For trends, render a line chart. For comparisons, a bar chart.
- Round currency to the nearest dollar; use thousands separators.
- Include the data source name in the answer footer.
```

---

## Calling from Code

### Chat REST API

```bash
ACCESS_TOKEN=$FABRIC_TOKEN

# Create a conversation (session)
CONV_ID=$(curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID/conversations" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' | jq -r .id)

# Send a turn (streaming via Server-Sent Events)
curl --no-buffer -X POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID/conversations/$CONV_ID/messages" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"content":"Q1 2026 booked revenue by region","role":"user"}'
```

The SSE stream emits events of types `text` (incremental answer text),
`generatedQuery` (the SQL/DAX/KQL the agent emitted), `queryResult`
(structured rows), `chartSuggestion`, and `finished`.

### Python wrapper

```python
import json, os, requests, sseclient

BASE = "https://api.fabric.microsoft.com/v1"
WS, AGENT = os.environ["FABRIC_WS"], os.environ["FABRIC_AGENT"]
TOKEN = os.environ["FABRIC_TOKEN"]
HDR = {"Authorization": f"Bearer {TOKEN}"}

def new_conversation() -> str:
    r = requests.post(f"{BASE}/workspaces/{WS}/dataAgents/{AGENT}/conversations",
                      headers=HDR, json={})
    r.raise_for_status()
    return r.json()["id"]

def chat(conv_id: str, message: str):
    """Yields (event_type, payload) tuples as the agent streams its answer."""
    url = f"{BASE}/workspaces/{WS}/dataAgents/{AGENT}/conversations/{conv_id}/messages"
    r = requests.post(
        url,
        headers={**HDR, "Accept": "text/event-stream", "Content-Type": "application/json"},
        json={"role": "user", "content": message},
        stream=True,
    )
    r.raise_for_status()
    for ev in sseclient.SSEClient(r).events():
        yield ev.event, json.loads(ev.data) if ev.data else None

if __name__ == "__main__":
    conv = new_conversation()
    answer, sql = [], None
    for kind, payload in chat(conv, "Q1 2026 booked revenue by region"):
        if kind == "text":            answer.append(payload["delta"])
        elif kind == "generatedQuery": sql = payload["query"]
        elif kind == "queryResult":    rows = payload["rows"]
        elif kind == "finished":       break
    print("".join(answer))
    print(f"\n--- generated query ---\n{sql}")
```

### semantic-link / `sempy` (in a Fabric notebook)

```python
import sempy.fabric as fabric

# Discover agents in the current workspace
agents = fabric.list_items(type="DataAgent")
print(agents[["Display Name", "Id"]])

# Programmatic chat (sempy convenience wrapper, available in recent runtimes)
from sempy.fabric import DataAgentClient

client = DataAgentClient(workspace=fabric.get_workspace_id(), agent_id=AGENT_ID)
response = client.chat("Top 5 reps by Q1 bookings")
print(response.text)
print(response.generated_query)   # the SQL/DAX/KQL it ran
print(response.dataframe.head())  # results as Pandas
```

---

## Embedding in Apps

### Embed in a Power BI report

1. Add a Copilot or Q&A visual to the report.
2. Pin the agent in workspace settings → *Copilot data agents*.
3. The visual will route the user's question to the agent (run-as-user mode).

### Embed in a Teams app / custom web app

```html
<!-- Fabric provides an embeddable chat widget; URL pattern as of early 2026 -->
<iframe
  src="https://app.fabric.microsoft.com/groups/{WS_ID}/dataAgents/{AGENT_ID}/embed"
  width="100%" height="600"
  allow="clipboard-write"
  sandbox="allow-scripts allow-same-origin allow-forms allow-popups">
</iframe>
```

For non-tenant users, use **Fabric Embedded** (capacity-backed) and acquire an
embed token via the embed-token API; bind the agent to a service-principal
execution identity.

### Teams bot pattern

```python
# Minimal Teams bot pattern using the Bot Framework SDK.
# Production: add error handling, conversation state, retries, idempotency.
from botbuilder.core import ActivityHandler, TurnContext
import requests, os, json

BASE = "https://api.fabric.microsoft.com/v1"
WS, AGENT = os.environ["FABRIC_WS"], os.environ["FABRIC_AGENT"]

# Cache (Teams user id → Fabric conversation id) for multi-turn
_conv_for_user: dict[str, str] = {}

def _fabric_token() -> str:
    # Use MSAL with the bot's SP credentials to get a Fabric token
    ...

def _new_conversation(token: str) -> str:
    r = requests.post(f"{BASE}/workspaces/{WS}/dataAgents/{AGENT}/conversations",
                      headers={"Authorization": f"Bearer {token}"}, json={})
    r.raise_for_status()
    return r.json()["id"]

def _ask(token: str, conv_id: str, q: str) -> tuple[str, str | None]:
    r = requests.post(
        f"{BASE}/workspaces/{WS}/dataAgents/{AGENT}/conversations/{conv_id}/messages",
        headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
        json={"role": "user", "content": q},
    )
    r.raise_for_status()
    body = r.json()
    return body.get("content", ""), body.get("generatedQuery")

class FabricAgentBot(ActivityHandler):
    async def on_message_activity(self, turn_context: TurnContext):
        token = _fabric_token()
        user_id = turn_context.activity.from_property.id
        if user_id not in _conv_for_user:
            _conv_for_user[user_id] = _new_conversation(token)

        text, sql = _ask(token, _conv_for_user[user_id], turn_context.activity.text)
        reply = text + (f"\n\n```sql\n{sql}\n```" if sql else "")
        await turn_context.send_activity(reply)
```

> A Teams bot uses **fixed-identity** (its own SP). The SP must be granted
> only audience-appropriate access. Don't connect a single bot agent to
> mixed-audience channels.

---

## Governance & Safety

### 1. Capacity & cost

Data Agent chat consumes **Fabric Capacity Units (CUs)**:
- Copilot orchestration tokens (charged per agent message)
- Underlying query execution (warehouse / lakehouse / Direct Lake / KQL CU)

Controls:

```bash
# Pause a capacity overnight to stop chat traffic burning CU
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/capacities/$CAPACITY_ID/suspend" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

- Use the **Fabric Capacity Metrics app** (free from AppSource) to see per-item
  CU consumption — agents show as `DataAgent` items in the report.
- Set a workspace-level Copilot budget alert via the admin portal.
- Cap warehouse query execution time via `RESOURCE GOVERNOR` (warehouse) or
  Spark session timeouts.

### 2. T-SQL row-level security (lakehouse SQL endpoint / warehouse)

```sql
-- Run in the warehouse / lakehouse SQL endpoint
CREATE FUNCTION dbo.fn_rep_predicate(@email AS NVARCHAR(256))
    RETURNS TABLE WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS allow
    WHERE @email = USER_NAME() OR IS_ROLEMEMBER('SalesLeadership') = 1;

CREATE SECURITY POLICY dbo.RepRowPolicy
    ADD FILTER PREDICATE dbo.fn_rep_predicate(owner_email) ON dbo.opportunities
    WITH (STATE = ON);
```

In **run-as-user** mode the agent's queries inherit the policy. In
**fixed-identity** mode the SP sees everything the SP is permitted to see —
RLS bypasses are silent and total.

### 3. Power BI RLS / OLS

```dax
// In the semantic model — Manage roles → Sales Rep
[owner_email] = USERPRINCIPALNAME()
```

Ground the agent on the semantic model and the per-user role applies in
run-as-user mode.

### 4. KQL row-level / column-level policies

```kql
.create-or-alter table Telemetry policy row_level_security
'@"AllowedDevices | where DeviceOwner == current_principal()"'

.create-or-alter table Telemetry column policy
[
  { "ColumnName": "DeviceOwner", "MaskingPolicy": "Hash" }
]
```

### 5. Audit logging

Fabric writes Copilot/Data Agent activity to the **Microsoft Purview / unified
audit log** under workload `Microsoft.Fabric` and operation
`CopilotInteraction` (operation names evolve — check the current Fabric
audit-log schema docs).

```powershell
# Pull last 24 h of agent chats from the unified audit log
Connect-ExchangeOnline -ShowBanner:$false
$end   = Get-Date
$start = $end.AddDays(-1)
Search-UnifiedAuditLog `
  -StartDate $start -EndDate $end `
  -RecordType CopilotInteraction `
  -ResultSize 5000 |
  Where-Object { $_.AuditData -match '"AgentId"\s*:\s*"'$AGENT_ID'"' } |
  Select-Object CreationDate, UserIds, AuditData |
  Export-Csv -NoTypeInformation agent_audit.csv
```

For real-time auditing at scale, route the Microsoft 365 Defender / Purview
audit stream into an Eventhouse, then query via KQL:

```kql
CopilotInteractions
| where Workload == "Microsoft.Fabric"
| where AgentId == "<agent-guid>"
| where Timestamp > ago(1d)
| project Timestamp, UserId = User.UserPrincipalName, Question, GeneratedQuery, ResultRowCount
| order by Timestamp desc
```

### 6. Refusal / jailbreak surface

The agent will execute *whatever SQL/DAX/KQL the model emits* against the
grounded items. Defense in depth:

1. **Ground only on curated, audience-appropriate items.** Never connect a
   raw bronze lakehouse or a finance warehouse to a broad-audience agent.
2. **Use run-as-user.** Each user's permissions are the floor of access.
3. **Set caps**: warehouse query timeout, capacity CU budget, Copilot per-day
   message quota.
4. **Use authorized views / RLS** in the grounded items so even adversarial
   prompts can't reach beyond the audience scope.
5. **Refusal rules in instructions are advisory** — they reduce off-scope
   answers but are not a security boundary.

---

## Evaluation

A Data Agent is a system, not a query — evaluate it as one.

### Golden-set evaluation harness

```python
# eval/run_eval.py
import csv, time
from chat_client import new_conversation, chat   # the wrapper from "Calling from Code"

def eval_one(question: str, expected_value: float | None) -> dict:
    conv = new_conversation()
    t0 = time.time()
    text, sql, rows = "", None, None
    for kind, payload in chat(conv, question):
        if kind == "text":             text += payload["delta"]
        elif kind == "generatedQuery": sql = payload["query"]
        elif kind == "queryResult":    rows = payload["rows"]
        elif kind == "finished":       break
    actual = float(rows[0][0]) if rows else None
    correct = expected_value is None or _close(actual, expected_value)
    return {
        "question": question, "answer": text, "generated_query": sql,
        "actual": actual, "expected": expected_value,
        "correct": correct, "latency_s": round(time.time() - t0, 2),
    }

def _close(a, b, tol=0.001):
    if a is None or b is None: return False
    return abs(float(a) - float(b)) / max(abs(float(b)), 1e-9) < tol

if __name__ == "__main__":
    rows = list(csv.DictReader(open("eval/golden.csv")))
    results = [eval_one(r["question"], float(r["expected"])) for r in rows]
    correct = sum(1 for r in results if r["correct"])
    print(f"{correct}/{len(results)} correct ({100*correct/len(results):.1f}%)")
    csv.DictWriter(open("eval/results.csv","w"), fieldnames=results[0].keys()).writerows(results)
```

```csv
# eval/golden.csv
question,expected
"Q1 2026 booked revenue total",4823910.00
"Top rep by Q1 bookings",1245000.00
"Pipeline coverage for Q2 2026 by region (sum)",2.4
```

### What to measure

| Dimension | How |
|---|---|
| **Correctness** | Golden Q&A with expected numeric answers; ≥95% target |
| **Routing accuracy** | % of questions routed to the *right* data source (lakehouse vs semantic model vs KQL). Diff `generatedQuery` kind against expected. |
| **Query faithfulness** | Diff `generatedQuery` AST against an expected SQL/DAX baseline (`sqlglot` for SQL; for DAX, normalize whitespace and compare token streams) |
| **Refusal rate** | Run an "out-of-scope" suite; agent should refuse, not fabricate or pick a tangentially-related table |
| **Cost per turn** | CU consumed per message — pull from the Capacity Metrics app, attribute by agent item ID |
| **Latency** | p50 / p95 end-to-end (Copilot + query execution) |
| **Hallucination** | Heuristic: % of answers where `generatedQuery` is null but the response cites figures |
| **Drift** | Re-run the golden set weekly; alert on regressions tied to model updates or schema changes |

---

## Common Patterns & Gotchas

### Pattern: agent-per-domain, not one mega-agent

```
sales-analyst       → sales_curated lakehouse, Sales Star semantic model
finance-analyst     → finance_warehouse, Finance Star semantic model
ops-analyst         → ops_curated lakehouse, telemetry KQL eventhouse
exec-summary        → cross-domain *_summary views in an exec semantic model
```

A single agent grounded on 200 tables produces worse SQL than 5 agents grounded
on 40 each (longer context, more ambiguous column resolution, slower planning).

### Pattern: prefer semantic model over lakehouse SQL when both are grounded

For business aggregations, the semantic model wins:
- Measures encode business logic deterministically (`[Booked Revenue]` = one definition)
- DAX result is typically smaller than equivalent SQL row sets → cheaper, faster
- Direct Lake means no data movement
Direct the agent in instructions: *"Prefer the semantic model for any question
that maps to an existing measure. Use the lakehouse only for row-level detail."*

### Pattern: snapshot for reproducible analytics

If users complain *"the agent gave me a different number yesterday,"* it's
usually because the underlying data changed. Snapshot curated tables daily:

```python
# In a scheduled notebook: snapshot sales_curated.opportunities → opportunities_YYYYMMDD
from datetime import date
spark.sql(f"""
  CREATE OR REPLACE TABLE sales_snapshot.opportunities_{date.today():%Y%m%d}
  SHALLOW CLONE sales_curated.opportunities
""")
```

Ground a "reproducible" agent on the snapshot dataset, and a "live" agent on
the moving curated tables. Users pick the agent that matches the question.

### Pattern: "why did this answer change?" debugging

```kql
// Eventhouse with the audit stream
CopilotInteractions
| where AgentId == "<agent-guid>"
| where Question has "Q1 revenue"
| order by Timestamp desc
| take 20
| project Timestamp, UserId = User.UserPrincipalName, Question, GeneratedQuery, ResultRowCount
```

Then re-run the `GeneratedQuery` as the same user (T-SQL: connect to the
SQL endpoint with that user; DAX: use *Run as different user* in Power BI
Desktop).

### Gotcha: agent caches schema, not data

`ALTER TABLE ... SET TBLPROPERTIES (comment=...)`, new measures, and added
example questions take effect on the next conversation. **But schema changes
(new columns, renamed columns, new tables) require explicit refresh of the
data source binding** — some surfaces auto-refresh on the next chat after a
publish, others require re-publishing the agent. Until refresh, the agent
emits queries against the stale schema and fails. Always re-publish after
schema changes.

### Gotcha: fixed-identity + RLS = silent over-permission

T-SQL `USER_NAME()`, DAX `USERPRINCIPALNAME()`, and KQL `current_principal()`
return the *SP* in fixed-identity mode. RLS predicates that filter on the user
become permissive (the SP is granted everything). **Use run-as-user, OR
audience-scope the bound items, OR both.**

### Gotcha: semantic-model `Read` without `Build` silently drops the agent's access

Workspace `Viewer` gives read on reports but not `Build` on the semantic model.
Without `Build`, the agent can't issue DAX queries on behalf of the user — the
agent will silently fall back to lakehouse SQL (often picking the wrong table)
or refuse. Grant `Build` on every semantic model the agent is grounded on, to
every audience group that should be able to chat.

### Gotcha: example-question overfit

If an example question joins `opportunities` to `quotas` on `owner_email`, the
agent will prefer that join even when the user asks a question about a
*different* dimension. Keep examples narrow and high-quality (5–30), not broad
and many (100+). Diverse, deduplicated examples beat exhaustive ones.

### Gotcha: cost from "exploration" turns

Every "show me top 10 anything" runs a real warehouse / Direct Lake / KQL query.
Without query-time caps, a Teams channel of 50 users averaging 5 questions per
day on a multi-TB lakehouse can saturate an F64 capacity. Set warehouse query
timeouts; monitor CU via the Capacity Metrics app; route low-priority agents
to a separate, smaller capacity.

### Gotcha: regional pinning & Copilot availability

The Copilot orchestrator runs in a Microsoft-managed region tied to the
capacity's home region. Some regions don't yet have Copilot for Fabric in GA.
Verify under tenant settings → *Copilot and Azure OpenAI Service*. If the
capacity is in an unsupported region, agents fail at chat time with an opaque
"Copilot is not available" error. Move the capacity, or enable
*"data sent to Azure OpenAI may be processed outside your geography"*.

### Gotcha: Direct Lake fallback hides behind agent answers

When the semantic model's grounded table exceeds the SKU's Direct Lake
guardrails (rows / size / columns — see the azure-fabric skill table), Power
BI silently falls back to DirectQuery. The agent's DAX still works, but
latency jumps from sub-second to several seconds and CU consumption rises.
Monitor with Power BI Performance Analyzer; if fallback is consistent, either
upsize the capacity or aggregate the table further.

### Gotcha: agents do not honor sensitivity labels for output gating

A Confidential-labeled lakehouse can be grounded on by a Public-audience agent.
Sensitivity labels propagate to *exports* (Excel, CSV) but do not block the
agent from reading and narrating the data in chat. Treat the agent's grounded
items as the security boundary, not the labels on those items.

---

## Quick Reference

| Action | REST endpoint (POST/PATCH/DELETE under `https://api.fabric.microsoft.com/v1`) |
|---|---|
| Create agent | `POST /workspaces/{ws}/dataAgents` |
| List agents | `GET  /workspaces/{ws}/dataAgents` |
| Get agent | `GET  /workspaces/{ws}/dataAgents/{id}` |
| Update agent (instructions, sources, examples) | `PATCH /workspaces/{ws}/dataAgents/{id}` |
| Publish | `POST /workspaces/{ws}/dataAgents/{id}/publish` |
| Delete | `DELETE /workspaces/{ws}/dataAgents/{id}` |
| New conversation | `POST /workspaces/{ws}/dataAgents/{id}/conversations` |
| Send message (SSE) | `POST /workspaces/{ws}/dataAgents/{id}/conversations/{conv}/messages` |
| List conversations (admin/audit) | `GET  /workspaces/{ws}/dataAgents/{id}/conversations` |

| Workspace role on the agent | What chat users can do |
|---|---|
| `Viewer`        | Chat with the published agent |
| `Contributor`   | Chat + view audit history |
| `Member`        | Edit draft, add examples, publish |
| `Admin`         | All of the above + delete + identity binding |

| Permission needed on grounded items (run-as-user mode) | |
|---|---|
| Lakehouse SQL endpoint | `Read` on the lakehouse + T-SQL `SELECT` on grounded tables |
| Warehouse              | `Read` on the warehouse + T-SQL `SELECT` on grounded tables |
| Semantic model         | `Read` + `Build` (Build is required for DAX execution) |
| KQL database           | `Viewer` on the database + table-level access |

| Token resource URIs | |
|---|---|
| Fabric REST API   | `https://api.fabric.microsoft.com` |
| Power BI XMLA     | `https://analysis.windows.net/powerbi/api` |
| OneLake (ABFS)    | `https://storage.azure.com` |
| KQL ingestion     | `https://<cluster>.<region>.kusto.fabric.microsoft.com` |
