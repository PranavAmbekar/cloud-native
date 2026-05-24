# Azure OpenAI Service

> Access OpenAI's powerful language models including GPT-4, GPT-3.5, DALL-E, and embeddings through Azure's enterprise platform.

## Overview

Azure OpenAI Service provides REST API access to OpenAI's powerful language models including GPT-4, GPT-4 Turbo, GPT-3.5-Turbo, Embeddings, and DALL-E. These models can be adapted to your specific task including content generation, summarization, semantic search, and natural language to code translation.

## Key Concepts

| Term | Definition |
|------|------------|
| Resource | Azure OpenAI instance |
| Deployment | Model instance with specific configuration |
| Model | AI model (GPT-4, GPT-3.5, etc.) |
| Token | Unit of text (~4 chars English) |
| Prompt | Input text to the model |
| Completion | Model's generated response |
| Embedding | Vector representation of text |

## Available Models

### GPT Models

| Model | Context | Description |
|-------|---------|-------------|
| gpt-4o | 128K | Latest, multimodal (text + images) |
| gpt-4-turbo | 128K | Powerful, vision capable |
| gpt-4 | 8K/32K | Advanced reasoning |
| gpt-35-turbo | 4K/16K | Fast, cost-effective |

### Other Models

| Model | Purpose |
|-------|---------|
| text-embedding-ada-002 | Text embeddings (1536 dimensions) |
| text-embedding-3-small | Newer embeddings (1536 dimensions) |
| text-embedding-3-large | High-performance embeddings (3072 dimensions) |
| dall-e-3 | Image generation |
| whisper | Speech-to-text |
| tts | Text-to-speech |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure OpenAI Service                          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Resource                               │   │
│  │                                                           │   │
│  │   ┌─────────────────┐  ┌─────────────────┐              │   │
│  │   │   Deployment    │  │   Deployment    │              │   │
│  │   │   "gpt4-prod"   │  │   "gpt35-dev"   │              │   │
│  │   │   (gpt-4)       │  │   (gpt-35-turbo)│              │   │
│  │   └─────────────────┘  └─────────────────┘              │   │
│  │                                                           │   │
│  │   ┌─────────────────┐  ┌─────────────────┐              │   │
│  │   │   Deployment    │  │   Deployment    │              │   │
│  │   │   "embeddings"  │  │   "dalle"       │              │   │
│  │   │   (ada-002)     │  │   (dall-e-3)    │              │   │
│  │   └─────────────────┘  └─────────────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Endpoint: https://myresource.openai.azure.com                 │
└─────────────────────────────────────────────────────────────────┘
```

## Create Resource and Deployment

```bash
# Create Azure OpenAI resource
az cognitiveservices account create \
  --name myopenai \
  --resource-group myRG \
  --location eastus \
  --kind OpenAI \
  --sku S0

# Create deployment
az cognitiveservices account deployment create \
  --name myopenai \
  --resource-group myRG \
  --deployment-name gpt4-deployment \
  --model-name gpt-4 \
  --model-version "0613" \
  --model-format OpenAI \
  --sku-name Standard \
  --sku-capacity 10
```

## API Usage

### Chat Completions (GPT-4, GPT-3.5)

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="your-api-key",
    api_version="2024-02-15-preview",
    azure_endpoint="https://myresource.openai.azure.com"
)

response = client.chat.completions.create(
    model="gpt4-deployment",  # deployment name
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Azure OpenAI?"}
    ],
    temperature=0.7,
    max_tokens=800
)

print(response.choices[0].message.content)
```

### Streaming Response

```python
response = client.chat.completions.create(
    model="gpt4-deployment",
    messages=[{"role": "user", "content": "Tell me a story"}],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### Embeddings

```python
response = client.embeddings.create(
    model="text-embedding-ada-002",
    input="Azure OpenAI provides access to powerful language models"
)

embedding = response.data[0].embedding  # List of 1536 floats
```

### Image Generation (DALL-E)

```python
response = client.images.generate(
    model="dalle3",
    prompt="A futuristic city with flying cars",
    n=1,
    size="1024x1024",
    quality="standard"  # or "hd"
)

image_url = response.data[0].url
```

### Function Calling

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {"type": "string"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                },
                "required": ["location"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="gpt4-deployment",
    messages=[{"role": "user", "content": "What's the weather in Seattle?"}],
    tools=tools,
    tool_choice="auto"
)

# Check if model wants to call a function
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    # Execute function and send result back
```

## Parameters

### Chat Completion Parameters

| Parameter | Description | Range |
|-----------|-------------|-------|
| temperature | Randomness | 0-2 (default 1) |
| max_tokens | Max response length | Model dependent |
| top_p | Nucleus sampling | 0-1 (default 1) |
| frequency_penalty | Reduce repetition | -2 to 2 |
| presence_penalty | Encourage new topics | -2 to 2 |
| stop | Stop sequences | Up to 4 strings |

### Temperature vs Top_p

```
Temperature 0: Deterministic, consistent
Temperature 1: Balanced creativity
Temperature 2: Very random, may be incoherent

Top_p 0.1: Consider only top 10% likely tokens
Top_p 1.0: Consider all tokens

Note: Use one or the other, not both
```

## Content Filtering

Azure OpenAI includes built-in content filtering.

| Category | Description |
|----------|-------------|
| Hate | Discrimination, slurs |
| Sexual | Sexual content |
| Violence | Graphic violence |
| Self-harm | Self-harm content |

### Severity Levels

| Level | Description |
|-------|-------------|
| Safe | Low severity, allowed |
| Low | Allowed with standard filter |
| Medium | Blocked with standard filter |
| High | Always blocked |

```python
# Content filter results in response
response.choices[0].content_filter_results = {
    "hate": {"filtered": False, "severity": "safe"},
    "self_harm": {"filtered": False, "severity": "safe"},
    "sexual": {"filtered": False, "severity": "safe"},
    "violence": {"filtered": False, "severity": "safe"}
}
```

## Quota and Rate Limits

### Tokens Per Minute (TPM)

```
Quota allocation:
├── GPT-4: 10,000 TPM (default, can increase)
├── GPT-35-Turbo: 120,000 TPM
└── Embeddings: 120,000 TPM

Rate limiting headers:
x-ratelimit-remaining-tokens
x-ratelimit-remaining-requests
```

### Provisioned Throughput

```
Standard (Pay-as-you-go):
├── Shared capacity
├── Variable latency
└── Best for: Dev/test, variable workloads

Provisioned (PTU):
├── Reserved capacity
├── Consistent latency
├── Best for: Production, predictable workloads
└── Billing: Per provisioned throughput unit/hour
```

## On Your Data

Connect to your own data sources.

```python
from azure.identity import DefaultAzureCredential

response = client.chat.completions.create(
    model="gpt4-deployment",
    messages=[{"role": "user", "content": "What products do we sell?"}],
    extra_body={
        "data_sources": [{
            "type": "azure_search",
            "parameters": {
                "endpoint": "https://mysearch.search.windows.net",
                "index_name": "products",
                "authentication": {
                    "type": "api_key",
                    "key": "search-api-key"
                }
            }
        }]
    }
)

# Response includes citations from your data
print(response.choices[0].message.context["citations"])
```

## Assistants API

Create AI assistants with tools and file handling.

```python
# Create assistant
assistant = client.beta.assistants.create(
    name="Data Analyst",
    instructions="You are a data analyst. Analyze data and create visualizations.",
    tools=[{"type": "code_interpreter"}],
    model="gpt4-deployment"
)

# Create thread
thread = client.beta.threads.create()

# Add message
message = client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="Analyze the attached CSV file"
)

# Run assistant
run = client.beta.threads.runs.create(
    thread_id=thread.id,
    assistant_id=assistant.id
)
```

## Security

### Authentication

| Method | Description |
|--------|-------------|
| API Key | Simple key-based auth |
| Azure AD | Managed identity, service principal |
| RBAC Roles | Cognitive Services User, Contributor |

### Network Security

```bash
# Private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id <resource-id> \
  --group-id account \
  --connection-name myConnection
```

## Pricing

| Model | Input (per 1K tokens) | Output (per 1K tokens) |
|-------|----------------------|------------------------|
| GPT-4o | $0.005 | $0.015 |
| GPT-4-Turbo | $0.01 | $0.03 |
| GPT-4 (8K) | $0.03 | $0.06 |
| GPT-35-Turbo | $0.0005 | $0.0015 |
| Embeddings (ada-002) | $0.0001 | N/A |
| DALL-E 3 (Standard) | $0.04/image | N/A |

## CLI Quick Reference

```bash
# Create resource
az cognitiveservices account create \
  --name myopenai \
  --resource-group myRG \
  --kind OpenAI \
  --sku S0 \
  --location eastus

# List deployments
az cognitiveservices account deployment list \
  --name myopenai \
  --resource-group myRG

# Create deployment
az cognitiveservices account deployment create \
  --name myopenai \
  --resource-group myRG \
  --deployment-name gpt4 \
  --model-name gpt-4 \
  --model-version "0613" \
  --model-format OpenAI \
  --sku-name Standard \
  --sku-capacity 10

# Get keys
az cognitiveservices account keys list \
  --name myopenai \
  --resource-group myRG

# Update quota
az cognitiveservices account deployment create \
  --name myopenai \
  --resource-group myRG \
  --deployment-name gpt4 \
  --sku-capacity 20
```

## Exam Tips (AZ-204, AZ-305)

1. **Deployment**: Model instance with specific name and configuration
2. **API versions**: Include in all requests, format: 2024-02-15-preview
3. **Content filtering**: Built-in, four categories
4. **Rate limits**: TPM (tokens) and RPM (requests)
5. **On Your Data**: Connect Azure Search for RAG
6. **Tokens**: ~4 chars English, varies by language
7. **Temperature**: 0 = deterministic, higher = more random
8. **Function calling**: Model decides when to call functions
9. **Private endpoint**: Required for VNet isolation
10. **RBAC**: Use Cognitive Services User for API access

## Gotchas

- Deployment name != model name (you choose deployment name)
- API version required in all requests
- Content filtering cannot be fully disabled
- Rate limits apply per deployment, not per resource
- Some models/features have regional availability
- Embedding dimensions fixed per model
- Token count differs between models
- Streaming responses have different structure
- Function calls count toward token usage
- PTU requires commitment and advance planning

## Limits

| Resource | Limit |
|----------|-------|
| Deployments per resource | 30 |
| Resources per subscription per region | 3 |
| Max tokens (GPT-4o) | 128K context |
| Max tokens (GPT-35-Turbo-16k) | 16K context |
| Max output tokens | 4096 |
| Functions per request | 128 |
| Messages per request | 2048 |
| Files per assistant | 20 |
| Image size (DALL-E) | 1024x1024, 1024x1792, 1792x1024 |
| Rate limit requests | Varies by model/region |
