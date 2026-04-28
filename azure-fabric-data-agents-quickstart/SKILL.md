---
name: azure-fabric-data-agents-quickstart
description: Fast-path quickstart for standing up a Microsoft Fabric Data Agent in ~20 minutes — workspace setup, one lakehouse table grounded, instructions, 3 example questions, publish, smoke test via REST. For the full reference (governance, eval, embedding, multi-source, gotchas) see the azure-fabric-data-agents skill.
license: MIT
version: 1.0.0
updated: 2026-04-28
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---

# Microsoft Fabric Data Agents — Quickstart

A 20-minute path from zero to a working **Fabric Data Agent** you can chat with
from the Fabric portal or via REST. Optimized for proof-of-concept / first-agent.
For production governance, identity modes, evaluation, embedding, and the full
gotcha list, see the companion skill **`azure-fabric-data-agents`**.

> Naming: workspace item type is `DataAgent`. Some Fabric UIs surface it as
> "AI Skills." REST base: `https://api.fabric.microsoft.com/v1/workspaces/{ws}/dataAgents`.

---

## Prereqs (2 min)

You need:
- A Microsoft Fabric tenant with **Copilot for Fabric enabled** by your tenant admin
  (Admin portal → Tenant settings → "Copilot and Azure OpenAI" → enabled for your group)
- A Fabric capacity with **F64 or higher** (Copilot/Data Agents require F64+)
- A workspace assigned to that capacity, where you have **Member** or **Admin** role
  (`Viewer`/`Contributor` is not enough to create agents)
- `azure-cli` for getting a token (`az login` already done)

```bash
# Set these once
export WS_ID="<your-workspace-guid>"          # Fabric portal → Workspace settings → ID
export AGENT_NAME="quickstart-analyst"
export LAKEHOUSE_NAME="quickstart_lh"

# Get a Fabric API token (valid ~1 hr)
export FABRIC_TOKEN=$(az account get-access-token \
  --resource https://api.fabric.microsoft.com \
  --query accessToken -o tsv)
```

---

## Step 1 — Create a lakehouse with a demo table (4 min)

If you already have a lakehouse with a described table, skip to Step 2.

```bash
# Create lakehouse
LH_RESP=$(curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/lakehouses" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"displayName\": \"$LAKEHOUSE_NAME\"}")

export LH_ID=$(echo "$LH_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Lakehouse: $LH_ID"

# Get the lakehouse's SQL endpoint connection string for later
SQL_EP=$(curl -s "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/lakehouses/$LH_ID" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['properties']['sqlEndpointProperties']['connectionString'])")
echo "SQL endpoint: $SQL_EP"
```

Now load a demo `orders` table. Easiest path is via the **Fabric portal**:
1. Open the new lakehouse → **Get data** → **New shortcut** is *not* what you want.
   Instead: **Upload** → upload a small Parquet/CSV, OR use a notebook:

```python
# Run this in a Fabric notebook attached to the lakehouse
from pyspark.sql import functions as F, types as T
import random, datetime

regions = ['NA','EMEA','APAC','LATAM']
statuses = ['Closed Won','Closed Lost','Open']
rows = []
today = datetime.date.today()
for i in range(5000):
    rows.append((
        f"o-{i:05d}",
        today - datetime.timedelta(days=random.randint(0, 180)),
        random.choice(regions),
        random.choice(statuses),
        round(random.random()*10000, 2),
        f"rep{random.randint(0,9)}@example.com",
    ))

schema = T.StructType([
    T.StructField("order_id",    T.StringType()),
    T.StructField("order_date",  T.DateType()),
    T.StructField("region",      T.StringType()),
    T.StructField("status",      T.StringType()),
    T.StructField("amount_usd",  T.DoubleType()),
    T.StructField("owner_email", T.StringType()),
])
df = spark.createDataFrame(rows, schema=schema)
df.write.mode("overwrite").format("delta").saveAsTable("orders")
```

### Add table & column descriptions (high-ROI step)

In a Fabric notebook (or SQL editor against the lakehouse SQL endpoint):

```sql
-- Run against the lakehouse SQL endpoint (T-SQL)
EXEC sp_addextendedproperty 'MS_Description',
  'One row per order. Use status=''Closed Won'' for booked revenue. Use order_date for revenue recognition.',
  'SCHEMA','dbo','TABLE','orders';

EXEC sp_addextendedproperty 'MS_Description',
  'Order amount in USD. SUM(amount_usd) WHERE status=''Closed Won'' = booked revenue.',
  'SCHEMA','dbo','TABLE','orders','COLUMN','amount_usd';

EXEC sp_addextendedproperty 'MS_Description',
  'One of: Closed Won (booked), Closed Lost, Open (still in pipeline).',
  'SCHEMA','dbo','TABLE','orders','COLUMN','status';

EXEC sp_addextendedproperty 'MS_Description',
  'Sales region: NA, EMEA, APAC, LATAM.',
  'SCHEMA','dbo','TABLE','orders','COLUMN','region';

EXEC sp_addextendedproperty 'MS_Description',
  'Email of the rep who owns the order.',
  'SCHEMA','dbo','TABLE','orders','COLUMN','owner_email';
```

> The agent reads these descriptions when generating SQL. Skipping this step is
> the single biggest cause of a vague-feeling agent.

---

## Step 2 — Write a minimal system instruction (1 min)

```bash
cat > ./instructions.md <<'EOF'
You are the Demo Analyst. Answer questions about orders using ONLY the
`dbo.orders` table in the quickstart_lh lakehouse.

## Scope
- For "revenue" or "bookings", default to `status='Closed Won'`.
- For "pipeline" or "open", use `status='Open'`.
- All amounts are USD in `amount_usd`.

## Refusals
- If a question requires a table outside `dbo.orders`, say so and stop.
- If asked for PII beyond `owner_email`, refuse.

## Output
- Always show the executed SQL beneath the answer.
- Round currency to whole dollars with thousands separators.
EOF
```

---

## Step 3 — Create the agent (3 min)

```bash
# Escape the instruction file content as JSON
INSTRUCTIONS_JSON=$(python3 -c "import json,sys; print(json.dumps(open('instructions.md').read()))")

CREATE_RESP=$(curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"displayName\": \"$AGENT_NAME\",
    \"description\": \"Quickstart agent over dbo.orders\",
    \"definition\": {
      \"instructions\": $INSTRUCTIONS_JSON,
      \"dataSources\": [
        {
          \"type\": \"LakehouseSqlEndpoint\",
          \"workspaceId\": \"$WS_ID\",
          \"itemId\": \"$LH_ID\",
          \"schemas\": [\"dbo\"],
          \"tables\": [\"dbo.orders\"]
        }
      ]
    }
  }")

export AGENT_ID=$(echo "$CREATE_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Agent: $AGENT_ID"
```

If you get `400 InvalidArgument` on `LakehouseSqlEndpoint`, your tenant may use
the older `Lakehouse` source type — substitute and retry. Some preview tenants
also require `"sqlEndpointId"` instead of (or in addition to) `"itemId"`.

---

## Step 4 — Add 3 example questions (2 min)

Three is the sweet spot for a quickstart. Don't bulk-load 50 — quality beats
quantity for routing.

```bash
cat > ./example_questions.json <<EOF
{
  "definition": {
    "exampleQuestions": [
      {
        "question": "Total booked revenue last 30 days",
        "answerKind": "SQL",
        "dataSourceRef": "$LH_ID",
        "sql": "SELECT SUM(amount_usd) AS booked_revenue_usd FROM dbo.orders WHERE status='Closed Won' AND order_date >= DATEADD(day, -30, CAST(GETDATE() AS DATE))"
      },
      {
        "question": "Booked revenue by region",
        "answerKind": "SQL",
        "dataSourceRef": "$LH_ID",
        "sql": "SELECT region, SUM(amount_usd) AS booked_revenue_usd FROM dbo.orders WHERE status='Closed Won' GROUP BY region ORDER BY booked_revenue_usd DESC"
      },
      {
        "question": "Top 5 reps by open pipeline",
        "answerKind": "SQL",
        "dataSourceRef": "$LH_ID",
        "sql": "SELECT TOP 5 owner_email, SUM(amount_usd) AS open_pipeline_usd FROM dbo.orders WHERE status='Open' GROUP BY owner_email ORDER BY open_pipeline_usd DESC"
      }
    ]
  }
}
EOF

curl -sX PATCH \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d @example_questions.json
```

---

## Step 5 — Publish (30 sec)

Drafts only serve chat to the creator. **Publish** to let other workspace members
(and your own smoke-test below) use it.

```bash
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID/publish" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

---

## Step 6 — Grant chat access (1 min)

In **END_USER mode** (the default), the chatting user needs:
1. **Workspace `Viewer`+** on the agent's workspace (to see/chat the agent)
2. **Read on the lakehouse** (the SQL endpoint enforces this)

```bash
# Add a user as Viewer of the workspace
curl -sX POST "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/roleAssignments" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "principal": {"id": "<user-or-group-object-id>", "type": "User"},
    "role": "Viewer"
  }'
```

> If a user can see the agent in the portal but gets "I don't have access to that
> data," they're missing #2 — the lakehouse SQL endpoint permission. Grant from
> the lakehouse → **Manage permissions** → Read.

---

## Step 7 — Smoke test (1 min)

### Via REST

```bash
# Open a session
SESS_RESP=$(curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID/sessions" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" -d '{}')
SESS_ID=$(echo "$SESS_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Ask
curl -sX POST \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID/sessions/$SESS_ID/messages" \
  -H "Authorization: Bearer $FABRIC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "What is total booked revenue in the last 30 days?"}' \
  | python3 -m json.tool
```

You should see a response containing narration text, the executed T-SQL, and a
result row.

### Via the portal

Open the workspace → click `quickstart-analyst` → **Chat**. Same agent.

### Common failures

| Symptom | Fix |
|---|---|
| `403` creating agent | Workspace role is `Viewer`/`Contributor` — needs `Member`+ |
| `409 CapacityNotEntitled` | Capacity is below F64 or Copilot disabled tenant-wide |
| Agent says "I can't see any tables" | Wrong `itemId`/`tables` in step 3, or you forgot to publish (step 5) |
| `Could not authenticate to data source` | User missing lakehouse read (step 6 #2) |
| Wildly wrong number | Add an example question for that exact phrasing, or tighten `instructions.md` |
| Schema changed but agent uses old columns | Re-PATCH the agent (re-send `dataSources`) and re-publish |

---

## Cleanup

```bash
curl -sX DELETE \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/dataAgents/$AGENT_ID" \
  -H "Authorization: Bearer $FABRIC_TOKEN"

curl -sX DELETE \
  "https://api.fabric.microsoft.com/v1/workspaces/$WS_ID/lakehouses/$LH_ID" \
  -H "Authorization: Bearer $FABRIC_TOKEN"
```

---

## What this quickstart deliberately skips

These belong in the production setup — see **`azure-fabric-data-agents`** skill:

- Service-principal / fixed-identity execution (for embedded apps & Teams bots)
- Multiple grounding sources (warehouse + semantic model + KQL together)
- Power BI **RLS / OLS** enforcement nuances
- Workspace RBAC + OneLake ACL composition rules
- Deployment Pipelines for Dev → Test → Prod promotion
- Purview integration / sensitivity labels propagation
- Eval harness with regression tracking against a golden Q&A set
- Embedded chat in Teams / Power BI / custom apps
- Audit logs (M365 unified audit) for "who asked what"
- Per-domain agent decomposition vs. mega-agent anti-pattern

---

## One-shot script

```bash
# fabric_quickstart.sh
set -euo pipefail
: "${WS_ID:?set WS_ID to your Fabric workspace GUID}"
: "${AGENT_NAME:=quickstart-analyst}"
: "${LAKEHOUSE_NAME:=quickstart_lh}"

FABRIC_TOKEN=$(az account get-access-token \
  --resource https://api.fabric.microsoft.com --query accessToken -o tsv)
H=(-H "Authorization: Bearer $FABRIC_TOKEN" -H "Content-Type: application/json")
BASE="https://api.fabric.microsoft.com/v1/workspaces/$WS_ID"

LH_ID=$(curl -sX POST "$BASE/lakehouses" "${H[@]}" \
  -d "{\"displayName\":\"$LAKEHOUSE_NAME\"}" | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")
echo "Lakehouse $LH_ID — now load dbo.orders via notebook (see quickstart Step 1)."
read -p "Press Enter once dbo.orders is populated and described..."

cat > /tmp/instructions.md <<'EOF'
You are the Demo Analyst. Use ONLY dbo.orders in quickstart_lh.
For "revenue"/"bookings" use status='Closed Won'. For "pipeline"/"open" use status='Open'.
Always show the executed SQL.
EOF
INST=$(python3 -c "import json;print(json.dumps(open('/tmp/instructions.md').read()))")

AGENT_ID=$(curl -sX POST "$BASE/dataAgents" "${H[@]}" -d "{
  \"displayName\":\"$AGENT_NAME\",
  \"definition\":{
    \"instructions\": $INST,
    \"dataSources\":[{\"type\":\"LakehouseSqlEndpoint\",\"workspaceId\":\"$WS_ID\",
                       \"itemId\":\"$LH_ID\",\"schemas\":[\"dbo\"],\"tables\":[\"dbo.orders\"]}]
  }}" | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

curl -sX PATCH "$BASE/dataAgents/$AGENT_ID" "${H[@]}" -d "{
  \"definition\":{\"exampleQuestions\":[
    {\"question\":\"Total booked revenue last 30 days\",\"answerKind\":\"SQL\",\"dataSourceRef\":\"$LH_ID\",
     \"sql\":\"SELECT SUM(amount_usd) FROM dbo.orders WHERE status='Closed Won' AND order_date >= DATEADD(day,-30,CAST(GETDATE() AS DATE))\"},
    {\"question\":\"Booked revenue by region\",\"answerKind\":\"SQL\",\"dataSourceRef\":\"$LH_ID\",
     \"sql\":\"SELECT region, SUM(amount_usd) FROM dbo.orders WHERE status='Closed Won' GROUP BY region\"},
    {\"question\":\"Top 5 reps by open pipeline\",\"answerKind\":\"SQL\",\"dataSourceRef\":\"$LH_ID\",
     \"sql\":\"SELECT TOP 5 owner_email, SUM(amount_usd) FROM dbo.orders WHERE status='Open' GROUP BY owner_email ORDER BY 2 DESC\"}
  ]}}"

curl -sX POST "$BASE/dataAgents/$AGENT_ID/publish" "${H[@]}"

SESS_ID=$(curl -sX POST "$BASE/dataAgents/$AGENT_ID/sessions" "${H[@]}" -d '{}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['id'])")

curl -sX POST "$BASE/dataAgents/$AGENT_ID/sessions/$SESS_ID/messages" "${H[@]}" \
  -d '{"content":"What is total booked revenue in the last 30 days?"}' | python3 -m json.tool
```

---

## See also

- **`azure-fabric-data-agents`** — full reference: identity modes, governance, eval, embedding
