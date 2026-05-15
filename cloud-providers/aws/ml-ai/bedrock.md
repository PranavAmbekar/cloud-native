# Amazon Bedrock

> Fully managed service to build generative AI applications using foundation models.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Foundation Model (FM) | Large pre-trained AI model |
| Model Provider | Company providing the model (Anthropic, Meta, etc.) |
| Inference | Running the model to get outputs |
| Provisioned Throughput | Reserved model capacity |
| Knowledge Base | RAG with your data |
| Agent | Autonomous AI that can take actions |
| Guardrails | Safety and content filters |

---

## Available Models

| Provider | Models | Strengths |
|----------|--------|-----------|
| **Anthropic** | Claude 3.5 Sonnet, Claude 3 Opus/Haiku | Reasoning, coding, analysis |
| **Meta** | Llama 3.1 (8B, 70B, 405B) | Open weights, customizable |
| **Mistral** | Mistral Large, Mixtral | Efficient, multilingual |
| **Amazon** | Titan Text, Titan Embeddings | AWS native, embeddings |
| **Cohere** | Command R/R+, Embed | Enterprise, RAG |
| **AI21** | Jamba, Jurassic | Text generation |
| **Stability AI** | Stable Diffusion | Image generation |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Amazon Bedrock                           │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Foundation Models                       │  │
│  │   Claude │ Llama │ Titan │ Mistral │ Cohere │ Stable     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│         ┌────────────────────┼────────────────────┐            │
│         │                    │                    │            │
│         ▼                    ▼                    ▼            │
│  ┌────────────┐     ┌────────────┐      ┌────────────┐        │
│  │  Invoke    │     │ Knowledge  │      │   Agents   │        │
│  │   Model    │     │   Bases    │      │            │        │
│  │            │     │   (RAG)    │      │            │        │
│  └────────────┘     └────────────┘      └────────────┘        │
│         │                    │                    │            │
│         └────────────────────┴────────────────────┘            │
│                              │                                  │
│                      ┌───────┴───────┐                         │
│                      │  Guardrails   │                         │
│                      └───────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Model Invocation

### InvokeModel API

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime')

# Claude example
response = bedrock.invoke_model(
    modelId='anthropic.claude-3-sonnet-20240229-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        'anthropic_version': 'bedrock-2023-05-31',
        'max_tokens': 1024,
        'messages': [
            {'role': 'user', 'content': 'Explain serverless computing'}
        ]
    })
)

result = json.loads(response['body'].read())
print(result['content'][0]['text'])
```

### Streaming Response

```python
response = bedrock.invoke_model_with_response_stream(
    modelId='anthropic.claude-3-sonnet-20240229-v1:0',
    body=json.dumps({...})
)

for event in response['body']:
    chunk = json.loads(event['chunk']['bytes'])
    print(chunk['delta']['text'], end='')
```

### Converse API (Unified)

```python
response = bedrock.converse(
    modelId='anthropic.claude-3-sonnet-20240229-v1:0',
    messages=[
        {'role': 'user', 'content': [{'text': 'Hello'}]}
    ],
    inferenceConfig={
        'maxTokens': 1024,
        'temperature': 0.7
    }
)
```

Works with all text models - no model-specific formatting.

---

## Knowledge Bases (RAG)

Retrieval Augmented Generation with your data.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Knowledge Base                           │
│                                                                 │
│  1. Data Sources                                                │
│     S3 bucket → Documents (PDF, TXT, HTML, MD, CSV, etc.)      │
│                                                                 │
│  2. Chunking                                                    │
│     Documents → Chunks (fixed size, semantic, etc.)            │
│                                                                 │
│  3. Embeddings                                                  │
│     Chunks → Titan Embeddings → Vectors                        │
│                                                                 │
│  4. Vector Store                                                │
│     Vectors → OpenSearch / Aurora / Pinecone / Redis           │
│                                                                 │
│  5. Query                                                       │
│     User question → Retrieve relevant chunks → Generate answer │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Supported Vector Stores
- Amazon OpenSearch Serverless
- Amazon Aurora PostgreSQL
- Pinecone
- Redis Enterprise
- MongoDB Atlas

### Query Knowledge Base

```python
bedrock_agent = boto3.client('bedrock-agent-runtime')

response = bedrock_agent.retrieve_and_generate(
    input={'text': 'What is our refund policy?'},
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': 'KB123456',
            'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0'
        }
    }
)

print(response['output']['text'])
```

---

## Agents

Autonomous AI that reasons, plans, and takes actions.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Bedrock Agent                           │
│                                                                 │
│   User: "Book a flight from NYC to London for next Friday"     │
│                              │                                  │
│                              ▼                                  │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                   Agent Reasoning                       │   │
│   │   1. Understand request                                 │   │
│   │   2. Plan steps                                         │   │
│   │   3. Execute actions                                    │   │
│   │   4. Return results                                     │   │
│   └────────────────────────────────────────────────────────┘   │
│                              │                                  │
│         ┌────────────────────┼────────────────────┐            │
│         ▼                    ▼                    ▼            │
│   ┌──────────┐        ┌──────────┐        ┌──────────┐        │
│   │  Action  │        │  Action  │        │Knowledge │        │
│   │  Group   │        │  Group   │        │  Base    │        │
│   │(Lambda)  │        │(API)     │        │          │        │
│   │          │        │          │        │          │        │
│   │searchFlts│        │ bookFlt  │        │  FAQs    │        │
│   └──────────┘        └──────────┘        └──────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Action Groups
Define what actions the agent can take:
- Lambda functions
- API schemas (OpenAPI)
- Return control to application

---

## Guardrails

Content filters and safety controls.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Guardrails                               │
│                                                                 │
│   Input ──▶ [Filter] ──▶ Model ──▶ [Filter] ──▶ Output         │
│                                                                 │
│   Filters:                                                      │
│   ├── Content filters (hate, violence, sexual, etc.)           │
│   ├── Denied topics (custom topics to block)                   │
│   ├── Word filters (profanity, custom words)                   │
│   ├── Sensitive info (PII detection/masking)                   │
│   └── Contextual grounding (hallucination prevention)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Configure Guardrail

```python
bedrock = boto3.client('bedrock')

response = bedrock.create_guardrail(
    name='content-filter',
    description='Block harmful content',
    contentPolicyConfig={
        'filtersConfig': [
            {'type': 'HATE', 'inputStrength': 'HIGH', 'outputStrength': 'HIGH'},
            {'type': 'VIOLENCE', 'inputStrength': 'HIGH', 'outputStrength': 'HIGH'}
        ]
    },
    sensitiveInformationPolicyConfig={
        'piiEntitiesConfig': [
            {'type': 'EMAIL', 'action': 'ANONYMIZE'},
            {'type': 'PHONE', 'action': 'BLOCK'}
        ]
    }
)
```

---

## Model Customization

### Fine-Tuning
Train model on your data.

```
Your Data (JSONL) → Fine-tuning Job → Custom Model
                         │
                    Training on
                    your examples
```

Supported models:
- Amazon Titan
- Cohere Command
- Meta Llama 2

### Continued Pre-training
Train on unlabeled domain data.

---

## Provisioned Throughput

Reserved capacity for consistent performance.

```
On-Demand:
- Pay per token
- Variable latency
- Good for: Development, variable workloads

Provisioned:
- Pay per model unit/hour
- Consistent latency
- Good for: Production, high-volume
```

---

## Embeddings

Convert text to vectors for semantic search.

```python
response = bedrock.invoke_model(
    modelId='amazon.titan-embed-text-v2:0',
    body=json.dumps({
        'inputText': 'What is machine learning?'
    })
)

embedding = json.loads(response['body'].read())['embedding']
# 1024-dimensional vector
```

Use cases:
- Semantic search
- RAG retrieval
- Similarity matching
- Clustering

---

## Image Generation

### Stable Diffusion

```python
response = bedrock.invoke_model(
    modelId='stability.stable-diffusion-xl-v1',
    body=json.dumps({
        'text_prompts': [
            {'text': 'A sunset over mountains, digital art'}
        ],
        'cfg_scale': 7,
        'steps': 50,
        'seed': 42
    })
)

image_data = json.loads(response['body'].read())['artifacts'][0]['base64']
```

### Titan Image Generator

```python
response = bedrock.invoke_model(
    modelId='amazon.titan-image-generator-v1',
    body=json.dumps({
        'taskType': 'TEXT_IMAGE',
        'textToImageParams': {
            'text': 'A futuristic city'
        }
    })
)
```

---

## CLI Quick Reference

```bash
# List foundation models
aws bedrock list-foundation-models

# Get model details
aws bedrock get-foundation-model --model-identifier anthropic.claude-3-sonnet-20240229-v1:0

# Invoke model
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-sonnet-20240229-v1:0 \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":256,"messages":[{"role":"user","content":"Hello"}]}' \
  --content-type application/json \
  --accept application/json \
  output.json

# List knowledge bases
aws bedrock-agent list-knowledge-bases

# List agents
aws bedrock-agent list-agents

# Create guardrail
aws bedrock create-guardrail --cli-input-json file://guardrail.json
```

---

## Pricing

| Model | Input (per 1K tokens) | Output (per 1K tokens) |
|-------|----------------------|------------------------|
| Claude 3.5 Sonnet | $0.003 | $0.015 |
| Claude 3 Opus | $0.015 | $0.075 |
| Claude 3 Haiku | $0.00025 | $0.00125 |
| Llama 3.1 70B | $0.00099 | $0.00099 |
| Titan Text Express | $0.0002 | $0.0006 |

Knowledge Bases: Storage + queries
Agents: Model invocations + actions

---

## Security

### Access Control
```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude*"
}
```

### VPC Endpoints
Private connectivity without internet.

### Model Access
Must explicitly enable models in console.

### CloudTrail
All API calls logged.

---

## Best Practices

1. **Use Converse API** for portability across models
2. **Implement guardrails** for production
3. **Stream responses** for better UX
4. **Use Knowledge Bases** instead of stuffing context
5. **Start with smaller models** (Haiku) for cost optimization
6. **Cache responses** where appropriate
7. **Monitor token usage** with CloudWatch
8. **Test with multiple models** before committing
9. **Use provisioned throughput** for production SLAs
10. **Implement retry logic** for rate limits

---

## Common Patterns

### Chatbot
```
User → Bedrock (Claude) → Response
         │
    Conversation history
    in messages array
```

### RAG Application
```
User query → Knowledge Base → Retrieved context → Claude → Answer
```

### Document Processing
```
PDF → Textract → Summarize with Claude → Store insights
```

### Code Generation
```
Prompt → Claude → Code → Review → Iterate
```

---

## Exam Tips

1. **Foundation models** - pre-trained, accessed via API
2. **Knowledge Bases** - RAG implementation
3. **Agents** - autonomous actions with reasoning
4. **Guardrails** - content filtering, PII protection
5. **Converse API** - unified API across models
6. **Provisioned throughput** - consistent latency
7. **Embeddings** - Titan for vector generation
8. **Model customization** - fine-tuning on your data
9. **VPC endpoints** - private access
10. **Must enable models** - not available by default
