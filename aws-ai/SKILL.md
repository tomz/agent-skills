---
name: aws-ai
description: AWS AI/ML services — Bedrock, SageMaker, Comprehend, Rekognition, Textract, Lex, Polly, Transcribe, Kendra, Q Developer
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

# AWS AI/ML Services — Comprehensive Reference

## Amazon Bedrock

### Model Access

```bash
# List available foundation models
aws bedrock list-foundation-models \
  --query 'modelSummaries[].[modelId,providerName,modelName]' \
  --output table --region us-east-1

# Request access to a model (done via console, but can check)
aws bedrock get-foundation-model --model-identifier anthropic.claude-3-5-sonnet-20241022-v2:0

# List models you have access to
aws bedrock list-foundation-models \
  --by-inference-type ON_DEMAND \
  --query 'modelSummaries[].modelId' --output text
```

### Invoke Model (Converse API — preferred)

```python
import boto3, json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

response = bedrock.converse(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    messages=[{'role': 'user', 'content': [{'text': 'Explain VPCs in 3 sentences.'}]}],
    system=[{'text': 'You are a helpful AWS expert.'}],
    inferenceConfig={'maxTokens': 512, 'temperature': 0.7, 'topP': 0.9},
)
print(response['output']['message']['content'][0]['text'])
print(f"Input tokens: {response['usage']['inputTokens']}, Output: {response['usage']['outputTokens']}")
```

### Streaming

```python
response = bedrock.converse_stream(
    modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
    messages=[{'role': 'user', 'content': [{'text': 'Write a Python quicksort.'}]}],
)
for event in response['stream']:
    if 'contentBlockDelta' in event:
        print(event['contentBlockDelta']['delta']['text'], end='', flush=True)
```

### CLI Invocation

```bash
aws bedrock-runtime converse \
  --model-id anthropic.claude-3-haiku-20240307-v1:0 \
  --messages '[{"role":"user","content":[{"text":"What is S3?"}]}]' \
  --query 'output.message.content[0].text' --output text
```

### Bedrock Agents

```bash
# Create agent
aws bedrock create-agent \
  --agent-name my-ops-agent \
  --foundation-model anthropic.claude-3-5-sonnet-20241022-v2:0 \
  --instruction "You are an AWS operations assistant. Help users manage their cloud resources." \
  --agent-resource-role-arn arn:aws:iam::123:role/BedrockAgentRole

# Create action group (connects agent to Lambda)
aws bedrock create-agent-action-group \
  --agent-id <agent-id> --agent-version DRAFT \
  --action-group-name AwsOps \
  --action-group-executor lambda=arn:aws:lambda:us-east-1:123:function:aws-ops \
  --api-schema payload=file://openapi-schema.json

# Prepare and invoke
aws bedrock prepare-agent --agent-id <agent-id>
aws bedrock-runtime invoke-agent \
  --agent-id <agent-id> --agent-alias-id TSTALIASID \
  --session-id sess-$(date +%s) \
  --input-text "List all running EC2 instances in us-east-1" \
  outfile.txt
```

### Knowledge Bases (RAG)

```bash
# Create knowledge base with S3 data source
aws bedrock create-knowledge-base \
  --name prod-kb \
  --role-arn arn:aws:iam::123:role/BedrockKBRole \
  --knowledge-base-configuration type=VECTOR,vectorKnowledgeBaseConfiguration={embeddingModelArn=arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0} \
  --storage-configuration type=OPENSEARCH_SERVERLESS,opensearchServerlessConfiguration={...}

# Sync data source
aws bedrock start-ingestion-job --knowledge-base-id <kb-id> --data-source-id <ds-id>

# Retrieve + generate
aws bedrock-agent-runtime retrieve-and-generate \
  --input text="What is our incident response process?" \
  --retrieve-and-generate-configuration '{
    "type": "KNOWLEDGE_BASE",
    "knowledgeBaseConfiguration": {
      "knowledgeBaseId": "<kb-id>",
      "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-20240307-v1:0"
    }
  }' \
  --query 'output.text' --output text
```

### Bedrock Guardrails

```bash
aws bedrock create-guardrail \
  --name content-filter \
  --content-policy-config '{
    "filtersConfig": [
      {"type": "SEXUAL", "inputStrength": "HIGH", "outputStrength": "HIGH"},
      {"type": "VIOLENCE", "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
      {"type": "HATE", "inputStrength": "HIGH", "outputStrength": "HIGH"}
    ]
  }' \
  --topic-policy-config '{
    "topicsConfig": [{
      "name": "FinancialAdvice",
      "definition": "Providing specific investment or financial advice",
      "type": "DENY"
    }]
  }' \
  --word-policy-config '{"wordsConfig": [{"text": "competitor"}], "managedWordListsConfig": [{"type": "PROFANITY"}]}'
```

---

## SageMaker

```bash
# Create notebook instance
aws sagemaker create-notebook-instance \
  --notebook-instance-name ml-experiments \
  --instance-type ml.t3.medium \
  --role-arn arn:aws:iam::123:role/SageMakerRole \
  --default-code-repository https://github.com/myorg/ml-notebooks

# Training job
aws sagemaker create-training-job \
  --training-job-name my-training-$(date +%s) \
  --algorithm-specification TrainingImage=763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:2.1.0-gpu-py310,TrainingInputMode=File \
  --role-arn arn:aws:iam::123:role/SageMakerRole \
  --input-data-config '[{"ChannelName":"training","DataSource":{"S3DataSource":{"S3Uri":"s3://my-bucket/data/train","S3DataType":"S3Prefix"}}}]' \
  --output-data-config S3OutputPath=s3://my-bucket/models/ \
  --resource-config InstanceType=ml.p3.2xlarge,InstanceCount=1,VolumeSizeInGB=50 \
  --stopping-condition MaxRuntimeInSeconds=3600 \
  --hyper-parameters lr=0.001,epochs=10,batch_size=32

# Wait for training
aws sagemaker wait training-job-completed-or-stopped \
  --training-job-name my-training-xxx

# Deploy endpoint
aws sagemaker create-model \
  --model-name my-model-v1 \
  --primary-container Image=763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-inference:2.1.0-cpu-py310,ModelDataUrl=s3://my-bucket/models/output/model.tar.gz \
  --execution-role-arn arn:aws:iam::123:role/SageMakerRole

aws sagemaker create-endpoint-config \
  --endpoint-config-name my-model-v1-config \
  --production-variants Name=primary,ModelName=my-model-v1,InstanceType=ml.m5.xlarge,InitialInstanceCount=1

aws sagemaker create-endpoint \
  --endpoint-name my-model-endpoint \
  --endpoint-config-name my-model-v1-config

# Invoke endpoint
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name my-model-endpoint \
  --content-type application/json \
  --body '{"inputs": "classify this text"}' \
  /tmp/prediction.json && cat /tmp/prediction.json
```

---

## Amazon Comprehend

```python
import boto3
comprehend = boto3.client('comprehend', region_name='us-east-1')

text = "AWS is headquartered in Seattle. Jeff Bezos founded Amazon in 1994."

# Detect entities
response = comprehend.detect_entities(Text=text, LanguageCode='en')
for entity in response['Entities']:
    print(f"{entity['Type']}: {entity['Text']} ({entity['Score']:.2f})")

# Detect sentiment
sentiment = comprehend.detect_sentiment(Text=text, LanguageCode='en')
print(sentiment['Sentiment'], sentiment['SentimentScore'])

# Key phrases
phrases = comprehend.detect_key_phrases(Text=text, LanguageCode='en')

# PII detection
pii = comprehend.detect_pii_entities(Text="SSN: 123-45-6789, Email: user@example.com", LanguageCode='en')
```

---

## Amazon Rekognition

```python
import boto3
rekognition = boto3.client('rekognition')

# Detect labels
with open('image.jpg', 'rb') as f:
    response = rekognition.detect_labels(
        Image={'Bytes': f.read()},
        MaxLabels=10, MinConfidence=80
    )
for label in response['Labels']:
    print(f"{label['Name']}: {label['Confidence']:.1f}%")

# Face detection
response = rekognition.detect_faces(
    Image={'S3Object': {'Bucket': 'my-bucket', 'Name': 'face.jpg'}},
    Attributes=['ALL']
)

# Content moderation
response = rekognition.detect_moderation_labels(
    Image={'S3Object': {'Bucket': 'my-bucket', 'Name': 'photo.jpg'}},
    MinConfidence=60
)

# Text in image (OCR)
response = rekognition.detect_text(
    Image={'S3Object': {'Bucket': 'my-bucket', 'Name': 'sign.jpg'}}
)
```

---

## Amazon Textract (Document OCR + Structure)

```python
import boto3
textract = boto3.client('textract')

# Synchronous (single page / small doc)
with open('form.pdf', 'rb') as f:
    response = textract.analyze_document(
        Document={'Bytes': f.read()},
        FeatureTypes=['FORMS', 'TABLES']
    )

# Async (multi-page PDFs)
response = textract.start_document_analysis(
    DocumentLocation={'S3Object': {'Bucket': 'my-bucket', 'Name': 'report.pdf'}},
    FeatureTypes=['FORMS', 'TABLES'],
    NotificationChannel={'SNSTopicArn': '...', 'RoleArn': '...'},
    OutputConfig={'S3Bucket': 'my-results-bucket'}
)
job_id = response['JobId']
```

---

## Amazon Transcribe

```python
import boto3, time
transcribe = boto3.client('transcribe')

# Start job
transcribe.start_transcription_job(
    TranscriptionJobName='meeting-2024-01-15',
    Media={'MediaFileUri': 's3://my-bucket/audio/meeting.mp4'},
    MediaFormat='mp4',
    LanguageCode='en-US',
    Settings={'ShowSpeakerLabels': True, 'MaxSpeakerLabels': 4}
)

# Poll for completion
while True:
    status = transcribe.get_transcription_job(TranscriptionJobName='meeting-2024-01-15')
    if status['TranscriptionJob']['TranscriptionJobStatus'] in ['COMPLETED', 'FAILED']:
        break
    time.sleep(10)

print(status['TranscriptionJob']['Transcript']['TranscriptFileUri'])
```

---

## Amazon Polly (Text to Speech)

```python
import boto3
polly = boto3.client('polly')

response = polly.synthesize_speech(
    Text='<speak>Hello! <break time="1s"/> Welcome to <emphasis>AWS</emphasis>.</speak>',
    TextType='ssml',
    OutputFormat='mp3',
    VoiceId='Joanna',
    Engine='neural'    # neural = higher quality
)

with open('output.mp3', 'wb') as f:
    f.write(response['AudioStream'].read())
```

---

## Amazon Kendra

```bash
# Create index
aws kendra create-index \
  --name prod-search \
  --edition DEVELOPER_EDITION \
  --role-arn arn:aws:iam::123:role/KendraRole

# Add S3 data source
aws kendra create-data-source \
  --index-id <index-id> \
  --name docs-s3 \
  --type S3 \
  --configuration S3Configuration={BucketName=my-docs-bucket} \
  --role-arn arn:aws:iam::123:role/KendraS3Role

aws kendra start-data-source-sync-job --index-id <index-id> --id <source-id>

# Query
aws kendra query \
  --index-id <index-id> \
  --query-text "How do I configure VPC peering?" \
  --query 'ResultItems[].[DocumentTitle.Text,DocumentExcerpt.Text]' --output table
```

---

## Amazon Q Developer

Q Developer is an AI coding assistant integrated into IDEs and CLI. It uses your codebase context.

```bash
# Install Q CLI
pip install amazon-q-developer-cli   # or via AWS Toolkit in VS Code / JetBrains

# CLI usage
q chat "explain this CloudFormation template"
q chat --file template.yaml "what security issues does this have?"

# Inline code suggestions: available in VS Code / JetBrains via AWS Toolkit extension
# Enable: Settings → AWS → Amazon Q → Enable Q Developer

# Q for command line (chat in terminal)
q --context . "write a Python Lambda that reads from SQS and writes to DynamoDB"
```

---

## Guardrails & Gotchas

- **Bedrock model availability** — not all models available in all regions. Check console for your region. Cross-region inference available via inference profiles.
- **Bedrock costs** — on-demand pricing per token. High-volume workloads should use Provisioned Throughput (commit to model units/month).
- **Bedrock Agents Lambda** — your Lambda must return a specific response schema matching the OpenAPI spec. Test with `InvokeAgent` before wiring to prod.
- **SageMaker endpoints never scale to zero** — use Serverless Inference (cold start) or Lambda + batch transform for sporadic workloads.
- **Rekognition streaming** — video analysis uses Kinesis Video Streams, not S3. Different API path.
- **Textract limits** — sync API max 10MB/3000 pages; use async + S3 for anything bigger.
- **Comprehend batch** — single API calls max 25 documents. Use `start-entities-detection-job` for bulk.
- **Knowledge Base embeddings** — changing the embedding model requires re-ingesting all documents. Plan ahead.
- **Q Developer data privacy** — by default, Q may use your code to improve models. Enable "professional tier" for enterprise data privacy guarantees.
