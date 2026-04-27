---
name: oci-ai
description: OCI AI services — Generative AI, Data Science, AI Language, AI Vision, AI Speech, Anomaly Detection, Digital Assistant
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

# OCI AI Services

## OCI Generative AI Service

OCI Gen AI provides hosted inference for large language models (Cohere and Meta models) with options for fine-tuning and dedicated clusters.

### Available Models (as of 2026)
| Model ID | Provider | Use Case |
|----------|----------|----------|
| `cohere.command-r-plus` | Cohere | Chat, RAG, complex reasoning |
| `cohere.command-r-08-2024` | Cohere | Chat, balanced performance/cost |
| `cohere.embed-multilingual-v3.0` | Cohere | Embeddings (multilingual) |
| `cohere.embed-english-v3.0` | Cohere | Embeddings (English-optimized) |
| `meta.llama-3.1-405b-instruct` | Meta | Large-scale reasoning |
| `meta.llama-3.1-70b-instruct` | Meta | General chat/generation |
| `meta.llama-3.2-90b-vision-instruct` | Meta | Vision + text |

### Chat / Text Generation

```bash
# List available models
oci generative-ai model list --compartment-id $C --all --output table

# Chat completion via CLI
oci generative-ai-inference chat \
  --compartment-id $C \
  --chat-request '{
    "apiFormat": "COHERE",
    "chatHistory": [],
    "message": "Explain OCI compartments in 2 sentences.",
    "servingMode": {
      "modelId": "cohere.command-r-plus",
      "servingType": "ON_DEMAND"
    },
    "maxTokens": 500,
    "temperature": 0.7
  }'

# Llama (uses GENERIC format)
oci generative-ai-inference generate-text \
  --compartment-id $C \
  --generate-text-detail '{
    "servingMode": {
      "modelId": "meta.llama-3.1-70b-instruct",
      "servingType": "ON_DEMAND"
    },
    "compartmentId": "'$C'",
    "inferenceRequest": {
      "apiFormat": "GENERIC",
      "prompt": "<|begin_of_text|><|start_header_id|>user<|end_header_id|>\nHello!<|eot_id|><|start_header_id|>assistant<|end_header_id|>",
      "maxTokens": 500,
      "temperature": 0.7
    }
  }'
```

### Python SDK — Chat
```python
import oci

config = oci.config.from_file()
client = oci.generative_ai_inference.GenerativeAiInferenceClient(
    config,
    service_endpoint="https://inference.generativeai.us-chicago-1.oci.oraclecloud.com"
)

chat_detail = oci.generative_ai_inference.models.CohereChatRequest(
    message="What are OCI security zones?",
    chat_history=[],
    max_tokens=1024,
    temperature=0.7,
    is_stream=False,
)

response = client.chat(
    chat_details=oci.generative_ai_inference.models.ChatDetails(
        serving_mode=oci.generative_ai_inference.models.OnDemandServingMode(
            model_id="cohere.command-r-plus"
        ),
        compartment_id=COMPARTMENT_ID,
        chat_request=chat_detail,
    )
)
print(response.data.chat_response.text)
```

### Python SDK — Embeddings
```python
embed_detail = oci.generative_ai_inference.models.EmbedTextDetails(
    inputs=["Oracle Cloud", "Machine Learning", "OCI Gen AI"],
    serving_mode=oci.generative_ai_inference.models.OnDemandServingMode(
        model_id="cohere.embed-english-v3.0"
    ),
    compartment_id=COMPARTMENT_ID,
    truncate="NONE",
    input_type="SEARCH_DOCUMENT",  # or SEARCH_QUERY, CLASSIFICATION, CLUSTERING
)
response = client.embed_text(embed_text_details=embed_detail)
vectors = response.data.embeddings   # list of float lists
```

### OpenAI-Compatible Endpoint
```python
from openai import OpenAI

client = OpenAI(
    api_key="<oci-api-key>",   # actual auth uses SDK signing
    base_url="https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/chat/completions"
)
# Note: OCI Gen AI does NOT fully implement OpenAI API — prefer native SDK
```

### Fine-Tuning
```bash
# Create a fine-tuning dedicated cluster (minimum commitment)
oci generative-ai dedicated-ai-cluster create \
  --compartment-id $C \
  --display-name "ft-cluster" \
  --type "FINE_TUNING" \
  --unit-count 1 \
  --unit-shape "LARGE_COHERE"

# Create fine-tuning job
oci generative-ai model create \
  --compartment-id $C \
  --display-name "my-custom-model" \
  --base-model-id "cohere.command-r-plus" \
  --fine-tune-details '{
    "dedicatedAiClusterId": "'$CLUSTER_ID'",
    "trainingDataset": {
      "datasetType": "OBJECT_STORAGE",
      "bucket": "ft-training-data",
      "namespace": "'$NAMESPACE'",
      "object": "training.jsonl"
    },
    "totalTrainingEpochs": 3,
    "learningRateScale": 0.01
  }'
```

Training data format (JSONL):
```jsonl
{"prompt": "What is OCI?", "completion": "Oracle Cloud Infrastructure (OCI) is Oracle's cloud platform..."}
{"prompt": "How do I create a VCN?", "completion": "To create a VCN, navigate to..."}
```

## OCI Data Science

### Notebooks
```bash
# Create Data Science project
DS_PROJECT_ID=$(oci data-science project create \
  --compartment-id $C \
  --display-name "ml-project" \
  --query 'data.id' --raw-output)

# Create notebook session
oci data-science notebook-session create \
  --compartment-id $C \
  --project-id $DS_PROJECT_ID \
  --display-name "dev-notebook" \
  --notebook-session-config-details '{
    "shape": "VM.Standard.E4.Flex",
    "ocpus": 4,
    "memoryInGBs": 32,
    "blockStorageSizeInGBs": 200
  }' \
  --notebook-session-runtime-config-details '{
    "notebookSessionLifecycleConfigDetails": {
      "notebookSessionLifecycleConfigType": "NO_LCC"
    }
  }'

# Get notebook URL
oci data-science notebook-session get \
  --notebook-session-id $NS_ID \
  --query 'data."notebook-session-url"' --raw-output
```

### Model Catalog & Deployment
```bash
# Register model in catalog
MODEL_ID=$(oci data-science model create \
  --compartment-id $C \
  --project-id $DS_PROJECT_ID \
  --display-name "xgboost-churn-v1" \
  --artifact-content-length 5242880 \
  --model-artifact ./model_artifact.zip \
  --query 'data.id' --raw-output)

# Create model deployment (REST endpoint)
oci data-science model-deployment create \
  --compartment-id $C \
  --project-id $DS_PROJECT_ID \
  --display-name "churn-predictor" \
  --model-deployment-configuration-details '{
    "deploymentType": "SINGLE_MODEL",
    "modelConfigurationDetails": {
      "modelId": "'$MODEL_ID'",
      "instanceConfiguration": {
        "instanceShapeName": "VM.Standard.E4.Flex",
        "modelDeploymentInstanceShapeConfigDetails": {"ocpus":2,"memoryInGBs":16}
      },
      "scalingPolicy": {
        "policyType": "FIXED_SIZE",
        "instanceCount": 1
      }
    }
  }'

# Invoke deployed model
MODEL_ENDPOINT=$(oci data-science model-deployment get \
  --model-deployment-id $MD_ID \
  --query 'data."model-deployment-url"' --raw-output)

curl -X POST "${MODEL_ENDPOINT}/predict" \
  -H "Authorization: Bearer $(oci iam auth-token ...)" \
  -H "Content-Type: application/json" \
  -d '{"features": [[0.5, 1.2, 0.8, 3.1]]}'
```

### Data Science Jobs (batch)
```bash
oci data-science job create \
  --compartment-id $C \
  --project-id $DS_PROJECT_ID \
  --display-name "training-job" \
  --job-infrastructure-configuration-details '{
    "jobInfrastructureType": "STANDALONE",
    "shapeName": "VM.Standard.E4.Flex",
    "jobShapeConfigDetails": {"ocpus":8,"memoryInGBs":64},
    "blockStorageSizeInGBs": 500
  }' \
  --job-artifact ./train.py

oci data-science job-run create \
  --compartment-id $C \
  --job-id $JOB_ID \
  --display-name "run-$(date +%Y%m%d)"
```

## AI Language

```bash
# Detect language
oci ai language batch-detect-dominant-language \
  --compartment-id $C \
  --documents '[{"key":"doc1","text":"Hello, how are you today?"},{"key":"doc2","text":"Bonjour, comment allez-vous?"}]'

# Sentiment analysis
oci ai language batch-detect-language-sentiments \
  --compartment-id $C \
  --documents '[{"key":"1","text":"The product is amazing! Great service.","languageCode":"en"}]'

# Named entity recognition
oci ai language batch-detect-language-entities \
  --compartment-id $C \
  --documents '[{"key":"1","text":"Oracle Corporation is headquartered in Austin, Texas.","languageCode":"en"}]'

# Key phrase extraction
oci ai language batch-detect-language-key-phrases \
  --compartment-id $C \
  --documents '[{"key":"1","text":"Machine learning and AI are transforming cloud computing.","languageCode":"en"}]'

# Text classification (with custom model)
oci ai language batch-detect-language-text-classification \
  --compartment-id $C \
  --documents '[{"key":"1","text":"I want to cancel my subscription.","languageCode":"en"}]' \
  --endpoint-id $CUSTOM_ENDPOINT_ID   # omit for pretrained
```

## AI Vision

```bash
# Analyze image (object detection, labels, text)
oci ai-vision image-job create \
  --compartment-id $C \
  --features '[{"featureType":"IMAGE_CLASSIFICATION"},{"featureType":"OBJECT_DETECTION"},{"featureType":"TEXT_DETECTION"}]' \
  --input-location '{
    "sourceType": "OBJECT_STORAGE_LOCATIONS",
    "objectLocations": [{"bucketName":"images","namespaceName":"'$NAMESPACE'","objectName":"photo.jpg"}]
  }' \
  --output-location '{"bucketName":"results","namespaceName":"'$NAMESPACE'","prefix":"vision-output/"}'

# Document analysis (structured extraction)
oci ai-vision document-job create \
  --compartment-id $C \
  --features '[{"featureType":"DOCUMENT_CLASSIFICATION"},{"featureType":"TEXT_EXTRACTION"},{"featureType":"TABLE_EXTRACTION"}]' \
  --input-location '{"sourceType":"OBJECT_STORAGE_LOCATIONS","objectLocations":[{"bucketName":"docs","namespaceName":"'$NAMESPACE'","objectName":"invoice.pdf"}]}' \
  --output-location '{"bucketName":"results","namespaceName":"'$NAMESPACE'","prefix":"doc-output/"}'
```

## AI Speech (STT)

```bash
# Transcribe audio file
oci ai-speech transcription-job create \
  --compartment-id $C \
  --display-name "meeting-transcript" \
  --input-location '{
    "locationType": "OBJECT_LIST_INLINE_INPUT_LOCATION",
    "objectLocations": [{"bucketName":"audio","namespaceName":"'$NAMESPACE'","objectNames":["meeting.mp3"]}]
  }' \
  --output-location '{"bucketName":"transcripts","namespaceName":"'$NAMESPACE'","prefix":"output/"}' \
  --model-details '{"modelType":"WHISPER_MEDIUM","domain":"GENERIC","languageCode":"en-US"}'

# Real-time streaming STT (via WebSocket SDK)
# Install: pip install oci-ai-speech-realtime
python3 -c "
from oci.ai_speech_realtime import RealtimeClient, RealtimeParameters
# See OCI docs for full streaming example
"
```

## AI Anomaly Detection

```bash
# Create project
oci anomaly-detection project create \
  --compartment-id $C \
  --display-name "sensor-anomaly"

# Create data asset (training data from Object Storage)
oci anomaly-detection data-asset create \
  --compartment-id $C \
  --project-id $PROJECT_ID \
  --display-name "sensor-training-data" \
  --data-source-details '{
    "dataSourceType": "ORACLE_OBJECT_STORAGE",
    "bucketName": "ml-data",
    "namespace": "'$NAMESPACE'",
    "objectName": "training_data.csv"
  }'

# Train model
oci anomaly-detection model create \
  --compartment-id $C \
  --project-id $PROJECT_ID \
  --display-name "sensor-model-v1" \
  --model-training-details '{
    "dataAssetIds": ["'$DATA_ASSET_ID'"],
    "targetFap": 0.01,
    "trainFractionPerStep": 0.65,
    "windowSize": 1
  }'

# Detect anomalies
oci anomaly-detection model detect-anomalies \
  --model-id $MODEL_ID \
  --compartment-id $C \
  --request-type "INLINE" \
  --inline-request-payload '{"requestType":"INLINE","signalNames":["temperature","pressure"],"data":[{"timestamp":"2024-01-01T00:00:00Z","values":[{"metadataId":0,"value":98.5},{"metadataId":1,"value":14.7}]}]}'
```

## Gotchas & Tips

- **Gen AI region** — OCI Generative AI is available only in specific regions: `us-chicago-1` (US), `eu-frankfurt-1` (EU), `ap-osaka-1` (APAC). Ensure your endpoint matches your region.
- **Dedicated AI clusters** — required for fine-tuned models. They have a minimum commitment (1 unit). Stop/start to avoid idle charges. Cannot be used for ON_DEMAND models.
- **Embedding dimensions** — Cohere embed-v3 produces 1024-dimensional vectors. Pre-allocate your vector store accordingly (e.g., OpenSearch k-NN, or OCI OpenSearch with vector field).
- **Data Science conda environments** — published conda environments on OCI (`oci://service-conda-envs@ociodscdev`) are pre-built with ML frameworks. Use `odsc conda install -s generalml_p38_cpu_v1` in notebooks.
- **Model deployment scaling** — `FIXED_SIZE` policy for predictable load; `AUTOSCALING` for variable. Cold-start on scale-out takes ~2 minutes for typical models.
- **AI Vision quotas** — default 5 requests/second for inline analysis. Batch jobs via Object Storage bypass rate limits.
- **Language detection minimum text** — AI Language NER/sentiment requires at least 100 characters for reliable results. Short texts get lower confidence scores.
- **Anomaly Detection training data** — needs ≥ 5,000 timestamps with no gaps. Fill gaps with interpolated values or the model training will fail.
