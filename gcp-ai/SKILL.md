---
name: gcp-ai
description: GCP AI/ML — Vertex AI, Gemini API, Document AI, Vision AI, Natural Language AI, AutoML, Vertex AI Search and Agents
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP AI & Machine Learning Skill

Build and deploy AI/ML workloads on Google Cloud using Vertex AI and Google AI services.

---

## Vertex AI Overview

| Service | Use Case |
|---------|---------|
| Model Garden | Access Google, open-source, and partner models |
| Training | Custom model training at scale |
| Endpoints | Online prediction (real-time inference) |
| Batch Prediction | Offline scoring of large datasets |
| Pipelines | MLOps orchestration (Kubeflow Pipelines) |
| Feature Store | Serve/share ML features consistently |
| Model Registry | Versioned model management |
| Experiments | Track runs, metrics, hyperparameters |

---

## Setup

```bash
# Enable required APIs
gcloud services enable aiplatform.googleapis.com \
  storage.googleapis.com \
  artifactregistry.googleapis.com

# Set region (us-central1 has widest model/GPU availability)
gcloud config set compute/region us-central1

# Install Python SDK
pip install google-cloud-aiplatform
```

---

## Gemini API via Vertex AI

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part

vertexai.init(project="my-project", location="us-central1")

model = GenerativeModel("gemini-2.0-flash-001")

# Simple text generation
response = model.generate_content("Explain IAM roles in GCP in 3 bullet points.")
print(response.text)

# Multi-turn chat
chat = model.start_chat()
response = chat.send_message("What is Cloud Run?")
response = chat.send_message("How does it compare to Cloud Functions?")
print(response.text)

# Multimodal: image + text
with open("screenshot.png", "rb") as f:
    image_bytes = f.read()

response = model.generate_content([
    Part.from_data(image_bytes, mime_type="image/png"),
    "Describe what you see in this GCP architecture diagram.",
])
print(response.text)

# Streaming response
for chunk in model.generate_content("Write a Terraform module for GCS", stream=True):
    print(chunk.text, end="", flush=True)
```

### Grounding with Google Search

```python
from vertexai.generative_models import Tool, grounding

model = GenerativeModel("gemini-2.0-flash-001")

google_search_tool = Tool.from_google_search_retrieval(
    grounding.GoogleSearchRetrieval()
)

response = model.generate_content(
    "What are the latest BigQuery pricing changes?",
    tools=[google_search_tool],
)
print(response.text)
```

### Function Calling

```python
from vertexai.generative_models import FunctionDeclaration, Tool

get_weather = FunctionDeclaration(
    name="get_weather",
    description="Get current weather for a city",
    parameters={
        "type": "object",
        "properties": {
            "city": {"type": "string", "description": "City name"},
            "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
        },
        "required": ["city"],
    },
)

weather_tool = Tool(function_declarations=[get_weather])
model = GenerativeModel("gemini-2.0-flash-001", tools=[weather_tool])

response = model.generate_content("What's the weather in Tokyo?")
if response.candidates[0].function_calls:
    fc = response.candidates[0].function_calls[0]
    print(f"Function: {fc.name}, Args: {dict(fc.args)}")
```

---

## Vertex AI Model Garden

```bash
# List available models
gcloud ai models list --region=us-central1

# Deploy a Model Garden model to an endpoint
gcloud ai endpoints deploy-model ENDPOINT_ID \
  --region=us-central1 \
  --model=MODEL_ID \
  --display-name=my-deployment \
  --machine-type=n1-standard-4 \
  --accelerator=count=1,type=nvidia-tesla-t4 \
  --min-replica-count=1 \
  --max-replica-count=3
```

```python
# Deploy Llama or other open models via Model Garden SDK
from vertexai.preview import language_models

# Use a deployed open-source model endpoint
from google.cloud import aiplatform

endpoint = aiplatform.Endpoint(endpoint_name="projects/my-project/locations/us-central1/endpoints/ENDPOINT_ID")
response = endpoint.predict(instances=[{"prompt": "Hello, world!"}])
```

---

## Custom Training

```python
# training_job.py — runs inside a training container
import vertexai
from vertexai.training import CustomJob

vertexai.init(project="my-project", location="us-central1")

job = CustomJob.from_local_script(
    display_name="my-training-job",
    script_path="trainer/train.py",
    container_uri="us-docker.pkg.dev/vertex-ai/training/pytorch-gpu.2-0:latest",
    requirements=["transformers", "datasets"],
    machine_type="n1-standard-8",
    accelerator_type="NVIDIA_TESLA_T4",
    accelerator_count=1,
    replica_count=1,
    args=["--epochs=10", "--learning-rate=0.001"],
    base_output_dir="gs://my-bucket/training-output",
)

job.run(sync=True)  # blocks until complete
```

```bash
# Submit via CLI
gcloud ai custom-jobs create \
  --region=us-central1 \
  --display-name=my-job \
  --config=custom_job.yaml

# List jobs
gcloud ai custom-jobs list --region=us-central1

# Stream logs
gcloud ai custom-jobs stream-logs JOB_ID --region=us-central1

# Cancel a job
gcloud ai custom-jobs cancel JOB_ID --region=us-central1
```

---

## Model Registry and Endpoints

```python
import vertexai
from vertexai.resources import Model

vertexai.init(project="my-project", location="us-central1")

# Upload a model to registry
model = Model.upload(
    display_name="my-model-v1",
    artifact_uri="gs://my-bucket/model-artifacts/",
    serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest",
    labels={"version": "v1", "framework": "sklearn"},
)
print(f"Model resource name: {model.resource_name}")

# Deploy to an endpoint
from vertexai.resources import Endpoint
endpoint = Endpoint.create(display_name="my-endpoint")
endpoint.deploy(
    model=model,
    deployed_model_display_name="my-model-deployed",
    machine_type="n1-standard-4",
    min_replica_count=1,
    max_replica_count=5,
    traffic_percentage=100,
)

# Online prediction
response = endpoint.predict(instances=[{"feature1": 1.0, "feature2": 2.0}])
print(response.predictions)

# Batch prediction
from vertexai.jobs import BatchPredictionJob
job = BatchPredictionJob.create(
    model_name=model.resource_name,
    job_display_name="batch-predict",
    gcs_source=["gs://my-bucket/input/data.jsonl"],
    gcs_destination_prefix="gs://my-bucket/output/",
    machine_type="n1-standard-4",
)
job.wait()
```

---

## Vertex AI Pipelines

```python
from kfp import dsl
from kfp.compiler import Compiler
import vertexai
from vertexai.pipeline_jobs import PipelineJob

@dsl.component(base_image="python:3.11", packages_to_install=["scikit-learn"])
def train(data_path: str, model_output: dsl.Output[dsl.Model]):
    import sklearn, joblib
    # ... training code ...
    joblib.dump(model, model_output.path)

@dsl.component(base_image="python:3.11", packages_to_install=["scikit-learn"])
def evaluate(model: dsl.Input[dsl.Model], metrics: dsl.Output[dsl.Metrics]):
    # ... evaluation code ...
    metrics.log_metric("accuracy", 0.95)

@dsl.pipeline(name="training-pipeline")
def my_pipeline(data_path: str = "gs://my-bucket/data/"):
    train_task = train(data_path=data_path)
    evaluate(model=train_task.outputs["model_output"])

# Compile
Compiler().compile(my_pipeline, "pipeline.yaml")

# Run
vertexai.init(project="my-project", location="us-central1")
job = PipelineJob(
    display_name="training-run",
    template_path="pipeline.yaml",
    pipeline_root="gs://my-bucket/pipeline-root/",
    parameter_values={"data_path": "gs://my-bucket/data/"},
)
job.run(sync=True)
```

---

## Document AI

```python
from google.cloud import documentai

client = documentai.DocumentProcessorServiceClient()

# Process a document
with open("invoice.pdf", "rb") as f:
    content = f.read()

processor_name = client.processor_path("my-project", "us", "PROCESSOR_ID")

request = documentai.ProcessRequest(
    name=processor_name,
    raw_document=documentai.RawDocument(content=content, mime_type="application/pdf"),
)

response = client.process_document(request=request)
document = response.document

# Extract text and entities
print(document.text)
for entity in document.entities:
    print(f"{entity.type_}: {entity.mention_text} (confidence: {entity.confidence:.2f})")
```

```bash
# List processor types
gcloud ai document-processors list --location=us --project=my-project

# Create a processor
gcloud ai document-processors create \
  --location=us \
  --display-name="Invoice Parser" \
  --type=INVOICE_PROCESSOR \
  --project=my-project
```

---

## Vision AI

```python
from google.cloud import vision

client = vision.ImageAnnotatorClient()

# Detect labels
with open("image.jpg", "rb") as f:
    content = f.read()

image = vision.Image(content=content)

# Label detection
response = client.label_detection(image=image)
for label in response.label_annotations:
    print(f"{label.description}: {label.score:.2f}")

# Text detection (OCR)
response = client.text_detection(image=image)
print(response.full_text_annotation.text)

# Object detection
response = client.object_localization(image=image)
for obj in response.localized_object_annotations:
    print(f"{obj.name}: {obj.score:.2f}")

# Safe search (content moderation)
response = client.safe_search_detection(image=image)
safe = response.safe_search_annotation
print(f"Adult: {safe.adult}, Violence: {safe.violence}")
```

---

## Natural Language AI

```python
from google.cloud import language_v1

client = language_v1.LanguageServiceClient()

text = "Google Cloud is excellent for building AI applications."
document = language_v1.Document(content=text, type_=language_v1.Document.Type.PLAIN_TEXT)

# Sentiment analysis
response = client.analyze_sentiment(request={"document": document})
print(f"Sentiment: {response.document_sentiment.score:.2f} (magnitude: {response.document_sentiment.magnitude:.2f})")

# Entity recognition
response = client.analyze_entities(request={"document": document})
for entity in response.entities:
    print(f"{entity.name} ({entity.type_.name}): salience={entity.salience:.2f}")

# Classification
response = client.classify_text(request={"document": document})
for category in response.categories:
    print(f"{category.name}: {category.confidence:.2f}")
```

---

## Vertex AI Search (formerly Enterprise Search)

```bash
# Create a search app (data store + search engine)
gcloud alpha discovery-engine engines create my-search-engine \
  --location=global \
  --display-name="My Search Engine" \
  --project=my-project

# Import documents from GCS
gcloud alpha discovery-engine documents import \
  --data-store=my-datastore \
  --location=global \
  --gcs-source=gs://my-bucket/documents/*.json
```

---

## Guardrails

- **Vertex AI endpoints bill per node-hour** — always set `min-replica-count=0` for dev/test endpoints and `max-replica-count` limits for prod.
- **Gemini API has token limits** — use `max_output_tokens` to prevent runaway generation and cost.
- **Never include PII in training data** unless you've reviewed data residency and compliance requirements.
- **Use `us-central1` for broadest model and GPU availability** — not all regions support all models.
- **Custom training jobs don't auto-stop on failure** — set `--max-running-time` to avoid runaway costs.
- **Model Registry versions are immutable** — create a new version rather than overwriting.
- **Batch prediction output can be large** — monitor GCS egress and BigQuery storage costs.
- **Test Vertex AI Pipelines with small data first** — pipeline bugs can spin up expensive VMs.
