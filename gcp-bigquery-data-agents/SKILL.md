---
name: gcp-bigquery-data-agents
description: BigQuery Data Agents (Gemini-powered conversational analytics on BigQuery) — provisioning, grounding on datasets, governance, NL→SQL guardrails, embedding in apps, and evaluation patterns.
license: MIT
version: 1.0.0
updated: 2026-04-27
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---
# GCP BigQuery Data Agents Skill

Conversational analytics agents that run on top of BigQuery, grounded in your
datasets, governed by IAM/policy tags, and accessible via the Conversational
Analytics API, the BigQuery Studio UI ("Ask Gemini"), Looker, and embedded
chat surfaces.

A "Data Agent" is a deployable resource (`google.geminidataanalytics.v1.DataAgent`)
that bundles:
- one or more **data sources** (BigQuery datasets / tables / views, Looker models)
- **context** (business glossary, semantic descriptions, golden queries, SQL examples)
- **published context** (what the agent is allowed to expose to end users)
- a **system instruction** (persona, scope, refusal policy)
- **access controls** (IAM on the agent + row/column policies on the underlying data)

> Naming note: the public API surface lives under `geminidataanalytics.googleapis.com`
> and the gcloud noun is `gcloud alpha geminidataanalytics`. The product is marketed
> as **BigQuery Data Agents** / **Conversational Analytics**.

---

## Architecture

```
User (chat surface: Studio, Looker, custom app, Slack, Workspace)
  │
  ▼
Conversational Analytics API (geminidataanalytics.googleapis.com)
  │  ├── DataAgent (resource) ─────────► system instruction + context
  │  ├── Conversation (session)
  │  └── ChatRequest / ChatResponse (streaming)
  ▼
Gemini (NL→intent→SQL) ──────► BigQuery (executes SQL as the *end user* via
  │                                        end-user credentials propagation,
  │                                        OR as the agent SA — see "Identity")
  │
  ├── reads context resources:
  │     • table/column descriptions
  │     • policy tags & data classification
  │     • saved queries / golden examples
  │     • Dataplex business glossary terms
  │
  └── returns: answer text + executed SQL + result table + chart suggestion
```

Three execution modes the agent picks between:
1. **Data QA** — NL → BigQuery SQL → result table → narrated answer
2. **Python analysis** — agent generates Python (pandas/matplotlib) over result rows
3. **Forecasting / ML** — agent calls `ML.FORECAST`, `AI.GENERATE_TEXT`, vector search

---

## Provisioning a Data Agent (gcloud)

```bash
# 0. Enable APIs
gcloud services enable \
  geminidataanalytics.googleapis.com \
  bigquery.googleapis.com \
  dataplex.googleapis.com \
  --project=$PROJECT_ID

# 1. Create the agent resource
gcloud alpha geminidataanalytics data-agents create sales-analyst \
  --location=us-central1 \
  --display-name="Sales Analyst" \
  --description="Answers revenue, pipeline, and quota questions for the sales org." \
  --system-instruction-file=instructions/sales.md \
  --data-source-bigquery-dataset=projects/$PROJECT_ID/datasets/sales_curated \
  --data-source-bigquery-dataset=projects/$PROJECT_ID/datasets/crm_curated

# 2. Attach Looker model (optional, gives the agent semantic layer)
gcloud alpha geminidataanalytics data-agents update sales-analyst \
  --location=us-central1 \
  --add-data-source-looker-model=models/sales_looker_model

# 3. Grant chat access (separate from data access — see "Identity" below)
gcloud alpha geminidataanalytics data-agents add-iam-policy-binding sales-analyst \
  --location=us-central1 \
  --role=roles/geminidataanalytics.dataAgentUser \
  --member=group:sales-team@example.com

# 4. Smoke-test from CLI
gcloud alpha geminidataanalytics data-agents chat sales-analyst \
  --location=us-central1 \
  --message="What was Q1 2026 booked revenue by region?"
```

### Terraform

```hcl
resource "google_geminidataanalytics_data_agent" "sales" {
  provider     = google-beta
  location     = "us-central1"
  data_agent_id = "sales-analyst"
  display_name = "Sales Analyst"

  system_instruction = file("${path.module}/instructions/sales.md")

  data_sources {
    bigquery_dataset {
      dataset = "projects/${var.project}/datasets/sales_curated"
    }
    bigquery_dataset {
      dataset = "projects/${var.project}/datasets/crm_curated"
    }
  }

  context {
    # Inline golden examples — see "Context engineering" section
    golden_queries = file("${path.module}/context/golden_queries.yaml")
  }
}

resource "google_geminidataanalytics_data_agent_iam_member" "sales_users" {
  provider     = google-beta
  location     = google_geminidataanalytics_data_agent.sales.location
  data_agent_id = google_geminidataanalytics_data_agent.sales.data_agent_id
  role         = "roles/geminidataanalytics.dataAgentUser"
  member       = "group:sales-team@example.com"
}
```

---

## Identity & Access (the part that bites you)

Data Agents have **two identities** that must be reasoned about separately:

| Identity | What it does | How to set | Risk if wrong |
|---|---|---|---|
| **Caller identity** (end user) | Auth to the *agent* (chat) | IAM `roles/geminidataanalytics.dataAgentUser` on the agent | User gets to ask questions but agent can't answer |
| **Data identity** (who runs SQL) | Auth to *BigQuery* | Either propagate end-user creds, or use agent service account | If you use SA, you've just bypassed all your row-level security |

### Two execution modes

**(A) End-user credential propagation** (default for human chat — *strongly recommended*):
```bash
# The agent passes the user's identity to BigQuery.
# Row-level security, column policy tags, and data masking apply per-user.
gcloud alpha geminidataanalytics data-agents update sales-analyst \
  --location=us-central1 \
  --execution-mode=END_USER_CREDENTIALS
```
The end user must hold both:
- `roles/geminidataanalytics.dataAgentUser` on the agent
- `roles/bigquery.dataViewer` (or finer) on the underlying datasets

**(B) Service account execution** (for scheduled / system-to-system / Slack bots):
```bash
# Create dedicated SA (least-privilege)
gcloud iam service-accounts create sa-sales-agent \
  --display-name="Sales Data Agent"

# Grant SA access to ONLY the curated datasets
bq add-iam-policy-binding \
  --member=serviceAccount:sa-sales-agent@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/bigquery.dataViewer \
  $PROJECT_ID:sales_curated

# Bind SA to the agent
gcloud alpha geminidataanalytics data-agents update sales-analyst \
  --location=us-central1 \
  --execution-mode=SERVICE_ACCOUNT \
  --service-account=sa-sales-agent@$PROJECT_ID.iam.gserviceaccount.com
```

> ⚠ **In SERVICE_ACCOUNT mode the agent runs every query as the SA, regardless
> of who asked.** Row-level security, column-level policy tags, and data masking
> evaluate against the SA's identity. *Do not point an SA-mode agent at a
> dataset that contains data the chatting user shouldn't see.*

### Recommended IAM separation

```bash
# Data sources: locked to a specific group
roles/bigquery.dataViewer     → group:sales-data-readers@example.com (ON sales_curated)

# Agent chat: broader audience
roles/geminidataanalytics.dataAgentUser  → group:sales-team@example.com (ON the agent)

# Agent admin: small group
roles/geminidataanalytics.dataAgentAdmin → group:platform-data@example.com (ON the agent)

# Result: a salesperson can chat; if they don't also have BQ data access,
#         they get "you don't have access to this data" instead of seeing it.
```

---

## Context Engineering (where 80% of quality comes from)

The base model is generic. Quality lives in the **context** you ship with the agent.

### 1. Table / column descriptions (source of truth)

```sql
-- Set on the table itself — the agent reads these.
ALTER TABLE sales_curated.opportunities
SET OPTIONS (description = """
  One row per Salesforce opportunity, snapshotted nightly at 02:00 UTC.
  Use stage = 'Closed Won' for booked revenue.
  Use close_date for revenue recognition; use created_date for pipeline aging.
""");

ALTER TABLE sales_curated.opportunities
ALTER COLUMN amount_usd
SET OPTIONS (description = """
  Deal amount in USD, normalized from local currency at close_date FX rate.
  NULL for opportunities still in 'Prospecting'. Always use SUM(amount_usd)
  filtered by stage='Closed Won' for booked revenue.
""");
```

### 2. Golden queries (NL → SQL examples the agent learns from)

Provide **5–30 high-value** examples. More than 50 dilutes; fewer than 5 underspecifies.

```yaml
# context/golden_queries.yaml
golden_queries:
  - natural_language: "Q1 2026 booked revenue by region"
    sql: |
      SELECT region,
             SUM(amount_usd) AS booked_revenue_usd
      FROM   `{project}.sales_curated.opportunities`
      WHERE  stage      = 'Closed Won'
        AND  close_date BETWEEN '2026-01-01' AND '2026-03-31'
      GROUP  BY region
      ORDER  BY booked_revenue_usd DESC

  - natural_language: "Pipeline coverage for next quarter by rep"
    sql: |
      SELECT owner_email,
             SUM(amount_usd) FILTER (WHERE stage NOT IN ('Closed Won','Closed Lost')) AS open_pipeline,
             (SELECT quota_usd FROM `{project}.sales_curated.quotas` WHERE quarter='2026Q2'
              AND owner_email = o.owner_email) AS quota,
             SAFE_DIVIDE(
               SUM(amount_usd) FILTER (WHERE stage NOT IN ('Closed Won','Closed Lost')),
               (SELECT quota_usd FROM `{project}.sales_curated.quotas` WHERE quarter='2026Q2'
                AND owner_email = o.owner_email)
             ) AS coverage_ratio
      FROM   `{project}.sales_curated.opportunities` o
      WHERE  close_date BETWEEN '2026-04-01' AND '2026-06-30'
      GROUP  BY owner_email
```

### 3. Business glossary (Dataplex)

```bash
# Create glossary terms — the agent resolves NL synonyms via these.
gcloud dataplex glossaries create sales-glossary \
  --location=us-central1 \
  --display-name="Sales Glossary"

gcloud dataplex glossary-terms create booked-revenue \
  --glossary=sales-glossary \
  --location=us-central1 \
  --display-name="Booked Revenue" \
  --description="SUM(amount_usd) WHERE stage='Closed Won'. Recognized on close_date."

# Link term to a column so the agent knows the canonical mapping
gcloud dataplex entries update \
  projects/$PROJECT_ID/locations/us-central1/entryGroups/@bigquery/entries/.../column/amount_usd \
  --aspect-types=glossary:term=booked-revenue
```

### 4. System instruction (persona + guardrails)

```markdown
<!-- instructions/sales.md -->
You are the Sales Analyst for Acme Corp. You answer questions about pipeline,
bookings, quotas, and rep performance using the `sales_curated` and `crm_curated`
BigQuery datasets only.

## Scope
- Use ONLY the datasets attached to this agent. Never query other projects.
- For revenue questions, default to `stage='Closed Won'` unless the user explicitly
  says "pipeline" or "open" — then exclude `Closed Won` and `Closed Lost`.
- All amounts are USD-normalized in `amount_usd`. Do not use `amount_local`.
- Fiscal year starts February 1. Use the `fiscal_quarter` column, not calendar quarters,
  unless the user says "calendar quarter".

## Refusal policy
- If a question requires PII (email, phone, address), refuse and suggest the user
  request access to `sales_pii` instead.
- If a question requires data from a dataset not attached, say so explicitly:
  "I can only see sales_curated and crm_curated. For X, ask the Y agent."
- If the SQL would scan > 1 TB, ask the user to narrow the time window first.

## Output format
- Always show the executed SQL below the answer.
- For trends, suggest a line chart. For comparisons, suggest a bar chart.
- Round currency to the nearest dollar. Use thousands separators.
```

---

## Calling from Code (Conversational Analytics API)

### Python

```python
from google.cloud import geminidataanalytics_v1
from google.cloud.geminidataanalytics_v1 import (
    ChatRequest, Conversation, UserMessage, DatasourceReferences,
)

client = geminidataanalytics_v1.DataAgentServiceClient()
chat_client = geminidataanalytics_v1.DataChatServiceClient()

agent_path = client.data_agent_path(
    project="my-project",
    location="us-central1",
    data_agent="sales-analyst",
)

# Create or resume a conversation
conversation = chat_client.create_conversation(
    parent=f"projects/my-project/locations/us-central1",
    conversation=Conversation(agent=agent_path),
)

# Stream a chat turn
request = ChatRequest(
    parent=conversation.name,
    user_message=UserMessage(text="What was Q1 booked revenue by region?"),
)

for response in chat_client.chat(request=request):
    if response.text:
        print(response.text, end="", flush=True)
    if response.executed_sql:
        print(f"\n--- SQL ---\n{response.executed_sql}")
    if response.result_table:
        # response.result_table is a structured dict of column→values
        print(f"\n--- {response.result_table.row_count} rows ---")
    if response.chart_suggestion:
        print(f"\n--- Chart: {response.chart_suggestion.chart_type} ---")
```

### REST (curl)

```bash
ACCESS_TOKEN=$(gcloud auth print-access-token)

# Create conversation
CONV=$(curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://geminidataanalytics.googleapis.com/v1/projects/$PROJECT_ID/locations/us-central1/conversations" \
  -d '{
    "agent": "projects/'"$PROJECT_ID"'/locations/us-central1/dataAgents/sales-analyst"
  }' | jq -r .name)

# Send a message (streaming; --no-buffer for incremental)
curl --no-buffer -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://geminidataanalytics.googleapis.com/v1/$CONV:chat" \
  -d '{
    "user_message": {"text": "Q1 2026 booked revenue by region"}
  }'
```

---

## Embedding in Apps

### Embedded widget (BigQuery Studio iframe)

```html
<iframe
  src="https://console.cloud.google.com/bigquery/data-agents/sales-analyst/embed?project=my-project"
  width="100%" height="600"
  allow="clipboard-write"
  sandbox="allow-scripts allow-same-origin allow-forms">
</iframe>
```
End users authenticate via Google SSO; their BQ permissions apply (END_USER_CREDENTIALS mode).

### Slack bot pattern

```python
# slack_bot.py — minimal pattern; production needs error handling, retries, threading.
import os
from slack_bolt import App
from google.cloud import geminidataanalytics_v1

app = App(token=os.environ["SLACK_BOT_TOKEN"])
chat = geminidataanalytics_v1.DataChatServiceClient()
AGENT = "projects/my-project/locations/us-central1/dataAgents/sales-analyst"

# Map Slack user → cached conversation for multi-turn
_conv_for_user: dict[str, str] = {}

@app.event("app_mention")
def handle(event, say):
    user, text = event["user"], event["text"].split(">", 1)[-1].strip()

    # Resume or create conversation per Slack user
    if user not in _conv_for_user:
        c = chat.create_conversation(
            parent="projects/my-project/locations/us-central1",
            conversation={"agent": AGENT},
        )
        _conv_for_user[user] = c.name

    # Stream and post
    parts = []
    for resp in chat.chat({
        "parent": _conv_for_user[user],
        "user_message": {"text": text},
    }):
        if resp.text:
            parts.append(resp.text)

    say("\n".join(parts))

if __name__ == "__main__":
    app.start(port=int(os.environ.get("PORT", 3000)))
```

> Slack bots use **SERVICE_ACCOUNT** mode (the bot's SA queries BQ).
> The SA must only have access to data appropriate for the channel's audience.
> Don't connect a single SA-mode agent to mixed-audience channels.

---

## Governance & Safety

### 1. Cost guardrails

```bash
# Bytes-billed cap per query (the agent will refuse queries that exceed)
gcloud alpha geminidataanalytics data-agents update sales-analyst \
  --location=us-central1 \
  --max-bytes-billed=1099511627776   # 1 TiB

# Daily query quota per agent (custom quota via QuotaPolicy)
gcloud alpha quotas create-policy \
  --service=geminidataanalytics.googleapis.com \
  --metric=ChatRequestsPerDay \
  --value=1000 \
  --target=projects/$PROJECT_ID
```

Always set `max_bytes_billed` — without it, a chatty user with vague questions
("show me everything") can run $50 queries.

### 2. Column-level policy tags (apply to the data, not the agent)

```sql
-- Tag PII columns
ALTER TABLE sales_curated.opportunities
ALTER COLUMN owner_email
SET OPTIONS (
  policy_tags=['projects/my-project/locations/us-central1/taxonomies/123/policyTags/456']
);
```
In END_USER_CREDENTIALS mode, the agent will return masked values for users without
`Fine-Grained Reader` on that policy tag — without the agent knowing or caring.

### 3. Row-level security

```sql
-- Sales rep sees only their own opportunities
CREATE OR REPLACE ROW ACCESS POLICY rep_sees_own
ON sales_curated.opportunities
GRANT TO ('group:sales-reps@example.com')
FILTER USING (owner_email = SESSION_USER());
```

### 4. Audit logging

```bash
# Enable Data Access logs on the agent
gcloud projects get-iam-policy $PROJECT_ID --format=json > policy.json
# Edit auditConfigs to include geminidataanalytics.googleapis.com / DATA_READ
gcloud projects set-iam-policy $PROJECT_ID policy.json

# Query who asked what
bq query --use_legacy_sql=false """
  SELECT
    timestamp,
    protopayload_auditlog.authenticationInfo.principalEmail AS user,
    JSON_VALUE(protopayload_auditlog.metadataJson, '$.userMessage') AS question,
    JSON_VALUE(protopayload_auditlog.metadataJson, '$.executedSql')  AS sql_executed
  FROM \`$PROJECT_ID.audit_logs.cloudaudit_googleapis_com_data_access\`
  WHERE protopayload_auditlog.serviceName = 'geminidataanalytics.googleapis.com'
    AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
  ORDER BY timestamp DESC
"""
```

### 5. Refusal/jailbreak surface

The agent will execute *whatever SQL the model emits* against the bound datasets.
Defense-in-depth:

1. **Bind only curated/public-to-the-audience datasets.** Never bind raw, PII,
   or finance datasets to a broad-audience agent.
2. **Use END_USER_CREDENTIALS** so user permissions are the floor of access.
3. **Set `max_bytes_billed`** to bound runaway scans.
4. **Use authorized views / row-access policies** in the bound datasets so even
   crafted SQL can't reach beyond what the audience should see.
5. **Refuse rules in the system instruction** are advisory — they reduce but
   don't prevent off-scope queries. They are not a security boundary.

---

## Evaluation

A data agent is a system, not a query — evaluate it like one.

### Golden-set evaluation harness

```python
# eval/run_eval.py
import csv, time
from google.cloud import geminidataanalytics_v1, bigquery

chat = geminidataanalytics_v1.DataChatServiceClient()
bq   = bigquery.Client()
AGENT = "projects/.../dataAgents/sales-analyst"

def eval_one(question: str, expected_sql: str, expected_value: float | None) -> dict:
    conv = chat.create_conversation(
        parent="projects/my-project/locations/us-central1",
        conversation={"agent": AGENT},
    )
    t0 = time.time()
    answer_text, executed_sql, result = "", "", None
    for resp in chat.chat({
        "parent": conv.name,
        "user_message": {"text": question},
    }):
        if resp.text:           answer_text  += resp.text
        if resp.executed_sql:   executed_sql  = resp.executed_sql
        if resp.result_table:   result        = resp.result_table

    # Compare to golden value (if structured assertion)
    actual_value = result.rows[0].values[0] if (result and result.rows) else None
    correct = expected_value is None or _close(actual_value, expected_value)

    return {
        "question": question,
        "executed_sql": executed_sql,
        "answer": answer_text,
        "actual_value": actual_value,
        "expected_value": expected_value,
        "correct": correct,
        "latency_s": round(time.time() - t0, 2),
    }

def _close(a, b, tol=0.001):
    if a is None or b is None: return False
    return abs(float(a) - float(b)) / max(abs(float(b)), 1e-9) < tol

if __name__ == "__main__":
    rows = list(csv.DictReader(open("eval/golden.csv")))
    results = [eval_one(r["question"], r["expected_sql"], float(r["expected_value"]))
               for r in rows]
    correct = sum(1 for r in results if r["correct"])
    print(f"{correct}/{len(results)} correct ({100*correct/len(results):.1f}%)")
    csv.DictWriter(open("eval/results.csv","w"), fieldnames=results[0].keys()).writerows(results)
```

```csv
# eval/golden.csv
question,expected_sql,expected_value
"Q1 2026 booked revenue total","SELECT SUM(amount_usd) FROM ... WHERE stage='Closed Won' AND close_date BETWEEN '2026-01-01' AND '2026-03-31'",4823910.00
"Top rep by Q1 bookings","SELECT owner_email, SUM(amount_usd) ... ORDER BY 2 DESC LIMIT 1","jdoe@example.com"
```

### What to measure

| Dimension | How |
|---|---|
| **Correctness** | Golden Q&A set with expected numeric answers; ≥95% target |
| **SQL faithfulness** | Diff `executed_sql` against `expected_sql` AST (use `sqlglot`) |
| **Refusal rate** | Run an "out-of-scope" suite; agent should refuse, not fabricate |
| **Cost per turn** | Sum `total_bytes_billed` from `INFORMATION_SCHEMA.JOBS_BY_PROJECT` |
| **Latency** | p50 / p95 end-to-end (model + BQ execution) |
| **Hallucination** | Heuristic: % of answers where `executed_sql` is empty but `result_table` is referenced |
| **Drift** | Re-run the golden set weekly; alert on regressions |

---

## Common Patterns & Gotchas

### Pattern: "curated layer" is non-negotiable

Don't point a Data Agent at raw landing tables. Build a `*_curated` dataset with:
- denormalized, business-named columns (`booked_revenue_usd`, not `amt_norm`)
- table/column descriptions
- row-access policies enforcing audience boundaries
- partitioned by date, clustered by likely group-by columns
- a `vw_*` view layer that hides nullable/unstable columns

The agent is only as good as the schema it sees.

### Pattern: agent-per-domain, not one-mega-agent

```
sales-analyst       → sales_curated, crm_curated      (group: sales)
finance-analyst     → finance_curated, billing_curated (group: finance)
ops-analyst         → ops_curated, telemetry_curated   (group: ops)
exec-summary        → cross-domain *_summary views     (group: exec)
```
A single agent with 200 tables produces worse SQL than 5 agents with 40 each
(longer context, more ambiguity in column resolution, slower).

### Pattern: "Why did this answer change?" debugging

```sql
-- Find the chat turn from the audit log
SELECT
  timestamp,
  JSON_VALUE(protopayload_auditlog.metadataJson, '$.userMessage') AS question,
  JSON_VALUE(protopayload_auditlog.metadataJson, '$.executedSql') AS sql,
  JSON_VALUE(protopayload_auditlog.metadataJson, '$.conversationId') AS conv_id
FROM `audit_logs.cloudaudit_googleapis_com_data_access`
WHERE protopayload_auditlog.serviceName = 'geminidataanalytics.googleapis.com'
  AND JSON_VALUE(protopayload_auditlog.metadataJson, '$.userMessage') LIKE '%Q1 revenue%'
ORDER BY timestamp DESC LIMIT 5;

-- Re-run the SQL as the same user to reproduce
bq query --use_legacy_sql=false --impersonate_service_account=$ORIGINAL_USER "<SQL>"
```

### Gotcha: the agent caches schema, not data

When you `ALTER TABLE ... SET OPTIONS (description=...)` or add golden queries,
the agent picks them up on the next chat turn — **but** schema changes (new
columns, renamed columns) require an explicit refresh:

```bash
gcloud alpha geminidataanalytics data-agents refresh sales-analyst \
  --location=us-central1
```
Until you refresh, the agent will SELECT the old column names and fail.

### Gotcha: SERVICE_ACCOUNT mode + RLS = silent over-permission

Row-access policies use `SESSION_USER()`. In SERVICE_ACCOUNT mode, that's the
SA's email — not the chatting user's. RLS will let the SA see *everything* the
SA is granted, regardless of who asked. **Use END_USER_CREDENTIALS or build
audience filtering into the dataset structure (separate datasets per audience).**

### Gotcha: golden queries override semantic descriptions

If your golden query joins `opportunities` to `quotas` on `owner_email`, the
agent will prefer that join even when the user asks a question about a
*different* dimension. Keep golden queries narrow and high-quality (5–30), not
broad-and-many (100+). More is worse here.

### Gotcha: cost from "exploration" turns

Every "show me top 10 anything" the user types runs a real BQ query. Without
`max_bytes_billed`, a Slack channel of 50 users averaging 5 questions/day on a
multi-TB dataset will burn ~$300/day of slot capacity. Set the cap.

### Gotcha: regional pinning

The agent and its Looker model and its BQ datasets must be in compatible regions.
- Agent in `us-central1` + dataset in `EU` → fails at first query.
- Multi-region datasets (`US`, `EU`) work with agents in any single region within
  the multi-region.
- For data-residency-constrained workloads, pin the agent to the same region as
  the data and verify with `gcloud alpha geminidataanalytics data-agents describe`.

### Gotcha: "the agent gave a different answer to the same question"

Two reasons, both fixable:
1. **Non-deterministic SQL generation** — same NL → different SQL across turns.
   Mitigate by adding a golden query for that specific question, or tightening
   the system instruction.
2. **Underlying data changed** — the dataset is a moving target. Snapshot
   `*_curated` daily into `*_curated_snapshot_YYYYMMDD` and bind the agent to
   the snapshot dataset for reproducible analytics.

---

## Quick Reference

| Action | Command |
|---|---|
| Create | `gcloud alpha geminidataanalytics data-agents create NAME --location=LOC --system-instruction-file=F --data-source-bigquery-dataset=projects/P/datasets/D` |
| List | `gcloud alpha geminidataanalytics data-agents list --location=LOC` |
| Describe | `gcloud alpha geminidataanalytics data-agents describe NAME --location=LOC` |
| Update context | `gcloud alpha geminidataanalytics data-agents update NAME --location=LOC --system-instruction-file=F2` |
| Refresh schema | `gcloud alpha geminidataanalytics data-agents refresh NAME --location=LOC` |
| Grant chat | `gcloud alpha geminidataanalytics data-agents add-iam-policy-binding NAME --location=LOC --role=roles/geminidataanalytics.dataAgentUser --member=group:G` |
| Smoke-test | `gcloud alpha geminidataanalytics data-agents chat NAME --location=LOC --message="Q"` |
| Delete | `gcloud alpha geminidataanalytics data-agents delete NAME --location=LOC` |

| IAM role | Purpose |
|---|---|
| `roles/geminidataanalytics.dataAgentAdmin` | Create / update / delete agents |
| `roles/geminidataanalytics.dataAgentUser`  | Chat with an agent |
| `roles/geminidataanalytics.dataAgentViewer`| See agent metadata, no chat |
| `roles/bigquery.dataViewer`                | Read the underlying data (END_USER mode) |
| `roles/bigquery.jobUser`                   | Run query jobs (also needed in END_USER mode) |
| `roles/dataplex.viewer`                    | Read business glossary terms the agent uses |
