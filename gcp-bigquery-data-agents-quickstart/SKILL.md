---
name: gcp-bigquery-data-agents-quickstart
description: Fast-path quickstart for standing up a GCP BigQuery Data Agent (Conversational Analytics) in ~15 minutes — minimal IAM, one curated dataset, a system instruction, 3 golden queries, and a smoke test. For the full reference (governance, eval, embedding, gotchas) see the gcp-bigquery-data-agents skill.
license: MIT
version: 1.0.0
updated: 2026-04-27
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
---

# GCP BigQuery Data Agents — Quickstart

A 15-minute path from zero to a working **BigQuery Data Agent** you can chat with.
Optimized for proof-of-concept / first-agent setup. For production governance,
evaluation harnesses, embedding patterns, and the full gotcha list, see the
companion skill **`gcp-bigquery-data-agents`**.

> Naming note: API is `geminidataanalytics.googleapis.com`. CLI noun is
> `gcloud alpha geminidataanalytics`. Product name: BigQuery Data Agents.

---

## Prereqs (1 min)

```bash
# Set these once
export PROJECT_ID="my-project"
export LOCATION="us-central1"
export AGENT_ID="quickstart-analyst"
export DATASET="demo_curated"          # the BQ dataset the agent will see

gcloud config set project $PROJECT_ID
gcloud auth login
gcloud auth application-default login
```

You need:
- A GCP project with billing enabled
- `roles/owner` (or: `roles/serviceusage.serviceUsageAdmin` + `roles/iam.securityAdmin`
  + `roles/bigquery.admin` + `roles/geminidataanalytics.dataAgentAdmin`)
- One BigQuery dataset with at least one table that has data in it

---

## Step 1 — Enable APIs (1 min)

```bash
gcloud services enable \
  geminidataanalytics.googleapis.com \
  bigquery.googleapis.com \
  --project=$PROJECT_ID
```

---

## Step 2 — Prepare a curated dataset (3 min)

If you already have a dataset, **add table & column descriptions** — the agent
reads them and quality jumps dramatically.

```bash
# Create a demo dataset + table if you don't have one
bq --location=US mk -d --description "Demo curated dataset for quickstart agent" \
  $PROJECT_ID:$DATASET

bq query --use_legacy_sql=false """
CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.orders\` AS
SELECT
  GENERATE_UUID()                                                  AS order_id,
  DATE_SUB(CURRENT_DATE(), INTERVAL CAST(RAND()*180 AS INT64) DAY) AS order_date,
  ['NA','EMEA','APAC','LATAM'][OFFSET(CAST(RAND()*4 AS INT64))]    AS region,
  ['Closed Won','Closed Lost','Open'][OFFSET(CAST(RAND()*3 AS INT64))] AS status,
  ROUND(RAND()*10000, 2)                                           AS amount_usd,
  CONCAT('rep', CAST(CAST(RAND()*10 AS INT64) AS STRING), '@example.com') AS owner_email
FROM UNNEST(GENERATE_ARRAY(1, 5000));
"""
```

Now annotate it (this is the highest-ROI step for agent quality):

```bash
bq query --use_legacy_sql=false """
ALTER TABLE \`$PROJECT_ID.$DATASET.orders\`
SET OPTIONS (description = 'One row per order. Use status=\"Closed Won\" for booked revenue. Use order_date for revenue recognition.');

ALTER TABLE \`$PROJECT_ID.$DATASET.orders\`
ALTER COLUMN amount_usd      SET OPTIONS (description = 'Order amount in USD. SUM(amount_usd) WHERE status=\"Closed Won\" = booked revenue.');

ALTER TABLE \`$PROJECT_ID.$DATASET.orders\`
ALTER COLUMN status          SET OPTIONS (description = 'One of: Closed Won (booked), Closed Lost, Open (still in pipeline).');

ALTER TABLE \`$PROJECT_ID.$DATASET.orders\`
ALTER COLUMN region          SET OPTIONS (description = 'Sales region: NA, EMEA, APAC, LATAM.');

ALTER TABLE \`$PROJECT_ID.$DATASET.orders\`
ALTER COLUMN owner_email     SET OPTIONS (description = 'Email of the rep who owns the order.');
"""
```

---

## Step 3 — Write a minimal system instruction (1 min)

```bash
mkdir -p ./agent
cat > ./agent/instructions.md <<'EOF'
You are the Demo Analyst. You answer questions about orders using ONLY the
`demo_curated` BigQuery dataset.

## Scope
- For "revenue" or "bookings", default to `status='Closed Won'`.
- For "pipeline" or "open", use `status='Open'`.
- All amounts are USD in `amount_usd`.

## Refusals
- If a question requires a dataset other than `demo_curated`, say so and stop.
- If asked for PII beyond `owner_email`, refuse.

## Output
- Always show the executed SQL beneath the answer.
- Round currency to whole dollars with thousands separators.
EOF
```

---

## Step 4 — Write 3 golden queries (2 min)

Three is the sweet spot for a quickstart. Don't add 50 — quality > quantity.

```bash
cat > ./agent/golden_queries.yaml <<EOF
golden_queries:
  - natural_language: "Total booked revenue last 30 days"
    sql: |
      SELECT SUM(amount_usd) AS booked_revenue_usd
      FROM   \`$PROJECT_ID.$DATASET.orders\`
      WHERE  status     = 'Closed Won'
        AND  order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)

  - natural_language: "Booked revenue by region"
    sql: |
      SELECT region,
             SUM(amount_usd) AS booked_revenue_usd
      FROM   \`$PROJECT_ID.$DATASET.orders\`
      WHERE  status = 'Closed Won'
      GROUP  BY region
      ORDER  BY booked_revenue_usd DESC

  - natural_language: "Top 5 reps by open pipeline"
    sql: |
      SELECT owner_email,
             SUM(amount_usd) AS open_pipeline_usd
      FROM   \`$PROJECT_ID.$DATASET.orders\`
      WHERE  status = 'Open'
      GROUP  BY owner_email
      ORDER  BY open_pipeline_usd DESC
      LIMIT  5
EOF
```

---

## Step 5 — Create the agent (2 min)

```bash
gcloud alpha geminidataanalytics data-agents create $AGENT_ID \
  --location=$LOCATION \
  --display-name="Quickstart Analyst" \
  --description="Demo agent over $DATASET" \
  --system-instruction-file=./agent/instructions.md \
  --data-source-bigquery-dataset=projects/$PROJECT_ID/datasets/$DATASET \
  --max-bytes-billed=10737418240   # 10 GiB cap — important even for demo
```

Attach the golden queries (the flag name varies by gcloud version; if `--context-file`
isn't recognized, use the REST patch shown at the bottom):

```bash
gcloud alpha geminidataanalytics data-agents update $AGENT_ID \
  --location=$LOCATION \
  --context-file=./agent/golden_queries.yaml
```

---

## Step 6 — Grant yourself chat access (1 min)

The two roles you need *separately*:

```bash
ME=$(gcloud config get-value account)

# 1. Permission to chat with the agent
gcloud alpha geminidataanalytics data-agents add-iam-policy-binding $AGENT_ID \
  --location=$LOCATION \
  --role=roles/geminidataanalytics.dataAgentUser \
  --member=user:$ME

# 2. Permission to actually read the data (END_USER_CREDENTIALS mode is default)
bq add-iam-policy-binding \
  --member=user:$ME \
  --role=roles/bigquery.dataViewer \
  $PROJECT_ID:$DATASET

# 3. Permission to run BQ jobs in this project
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:$ME \
  --role=roles/bigquery.jobUser
```

> If a user can chat but gets "permission denied" on the data, they're missing
> #2 or #3 — not #1. This is the #1 cause of "the agent doesn't work" tickets.

---

## Step 7 — Smoke test (30 sec)

```bash
gcloud alpha geminidataanalytics data-agents chat $AGENT_ID \
  --location=$LOCATION \
  --message="What's the total booked revenue in the last 30 days?"
```

Expected output: a narrated answer, the executed SQL, and a result row.
If you instead see:

| Symptom | Fix |
|---|---|
| `PERMISSION_DENIED` on agent | Step 6.1 didn't take effect — wait 60s and retry |
| `PERMISSION_DENIED` on BigQuery | Step 6.2 / 6.3 missing |
| Agent says "I don't see that table" | Wrong dataset path in step 5; or run `gcloud alpha geminidataanalytics data-agents refresh $AGENT_ID --location=$LOCATION` |
| Wildly wrong number | Add a golden query for that exact question, or tighten `instructions.md` |
| `INVALID_ARGUMENT: bytes billed` | Query exceeded `--max-bytes-billed`; raise it or narrow the question |

---

## Step 8 — Try it from the UI (optional)

Open BigQuery Studio → **Data agents** → `quickstart-analyst` → click **Chat**.
Same agent, same answers, browser UI.

---

## Cleanup

```bash
gcloud alpha geminidataanalytics data-agents delete $AGENT_ID \
  --location=$LOCATION --quiet

bq rm -r -f -d $PROJECT_ID:$DATASET
```

---

## What this quickstart deliberately skips

These belong in the production setup — see **`gcp-bigquery-data-agents`** skill:

- SERVICE_ACCOUNT execution mode (for Slack bots / scheduled runs)
- Row-access policies & column-level policy tags
- Dataplex business glossary integration
- Looker semantic model attachment
- Audit-log queries for "who asked what"
- Golden-set evaluation harness with regression tracking
- Multi-agent / per-domain decomposition
- Embedded iframe / Slack bot integration
- Snapshotting curated data for reproducible answers

---

## REST fallback (if `--context-file` flag isn't in your gcloud version)

```bash
ACCESS_TOKEN=$(gcloud auth print-access-token)

curl -X PATCH \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://geminidataanalytics.googleapis.com/v1/projects/$PROJECT_ID/locations/$LOCATION/dataAgents/$AGENT_ID?updateMask=context" \
  -d @<(python3 -c "
import json, yaml, sys
ctx = yaml.safe_load(open('./agent/golden_queries.yaml'))
print(json.dumps({'context': {'goldenQueries': ctx['golden_queries']}}))
")
```

---

## One-shot script

Everything above as a single runnable file:

```bash
# quickstart.sh
set -euo pipefail
: "${PROJECT_ID:?}"; : "${LOCATION:=us-central1}"
: "${AGENT_ID:=quickstart-analyst}"; : "${DATASET:=demo_curated}"

gcloud config set project "$PROJECT_ID"
gcloud services enable geminidataanalytics.googleapis.com bigquery.googleapis.com

bq --location=US mk -f -d "$PROJECT_ID:$DATASET"
bq query --use_legacy_sql=false "CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET.orders\` AS
SELECT GENERATE_UUID() AS order_id,
       DATE_SUB(CURRENT_DATE(), INTERVAL CAST(RAND()*180 AS INT64) DAY) AS order_date,
       ['NA','EMEA','APAC','LATAM'][OFFSET(CAST(RAND()*4 AS INT64))]    AS region,
       ['Closed Won','Closed Lost','Open'][OFFSET(CAST(RAND()*3 AS INT64))] AS status,
       ROUND(RAND()*10000, 2)                                           AS amount_usd,
       CONCAT('rep', CAST(CAST(RAND()*10 AS INT64) AS STRING), '@example.com') AS owner_email
FROM UNNEST(GENERATE_ARRAY(1,5000));"

mkdir -p ./agent
cat > ./agent/instructions.md <<'EOF'
You are the Demo Analyst. Use ONLY the demo_curated dataset.
For "revenue"/"bookings" use status='Closed Won'. For "pipeline"/"open" use status='Open'.
Always show the executed SQL.
EOF

gcloud alpha geminidataanalytics data-agents create "$AGENT_ID" \
  --location="$LOCATION" \
  --display-name="Quickstart Analyst" \
  --system-instruction-file=./agent/instructions.md \
  --data-source-bigquery-dataset="projects/$PROJECT_ID/datasets/$DATASET" \
  --max-bytes-billed=10737418240

ME=$(gcloud config get-value account)
gcloud alpha geminidataanalytics data-agents add-iam-policy-binding "$AGENT_ID" \
  --location="$LOCATION" --role=roles/geminidataanalytics.dataAgentUser --member="user:$ME"
bq add-iam-policy-binding --member="user:$ME" --role=roles/bigquery.dataViewer "$PROJECT_ID:$DATASET"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="user:$ME" --role=roles/bigquery.jobUser

sleep 30  # IAM propagation

gcloud alpha geminidataanalytics data-agents chat "$AGENT_ID" \
  --location="$LOCATION" \
  --message="What is total booked revenue in the last 30 days?"
```

---

## See also

- **`gcp-bigquery-data-agents`** — full reference: governance, eval, embedding, gotchas
- **`azure-fabric-data-agents`** — equivalent on the Microsoft Fabric side
