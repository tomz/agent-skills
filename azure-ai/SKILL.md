---
name: azure-ai
description: Azure OpenAI Service, Azure AI Search, Azure AI Services (Cognitive Services), Azure Machine Learning — deployments, patterns, content filtering, RAG, compute
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

# Azure AI Skills

## Azure OpenAI Service

### Setup
```bash
# Create Azure OpenAI resource
az cognitiveservices account create \
  --name myAOAI \
  --resource-group myRG \
  --location eastus \
  --kind OpenAI \
  --sku S0 \
  --custom-domain myaoai-unique

# List available models in a region
az cognitiveservices account list-models \
  --name myAOAI \
  --resource-group myRG \
  --output table

# Get endpoint and keys
az cognitiveservices account show \
  --name myAOAI \
  --resource-group myRG \
  --query properties.endpoint -o tsv

az cognitiveservices account keys list \
  --name myAOAI \
  --resource-group myRG \
  --query key1 -o tsv
```

### Model Deployments
```bash
# Deploy a model
az cognitiveservices account deployment create \
  --name myAOAI \
  --resource-group myRG \
  --deployment-name gpt-4o \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-capacity 100 \
  --sku-name GlobalStandard   # GlobalStandard, Standard, DataZoneStandard, ProvisionedManaged

# List deployments
az cognitiveservices account deployment list \
  --name myAOAI \
  --resource-group myRG \
  --output table

# Delete a deployment
az cognitiveservices account deployment delete \
  --name myAOAI \
  --resource-group myRG \
  --deployment-name gpt-4o
```

### Deployment SKU Types
```
GlobalStandard       → Routes globally for best availability; TPM-based billing; default for most
Standard             → Region-locked; predictable latency; TPM-based billing
DataZoneStandard     → Data stays in geographic zone (EU, US); compliance requirement
GlobalBatch          → Asynchronous batch; 50% cost discount; 24h SLA
ProvisionedManaged   → PTU (Provisioned Throughput Units); reserved capacity; predictable throughput
```

### PTU vs Pay-as-you-go
```
Pay-as-you-go (Standard/GlobalStandard):
  + No commitment
  + Auto-scales with demand
  - Subject to rate limiting (TPM/RPM quotas)
  - Variable cost at high volume
  - Noisy neighbor risk

Provisioned Throughput Units (PTU):
  + Guaranteed throughput (no rate limits)
  + Predictable latency
  + Better for consistent high-volume workloads
  - Requires 1-month+ commitment
  - More expensive at low utilization
  - Must purchase in unit increments (e.g., 25 PTU minimum)

Rule of thumb: Use PTU when utilization > 60% of a PTU unit consistently
```

### Using the API (Python)
```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="your-key",                           # or use DefaultAzureCredential
    api_version="2024-10-21",                     # use latest GA version
    azure_endpoint="https://myaoai-unique.openai.azure.com"
)

# Chat completion
response = client.chat.completions.create(
    model="gpt-4o",          # deployment name, not model name
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Azure OpenAI?"}
    ],
    max_tokens=1000,
    temperature=0.7,
    stream=True              # streaming response
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)

# With Managed Identity (no key needed)
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default")

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    api_version="2024-10-21",
    azure_endpoint="https://myaoai-unique.openai.azure.com"
)
```

### Embeddings
```python
# Generate embeddings (for RAG, semantic search)
response = client.embeddings.create(
    model="text-embedding-3-large",    # deployment name
    input=["First document", "Second document"],
    dimensions=1536                     # reduce dimensions for efficiency
)

embeddings = [item.embedding for item in response.data]
```

### Content Filtering
```
Default content filters (cannot be turned off):
  - Hate speech: Medium threshold
  - Sexual content: Medium threshold
  - Violence: Medium threshold
  - Self-harm: Medium threshold

Configurable (requires approval):
  - Jailbreak detection
  - Protected material (text/code)
  - Groundedness detection (for RAG)
  - Prompt shields (indirect injection attacks)

Custom filter policies: Create in Azure AI Foundry portal
Applied per deployment — one policy per deployment
```

### DALL-E / Image Generation
```python
# Generate image
response = client.images.generate(
    model="dall-e-3",           # deployment name
    prompt="A serene mountain lake at sunset",
    size="1792x1024",           # 1024x1024, 1024x1792, 1792x1024
    quality="hd",               # standard or hd
    style="natural",            # vivid or natural
    n=1
)

image_url = response.data[0].url
```

## Azure AI Search (formerly Cognitive Search)

```bash
# Create Search service
az search service create \
  --name mysearch-unique \
  --resource-group myRG \
  --sku standard \
  --location eastus \
  --partition-count 1 \
  --replica-count 3   # 3 replicas for HA (99.9% SLA)

# Get admin key
az search admin-key show \
  --resource-group myRG \
  --service-name mysearch-unique \
  --query primaryKey -o tsv

# SKUs: free (1 index, 50MB), basic, standard (S1/S2/S3), storage-optimized (L1/L2)
```

### Index Operations (REST API)
```bash
# Create index
curl -X PUT \
  "https://mysearch-unique.search.windows.net/indexes/myindex?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: $SEARCH_ADMIN_KEY" \
  -d '{
    "name": "myindex",
    "fields": [
      {"name": "id", "type": "Edm.String", "key": true, "retrievable": true},
      {"name": "content", "type": "Edm.String", "searchable": true, "retrievable": true, "analyzer": "en.microsoft"},
      {"name": "title", "type": "Edm.String", "searchable": true, "filterable": true, "sortable": true},
      {"name": "category", "type": "Edm.String", "filterable": true, "facetable": true},
      {"name": "embedding", "type": "Collection(Edm.Single)", "dimensions": 1536, "vectorSearchProfile": "myVectorProfile", "searchable": true}
    ],
    "vectorSearch": {
      "algorithms": [{"name": "myHnsw", "kind": "hnsw", "hnswParameters": {"metric": "cosine"}}],
      "profiles": [{"name": "myVectorProfile", "algorithm": "myHnsw"}]
    },
    "semantic": {
      "configurations": [{
        "name": "default",
        "prioritizedFields": {
          "contentFields": [{"fieldName": "content"}],
          "titleField": {"fieldName": "title"}
        }
      }]
    }
  }'
```

### Hybrid Search (Vector + Keyword)
```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.core.credentials import AzureKeyCredential

client = SearchClient(
    endpoint="https://mysearch-unique.search.windows.net",
    index_name="myindex",
    credential=AzureKeyCredential(os.environ["SEARCH_ADMIN_KEY"])
)

# Generate query embedding
query = "How do I configure network security groups?"
query_embedding = get_embedding(query)  # using Azure OpenAI embeddings

# Hybrid search: vector + keyword + semantic reranking
results = client.search(
    search_text=query,
    vector_queries=[VectorizedQuery(
        vector=query_embedding,
        k_nearest_neighbors=50,
        fields="embedding"
    )],
    query_type="semantic",
    semantic_configuration_name="default",
    select=["id", "title", "content"],
    top=5
)

for result in results:
    print(f"Score: {result['@search.score']:.4f} | {result['title']}")
    print(result['content'][:200])
```

### Skillsets (AI Enrichment Pipeline)
```bash
# Skillsets process documents during indexing via AI enrichment
# Common built-in skills:
# - EntityRecognitionSkill: extract people, organizations, locations
# - KeyPhraseExtractionSkill: extract key phrases
# - LanguageDetectionSkill: detect document language
# - OCRSkill: extract text from images
# - MergeSkill: combine text fields
# - SplitSkill: chunk text for vector search
# - AzureOpenAIEmbeddingSkill: generate embeddings during indexing
```

## Azure AI Services (Cognitive Services)

```bash
# Create multi-service account (access all AI services with one key)
az cognitiveservices account create \
  --name myAIServices \
  --resource-group myRG \
  --kind CognitiveServices \
  --sku S0 \
  --location eastus

# Or create single-service accounts:
az cognitiveservices account create --kind TextAnalytics --name myTA --sku S --resource-group myRG -l eastus
az cognitiveservices account create --kind ComputerVision --name myCV --sku S1 --resource-group myRG -l eastus
az cognitiveservices account create --kind SpeechServices --name mySpeech --sku S0 --resource-group myRG -l eastus
az cognitiveservices account create --kind FormRecognizer --name myDI --sku S0 --resource-group myRG -l eastus
```

### Language Service (Text Analytics)
```python
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

client = TextAnalyticsClient(
    endpoint="https://myAIServices.cognitiveservices.azure.com/",
    credential=AzureKeyCredential(key)
)

# Sentiment analysis
documents = ["I love Azure! The services are fantastic.", "This error is very frustrating."]
result = client.analyze_sentiment(documents=documents)
for doc in result:
    print(f"Sentiment: {doc.sentiment}, Confidence: {doc.confidence_scores}")

# Key phrase extraction
result = client.extract_key_phrases(documents=documents)
for doc in result:
    print(f"Key phrases: {doc.key_phrases}")

# Named entity recognition
result = client.recognize_entities(documents=["Microsoft Azure was founded in 2010."])
for doc in result:
    for entity in doc.entities:
        print(f"{entity.text} ({entity.category})")

# PII detection & redaction
result = client.recognize_pii_entities(documents=["Call me at 555-1234 or email@example.com"])
for doc in result:
    print(f"Redacted: {doc.redacted_text}")
```

### Document Intelligence (Form Recognizer)
```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.core.credentials import AzureKeyCredential

client = DocumentIntelligenceClient(
    endpoint="https://myDI.cognitiveservices.azure.com/",
    credential=AzureKeyCredential(key)
)

# Analyze invoice (prebuilt model)
with open("invoice.pdf", "rb") as f:
    poller = client.begin_analyze_document("prebuilt-invoice", body=f)
result = poller.result()

for invoice in result.documents:
    fields = invoice.fields
    print(f"Vendor: {fields.get('VendorName', {}).get('content')}")
    print(f"Total: {fields.get('InvoiceTotal', {}).get('content')}")

# Available prebuilt models: prebuilt-invoice, prebuilt-receipt,
# prebuilt-idDocument, prebuilt-businessCard, prebuilt-layout, prebuilt-read
```

### Computer Vision
```python
from azure.ai.vision.imageanalysis import ImageAnalysisClient
from azure.ai.vision.imageanalysis.models import VisualFeatures
from azure.core.credentials import AzureKeyCredential

client = ImageAnalysisClient(
    endpoint="https://myCV.cognitiveservices.azure.com/",
    credential=AzureKeyCredential(key)
)

result = client.analyze_from_url(
    image_url="https://example.com/image.jpg",
    visual_features=[
        VisualFeatures.CAPTION,
        VisualFeatures.OBJECTS,
        VisualFeatures.TAGS,
        VisualFeatures.READ   # OCR
    ]
)

print(f"Caption: {result.caption.text}")
for obj in result.objects.list:
    print(f"Object: {obj.tags[0].name} at {obj.bounding_box}")
```

## Azure Machine Learning

```bash
# Create AML workspace
az ml workspace create \
  --name myAMLWorkspace \
  --resource-group myRG \
  --location eastus \
  --storage-account mystorageacct \
  --key-vault myKV-unique \
  --application-insights myAppInsights

# Create compute cluster (for training)
az ml compute create \
  --name myTrainingCluster \
  --type AmlCompute \
  --workspace-name myAMLWorkspace \
  --resource-group myRG \
  --min-instances 0 \
  --max-instances 4 \
  --size Standard_NC6s_v3 \
  --idle-time-before-scale-down 120

# Create compute instance (for dev/notebooks)
az ml compute create \
  --name myDevInstance \
  --type ComputeInstance \
  --workspace-name myAMLWorkspace \
  --resource-group myRG \
  --size Standard_DS3_v2

# List environments
az ml environment list \
  --workspace-name myAMLWorkspace \
  --resource-group myRG \
  --output table

# Submit a training job
az ml job create \
  --file training-job.yml \
  --workspace-name myAMLWorkspace \
  --resource-group myRG

# training-job.yml example:
cat > training-job.yml << 'EOF'
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
type: command
code: ./src
command: python train.py --data ${{inputs.training_data}} --epochs 10
inputs:
  training_data:
    type: uri_folder
    path: azureml://datastores/workspaceblobstore/paths/data/training/
environment: azureml:AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest
compute: azureml:myTrainingCluster
experiment_name: my-experiment
display_name: sklearn-training-run
EOF

# Register a model
az ml model create \
  --name mymodel \
  --version 1 \
  --path ./model \
  --workspace-name myAMLWorkspace \
  --resource-group myRG

# Deploy model to online endpoint
az ml online-endpoint create \
  --name myendpoint \
  --workspace-name myAMLWorkspace \
  --resource-group myRG

az ml online-deployment create \
  --name mydeployment \
  --endpoint myendpoint \
  --workspace-name myAMLWorkspace \
  --resource-group myRG \
  --model azureml:mymodel:1 \
  --instance-type Standard_DS3_v2 \
  --instance-count 1

# Test endpoint
az ml online-endpoint invoke \
  --name myendpoint \
  --request-file test-request.json \
  --workspace-name myAMLWorkspace \
  --resource-group myRG
```

## RAG Pattern (Retrieval-Augmented Generation)

```python
# Complete RAG pipeline using Azure AI Search + Azure OpenAI
import os
from openai import AzureOpenAI
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.core.credentials import AzureKeyCredential

aoai_client = AzureOpenAI(
    api_key=os.environ["AOAI_KEY"],
    api_version="2024-10-21",
    azure_endpoint=os.environ["AOAI_ENDPOINT"]
)

search_client = SearchClient(
    endpoint=os.environ["SEARCH_ENDPOINT"],
    index_name="myindex",
    credential=AzureKeyCredential(os.environ["SEARCH_KEY"])
)

def get_embedding(text: str) -> list[float]:
    response = aoai_client.embeddings.create(
        model="text-embedding-3-large",
        input=text
    )
    return response.data[0].embedding

def retrieve_context(query: str, top_k: int = 5) -> str:
    embedding = get_embedding(query)
    results = search_client.search(
        search_text=query,
        vector_queries=[VectorizedQuery(
            vector=embedding, k_nearest_neighbors=top_k, fields="embedding"
        )],
        query_type="semantic",
        semantic_configuration_name="default",
        select=["title", "content"],
        top=top_k
    )
    context_parts = []
    for r in results:
        context_parts.append(f"[Source: {r['title']}]\n{r['content']}")
    return "\n\n---\n\n".join(context_parts)

def rag_chat(user_question: str) -> str:
    context = retrieve_context(user_question)
    messages = [
        {"role": "system", "content": f"""You are a helpful assistant. Answer the user's question
using ONLY the provided context. If the answer is not in the context, say so.

Context:
{context}"""},
        {"role": "user", "content": user_question}
    ]
    response = aoai_client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        max_tokens=1000,
        temperature=0
    )
    return response.choices[0].message.content
```

## Gotchas

- **Model availability by region**: Not all models are available in all regions. Check Azure OpenAI model availability page. eastus and westus have broadest selection.
- **Quota per deployment, not per resource**: Each deployment has its own TPM quota. Create multiple deployments of the same model for higher combined throughput.
- **`api_version` matters**: Use latest GA version (`2024-10-21` as of late 2024). Preview API versions add features but may change.
- **Azure AI Search `key` field**: Must be a string, retrievable, and not searchable. Only one key field per index.
- **Vector search dimensions**: Must match the embedding model's output dimensions. Mismatch causes index creation failure.
- **Content filter 429 errors**: Triggering content filters returns 400 (content blocked) not 429 (rate limit). Handle both error codes.
- **AML compute min-instances=0**: Scale to zero when idle to save cost. First job after idle has cold-start delay (~2-3 min).
- **Document Intelligence version**: Use `2024-02-29-preview` or later for the unified `prebuilt-invoice` model.
- **Azure AI Search semantic ranking**: Requires Standard tier or above. Not available on free tier.
- **PTU utilization monitoring**: Monitor `AzureOpenAIProvisionedManagedUtilizationV2` metric to ensure PTU isn't under-utilized (wasted spend) or over-subscribed (throttling).
