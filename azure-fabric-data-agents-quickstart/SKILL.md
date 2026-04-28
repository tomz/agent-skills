---
name: azure-fabric-data-agents-quickstart
description: Fast-path quickstart for standing up a Microsoft Fabric Data Agent in ~20 minutes — using the Microsoft fabric-data-agent-sdk inside a Fabric notebook. Lakehouse + table descriptions + 3 few-shot examples + publish + smoke test. For the full reference (governance, Purview, multi-source, sharing, evaluation) see the azure-fabric-data-agents skill.
license: MIT
version: 2.0.0
updated: 2026-04-28
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---

# Microsoft Fabric Data Agents — Quickstart

A 20-minute path from zero to a working **Fabric Data Agent** you can chat with
from the Fabric portal or from a notebook. Optimized for proof-of-concept /
first-agent setup. For production governance, sharing model, evaluation harness,
multi-source patterns, and the full gotcha list, see the companion skill
**`azure-fabric-data-agents`**.

> **Sources** — this quickstart is grounded in the Microsoft Learn docs at
> `learn.microsoft.com/en-us/fabric/data-science/how-to-create-data-agent`,
> `…/concept-data-agent`, `…/fabric-data-agent-sdk`, and the official sample
> notebooks at `github.com/microsoft/fabric-samples/tree/main/docs-samples/data-science/data-agent-sdk`.
> Programmatic management uses the **`fabric-data-agent-sdk`** Python package
> (preview, designed to run inside a Fabric notebook). The SDK is **not**
> supported for local execution — all code below assumes a Fabric notebook.

---

## Prereqs

You need:
- A Fabric tenant with **Data Agents tenant switches enabled** (Admin portal →
  Tenant settings → "AI skill", "Copilot for Fabric", and the **cross-geo
  processing / cross-geo storing for AI** toggles per
  `data-agent-tenant-settings`).
- A capacity:
  - **Per Microsoft prerequisites**: a paid **F2 or higher** F SKU, or P1+
    Power BI Premium with Fabric enabled.
  - **Per the official SDK sample notebook**: F64+ is recommended for
    end-to-end reliability of the OpenAI-Assistants-API consumption path.
  - The honest truth: F2 will let you create and chat in the portal; the SDK
    consumption sample explicitly states F64. Start on whatever you have.
- A workspace where you have **Contributor** role or higher (Contributor is
  enough — that's what the REST `Create DataAgent` API requires).
- At least one of: **lakehouse, warehouse, Power BI semantic model, KQL
  database, mirrored database, or ontology** — with data and **Read access**
  for the user who will chat.

---

## Step 1 — Create a lakehouse with a demo table (5 min)

In the Fabric portal:
1. Open your workspace → **+ New item** → search **Lakehouse** → name it
   `quickstart_lh`.
2. Open the lakehouse → **Open notebook** → **New notebook** (this attaches
   the lakehouse as the default).

Paste and run:

```python
# Cell 1 — generate a demo orders table
from pyspark.sql import functions as F, types as T
import random, datetime

regions  = ["NA", "EMEA", "APAC", "LATAM"]
statuses = ["Closed Won", "Closed Lost", "Open"]
today    = datetime.date.today()
rows = [(
    f"o-{i:05d}",
    today - datetime.timedelta(days=random.randint(0, 180)),
    random.choice(regions),
    random.choice(statuses),
    round(random.random()*10000, 2),
    f"rep{random.randint(0,9)}@example.com",
) for i in range(5000)]

schema = T.StructType([
    T.StructField("order_id",    T.StringType()),
    T.StructField("order_date",  T.DateType()),
    T.StructField("region",      T.StringType()),
    T.StructField("status",      T.StringType()),
    T.StructField("amount_usd",  T.DoubleType()),
    T.StructField("owner_email", T.StringType()),
])
spark.createDataFrame(rows, schema=schema) \
     .write.mode("overwrite").format("delta").saveAsTable("orders")
```

### Add table & column descriptions (high-ROI step)

The agent reads schema descriptions when generating SQL. Use Spark SQL on the
underlying Delta table — the lakehouse SQL endpoint is **read-only** so DDL
must go through Spark, not T-SQL.

```python
# Cell 2 — describe the table & columns (Delta column comments)
spark.sql("""
ALTER TABLE orders SET TBLPROPERTIES (
  'comment' = 'One row per order. Use status="Closed Won" for booked revenue. Use order_date for revenue recognition.'
)
""")

for col, desc in [
    ("amount_usd",  'Order amount in USD. SUM(amount_usd) WHERE status="Closed Won" = booked revenue.'),
    ("status",      'One of: Closed Won (booked), Closed Lost, Open (still in pipeline).'),
    ("region",      'Sales region: NA, EMEA, APAC, LATAM.'),
    ("owner_email", 'Email of the rep who owns the order.'),
    ("order_date",  'Calendar date the order was placed/closed.'),
]:
    spark.sql(f"ALTER TABLE orders ALTER COLUMN {col} COMMENT '{desc}'")
```

> Skipping the descriptions is the single biggest cause of a vague-feeling agent.

---

## Step 2 — Create the agent via the SDK (3 min)

Still in the Fabric notebook:

```python
# Cell 3 — install SDK (only needed once per session/environment)
%pip install fabric-data-agent-sdk
```

```python
# Cell 4 — create the agent
from fabric.dataagent.client import (
    FabricDataAgentManagement,
    create_data_agent,
    delete_data_agent,
)

DATA_AGENT_NAME = "quickstart-analyst"

# Creates the workspace item. Use FabricDataAgentManagement(name) to attach
# to an existing one (create_data_agent on an existing name errors out).
data_agent = create_data_agent(DATA_AGENT_NAME)
data_agent.get_configuration()
```

---

## Step 3 — Add the lakehouse and select tables (2 min)

```python
# Cell 5 — attach the lakehouse
# datasource type can be: "lakehouse", "warehouse", "kqldatabase", "semanticmodel"
data_agent.add_datasource("quickstart_lh", type="lakehouse")

datasource = data_agent.get_datasources()[0]
datasource.pretty_print()   # tables NOT marked "*" are NOT visible to the agent
```

```python
# Cell 6 — explicitly select the table(s) you want the agent to use
datasource.select("dbo", "orders")
datasource.pretty_print()   # "orders" should now have a "*"
```

> The agent ignores any unselected tables. This is your scoping mechanism:
> select only what the agent should see.

---

## Step 4 — Add agent + datasource instructions (1 min)

```python
# Cell 7 — agent-level instructions (persona, refusal policy)
agent_instructions = """\
You are the Demo Analyst. Answer questions about orders using ONLY the
dbo.orders table in the quickstart_lh lakehouse.

Rules:
- For "revenue" or "bookings": filter status = 'Closed Won'.
- For "pipeline" or "open":     filter status = 'Open'.
- All amounts are USD in amount_usd.
- Always show the executed SQL beneath the answer.
- Round currency to whole dollars with thousands separators.
- If a question is out of scope (root cause, prediction, factors not in the
  table), say so and stop. Do not fabricate.
"""
data_agent.update_configuration(instructions=agent_instructions)
```

```python
# Cell 8 — datasource-specific instructions (query-generation hints)
datasource.update_configuration(instructions="""\
- Booked revenue = SUM(amount_usd) WHERE status = 'Closed Won'.
- Open pipeline  = SUM(amount_usd) WHERE status = 'Open'.
- "Last 30 days" means order_date >= current_date - 30.
""")
```

---

## Step 5 — Add 3 few-shot examples (2 min)

Three is the sweet spot for a quickstart. Don't bulk-load 50 — quality beats
quantity for routing.

```python
# Cell 9 — few-shot examples (NL → SQL, lakehouse Spark SQL dialect)
examples = {
    "Total booked revenue last 30 days":
        "SELECT SUM(amount_usd) AS booked_revenue_usd "
        "FROM dbo.orders "
        "WHERE status = 'Closed Won' "
        "  AND order_date >= date_sub(current_date(), 30)",

    "Booked revenue by region":
        "SELECT region, SUM(amount_usd) AS booked_revenue_usd "
        "FROM dbo.orders "
        "WHERE status = 'Closed Won' "
        "GROUP BY region "
        "ORDER BY booked_revenue_usd DESC",

    "Top 5 reps by open pipeline":
        "SELECT owner_email, SUM(amount_usd) AS open_pipeline_usd "
        "FROM dbo.orders "
        "WHERE status = 'Open' "
        "GROUP BY owner_email "
        "ORDER BY open_pipeline_usd DESC "
        "LIMIT 5",
}

datasource.add_fewshots(examples)
datasource.get_fewshots()   # verify
```

---

## Step 6 — Publish (30 sec)

Until you publish, only you (the creator) can use the agent. Publishing
snapshots the current draft into the consumable version.

```python
# Cell 10 — publish
data_agent.publish()
```

---

## Step 7 — Smoke test (2 min)

### A. From the Fabric portal (recommended for first test)

1. Open the workspace → click `quickstart-analyst` → the chat surface opens.
2. Ask: *"What is total booked revenue in the last 30 days?"*
3. You should see a narrated answer + the executed SQL + a result row.

### B. From the notebook via the OpenAI client wrapper

```python
# Cell 11 — programmatic chat using the bundled OpenAI client
from fabric.dataagent.client import FabricOpenAI

fabric_client = FabricOpenAI(artifact_name=DATA_AGENT_NAME)
assistant = fabric_client.beta.assistants.create(model="gpt-4o")
thread    = fabric_client.beta.threads.create()

# Send a message
fabric_client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="What is total booked revenue in the last 30 days?",
)

# Run and wait
run = fabric_client.beta.threads.runs.create_and_poll(
    thread_id=thread.id, assistant_id=assistant.id,
)
print("status:", run.status)

# Read the assistant's reply
msgs = fabric_client.beta.threads.messages.list(thread_id=thread.id)
for m in msgs.data:
    if m.role == "assistant":
        for part in m.content:
            if part.type == "text":
                print(part.text.value)
        break
```

### Common failures

| Symptom | Fix |
|---|---|
| `403` creating the agent | You're a Viewer; need Contributor+ on the workspace |
| `409 ConflictName` from `create_data_agent` | Agent already exists — use `FabricDataAgentManagement(name)` instead |
| Agent says "I can't see any tables" | You forgot `datasource.select("dbo", "orders")` (Step 3) |
| Other users get "no access" | You forgot `data_agent.publish()` (Step 6), or they lack Read on the lakehouse |
| Wildly wrong number | Add a few-shot example for that exact phrasing, or tighten agent instructions |
| `%pip install` fails | Tenant blocks public PyPI — install `fabric-data-agent-sdk` into a Fabric Environment instead |
| Cross-region error | Agent's workspace capacity must be in the same region as the data source's capacity |

---

## Cleanup

```python
from fabric.dataagent.client import delete_data_agent
delete_data_agent(DATA_AGENT_NAME)
```

(Then delete the lakehouse from the workspace UI if you no longer need it.)

---

## What this quickstart deliberately skips

These belong in the production setup — see **`azure-fabric-data-agents`** skill:

- **Sharing model** — read/write/owner permission tiers, sharing to specific
  users vs. publish-to-workspace.
- **Multi-source agents** — combining lakehouse + warehouse + semantic model +
  KQL (up to 5 sources total per agent).
- **Microsoft Purview governance** — DLP policies in Warehouse (GA), access
  restriction policies (preview) for KQL/SQL DB/Warehouse, sensitivity labels,
  Insider Risk, audit/eDiscovery.
- **Evaluation harness** — `fabric-data-agent-sdk` evaluation utilities and the
  `Fabric-DataAgent-Evaluation-sample.ipynb` pattern for golden-set scoring.
- **Few-shot validator** — `Fabric-DataAgent-Few-Shot-Examples-Validator-sample.ipynb`
  for checking that examples actually return correct results.
- **Cross-tenant data sharing** — querying tables shared into your tenant via
  OneLake external sharing.
- **Out-of-scope question patterns** — what questions to refuse (root-cause,
  forecasting, ML — Data Agents do retrieval, not analytics).

---

## See also

- **`azure-fabric-data-agents`** — full reference (governance, Purview, sharing,
  multi-source, evaluation)
- Microsoft Learn: `learn.microsoft.com/en-us/fabric/data-science/how-to-create-data-agent`
- Microsoft Learn: `learn.microsoft.com/en-us/fabric/data-science/fabric-data-agent-sdk`
- Sample notebooks: `github.com/microsoft/fabric-samples/tree/main/docs-samples/data-science/data-agent-sdk`
