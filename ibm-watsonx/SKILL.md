---
name: ibm-watsonx
description: watsonx.ai (foundation models, prompt lab, tuning), watsonx.data (Presto/Iceberg lakehouse), watsonx.governance (AI factsheets, monitoring, bias), Python SDK and REST API patterns
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools:
  - shell
  - read_file
  - write_file
  - glob
  - grep
---

# IBM watsonx Skill

Covers all three watsonx pillars: **watsonx.ai** (foundation model inference and fine-tuning),
**watsonx.data** (open lakehouse), and **watsonx.governance** (AI lifecycle management).

---

## watsonx.ai — Foundation Models

### Provisioning
watsonx.ai runs in an IBM Cloud project. Provision via IBM Cloud catalog → "IBM watsonx.ai".

```bash
# Create a watsonx.ai instance (Lite or Plus plan)
ibmcloud resource service-instance-create my-watsonx \
  "pm-20" \
  "lite" \
  us-south

# Get API endpoint for your region
# us-south: https://us-south.ml.cloud.ibm.com
# eu-de:    https://eu-de.ml.cloud.ibm.com
# jp-tok:   https://jp-tok.ml.cloud.ibm.com
# au-syd:   https://au-syd.ml.cloud.ibm.com
```

### Python SDK — ibm-watsonx-ai

```bash
pip install ibm-watsonx-ai
```

```python
from ibm_watsonx_ai import APIClient, Credentials

credentials = Credentials(
    url="https://us-south.ml.cloud.ibm.com",
    api_key="YOUR_IBM_CLOUD_API_KEY",
)
client = APIClient(credentials)

# Set a project (required for inference)
client.set.default_project(project_id="your-project-id")
# Or a deployment space
client.set.default_space(space_id="your-space-id")
```

### Text Generation — Core Pattern

```python
from ibm_watsonx_ai.foundation_models import ModelInference
from ibm_watsonx_ai.metanames import GenTextParamsMetaNames as Params

model = ModelInference(
    model_id="ibm/granite-3-8b-instruct",   # see model list below
    credentials=credentials,
    project_id="your-project-id",
    params={
        Params.MAX_NEW_TOKENS: 512,
        Params.MIN_NEW_TOKENS: 1,
        Params.TEMPERATURE: 0.7,
        Params.TOP_P: 0.95,
        Params.TOP_K: 50,
        Params.STOP_SEQUENCES: ["\n\n"],
        Params.REPETITION_PENALTY: 1.1,
    }
)

# Single call
response = model.generate_text(prompt="Explain quantum entanglement in two sentences.")
print(response)

# Streaming
for chunk in model.generate_text_stream(prompt="Write a haiku about distributed systems."):
    print(chunk, end="", flush=True)

# Full response object (includes token counts, stop reason)
result = model.generate(prompt="Summarize: ...")
print(result["results"][0]["generated_text"])
print(result["results"][0]["stop_reason"])          # eos_token, max_tokens, etc.
print(result["results"][0]["generated_token_count"])
```

### Foundation Models Available (April 2026)

| Model ID | Description |
|----------|-------------|
| `ibm/granite-3-8b-instruct` | IBM Granite 3.0 8B — general purpose, multilingual |
| `ibm/granite-3-2b-instruct` | IBM Granite 3.0 2B — lightweight, fast |
| `ibm/granite-20b-multilingual` | Granite 20B — multilingual tasks |
| `ibm/granite-34b-code-instruct` | Granite 34B — code generation |
| `ibm/granite-8b-code-instruct` | Granite 8B — code tasks |
| `meta-llama/llama-3-70b-instruct` | Meta Llama 3 70B |
| `meta-llama/llama-3-8b-instruct` | Meta Llama 3 8B |
| `mistralai/mixtral-8x7b-instruct-v01` | Mixtral MoE 8×7B |
| `mistralai/mistral-large` | Mistral Large |
| `ibm/granite-guardian-3-8b` | Granite Guardian — safety/alignment |

```python
# List available models programmatically
from ibm_watsonx_ai.foundation_models.utils.enums import ModelTypes
for m in client.foundation_models.get_model_specs()["resources"]:
    print(m["model_id"], "-", m.get("short_description", ""))
```

### Prompt Templates

```python
from ibm_watsonx_ai.foundation_models.prompts import PromptTemplate

# Build a structured prompt for Granite instruct format
prompt = f"""<|system|>
You are a helpful assistant. Be concise.
<|user|>
{user_input}
<|assistant|>
"""

# Llama 3 instruct format
prompt = f"""<|begin_of_text|><|start_header_id|>system<|end_header_id|>
You are a helpful assistant.<|eot_id|><|start_header_id|>user<|end_header_id|>
{user_input}<|eot_id|><|start_header_id|>assistant<|end_header_id|>
"""
```

### Embeddings

```python
from ibm_watsonx_ai.foundation_models import Embeddings

embed = Embeddings(
    model_id="ibm/slate-30m-english-rtrvr",  # or slate-125m-english-rtrvr
    credentials=credentials,
    project_id="your-project-id",
)
vectors = embed.embed_documents(["Text to embed", "Another document"])
query_vector = embed.embed_query("search query")
```

### Tuning Studio — Fine-Tuning

watsonx.ai supports prompt tuning (soft prompts) and LoRA fine-tuning on select models.

```python
from ibm_watsonx_ai.foundation_models.tuning import TuningAsset

tuning = TuningAsset(
    client=client,
    model_id="ibm/granite-3-8b-instruct",
    task_id="generation",
    name="my-tuned-model",
    training_data_reference={
        "type": "container",
        "location": {"path": "training_data/"},
    },
    num_epochs=3,
    learning_rate=0.3,
    accumulate_steps=16,
    verbalizer="Input: {{input}} Output:",
)
tuning_id = tuning.run()
```

### Synthetic Data Generation

Use watsonx.ai + Granite to generate synthetic training data programmatically:

```python
template = """Generate {n} diverse question-answer pairs about {topic}.
Format as JSON array: [{{"question": "...", "answer": "..."}}]"""

synthetic = model.generate_text(
    prompt=template.format(n=20, topic="cloud networking"),
)
import json
pairs = json.loads(synthetic)
```

---

## REST API Patterns

IBM Cloud uses IAM tokens for REST API auth. Exchange your API key for a bearer token:

```bash
# Get IAM token (valid ~1 hour)
IAM_TOKEN=$(curl -s -X POST "https://iam.cloud.ibm.com/identity/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=urn:ibm:params:oauth:grant-type:apikey&apikey=$IBMCLOUD_API_KEY" | \
  jq -r '.access_token')

# Text generation via REST
curl -s -X POST "https://us-south.ml.cloud.ibm.com/ml/v1/text/generation?version=2023-05-29" \
  -H "Authorization: Bearer $IAM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model_id": "ibm/granite-3-8b-instruct",
    "input": "What is 2+2?",
    "parameters": {"max_new_tokens": 50},
    "project_id": "'"$PROJECT_ID"'"
  }' | jq '.results[0].generated_text'
```

---

## watsonx.data — Open Lakehouse

watsonx.data is a managed Presto/Spark lakehouse with Apache Iceberg table format.

### Connecting via Presto (JDBC/Python)

```python
# Install: pip install pyhive thrift thrift-sasl
from pyhive import presto

conn = presto.connect(
    host="your-instance.lakehouse.cloud.ibm.com",
    port=8443,
    username="ibmlhapikey",           # literal string — not your username
    password=ibm_cloud_api_key,       # your API key goes here
    requests_kwargs={"verify": False}, # or point to IBM cert bundle
    protocol="https",
)
cursor = conn.cursor()
cursor.execute("SHOW CATALOGS")
print(cursor.fetchall())

cursor.execute("""
    SELECT * FROM iceberg_data.my_schema.my_table
    LIMIT 100
""")
rows = cursor.fetchall()
```

### Iceberg Tables

```sql
-- Create Iceberg table via Presto
CREATE TABLE iceberg_data.my_schema.events (
    event_id VARCHAR,
    event_time TIMESTAMP(6) WITH TIME ZONE,
    user_id BIGINT,
    payload VARCHAR
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY['day(event_time)']
);

-- Time travel
SELECT * FROM iceberg_data.my_schema.events
FOR TIMESTAMP AS OF TIMESTAMP '2026-01-01 00:00:00 UTC';

-- Snapshot rollback
CALL iceberg_data.system.rollback_to_snapshot('my_schema.events', 1234567890);

-- Expire old snapshots
CALL iceberg_data.system.expire_snapshots('my_schema.events',
    TIMESTAMP '2026-01-01 00:00:00 UTC');
```

### Data Connectors

watsonx.data connects to external sources (Db2, PostgreSQL, S3, Hive) via catalogs.
Configure in the console or via REST API — not CLI-configurable directly.

---

## watsonx.governance — AI Lifecycle

watsonx.governance tracks models from development through production with factsheets,
monitoring for drift/bias, and approval workflows.

### AI Factsheets (OpenScale / Watson OpenScale successor)

```python
from ibm_aigov_facts_client import AIGovFactsClient

client = AIGovFactsClient(
    api_key="YOUR_IBM_CLOUD_API_KEY",
    container_id="your-project-or-space-id",
    container_type="space",              # or "project"
    experiment_name="my-experiment",
    run_name="run-v1",
)

# Log model metadata
client.export_facts.log_model_info(
    model_id="my-model",
    model_type="binary_classification",
    training_data_reference="cos://bucket/data",
    label_column="churn",
    feature_columns=["age", "balance", "tenure"],
)

# Log metrics
client.export_facts.log_model_metrics({
    "training_accuracy": 0.94,
    "f1_score": 0.91,
    "auc": 0.97,
})
```

### Model Monitoring (Watson OpenScale API)

```python
from ibm_watson_openscale import APIClient as WOSClient
from ibm_watson_openscale.supporting_classes.enums import TargetTypes

wos = WOSClient(
    service_url="https://aiopenscale.cloud.ibm.com",
    authenticator=IAMAuthenticator(api_key),
)

# List monitors on a deployment
monitors = wos.monitor_instances.list().result
for m in monitors["monitor_instances"]:
    print(m["entity"]["monitor_definition_id"], m["entity"]["status"]["state"])
```

### Bias Detection

Bias monitors check for disparate impact across protected attributes:
- Configure via watsonx.governance console (preferred for initial setup)
- Monitors run on schedule or on-demand
- Output: `favorable_rate` by group, `disparate_impact_ratio` (should be > 0.8)

---

## Guardrails

- **Project ID vs Space ID** — development uses projects, production deployment uses spaces. Wrong ID = 404.
- **Token expiry** — IAM tokens last ~1 hour. In long-running scripts, refresh before expiry.
- **Model availability is region-specific** — not all models are available in all regions; check the console.
- **Token limits** — Granite 3 8B context is 4096 tokens input; Llama 3 70B is 8192. Chunk long docs.
- **Tuning costs** — fine-tuning consumes CUH (Capacity Unit Hours); check plan limits before kicking off jobs.
- **watsonx.data Presto** — does not support DML (UPDATE/DELETE) on non-Iceberg tables; use Iceberg.
- **Governance is additive** — start with factsheets first, layer on bias/drift monitoring incrementally.
