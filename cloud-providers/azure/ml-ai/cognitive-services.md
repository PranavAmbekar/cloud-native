# Azure AI Services (Cognitive Services)

> Pre-built AI capabilities for vision, speech, language, and decision-making without ML expertise.

## Overview

Azure AI Services (formerly Cognitive Services) provides pre-built AI capabilities that enable developers to add intelligence to applications without building custom ML models. Services are available as REST APIs and SDKs.

## Service Categories

| Category | Services |
|----------|----------|
| **Vision** | Computer Vision, Custom Vision, Face, Document Intelligence |
| **Speech** | Speech-to-Text, Text-to-Speech, Speech Translation, Speaker Recognition |
| **Language** | Language Understanding, Text Analytics, Translator, QnA Maker |
| **Decision** | Content Moderator, Personalizer, Anomaly Detector |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure AI Services                             │
│                                                                  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────────┐  │
│  │  Vision   │ │  Speech   │ │ Language  │ │   Decision    │  │
│  │           │ │           │ │           │ │               │  │
│  │• Computer │ │• Speech   │ │• Language │ │• Content      │  │
│  │  Vision   │ │  to Text  │ │  Service  │ │  Moderator    │  │
│  │• Custom   │ │• Text to  │ │• Translator│ │• Personalizer│  │
│  │  Vision   │ │  Speech   │ │• QnA Maker│ │• Anomaly      │  │
│  │• Face     │ │• Speaker  │ │           │ │  Detector     │  │
│  │• Document │ │  Recog    │ │           │ │               │  │
│  └───────────┘ └───────────┘ └───────────┘ └───────────────┘  │
│                              │                                   │
│                     REST API / SDKs                             │
└─────────────────────────────────────────────────────────────────┘
```

## Resource Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Multi-service** | Access all services with one key | Multiple services needed |
| **Single-service** | Dedicated to one service | Production, isolation |

```bash
# Create multi-service resource
az cognitiveservices account create \
  --name my-ai-services \
  --resource-group myRG \
  --kind CognitiveServices \
  --sku S0 \
  --location eastus

# Create single-service (Computer Vision)
az cognitiveservices account create \
  --name my-vision \
  --resource-group myRG \
  --kind ComputerVision \
  --sku S1 \
  --location eastus
```

## Computer Vision

### Image Analysis

```python
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

client = ComputerVisionClient(
    endpoint="https://my-vision.cognitiveservices.azure.com",
    credentials=CognitiveServicesCredentials("your-key")
)

# Analyze image
features = ["categories", "description", "faces", "objects", "tags"]
result = client.analyze_image(image_url, visual_features=features)

print(f"Description: {result.description.captions[0].text}")
for tag in result.tags:
    print(f"Tag: {tag.name} ({tag.confidence:.2%})")
```

### OCR (Read API)

```python
# Read text from image
read_response = client.read(image_url, raw=True)
operation_location = read_response.headers["Operation-Location"]

# Get results
import time
while True:
    result = client.get_read_result(operation_id)
    if result.status not in ['notStarted', 'running']:
        break
    time.sleep(1)

for page in result.analyze_result.read_results:
    for line in page.lines:
        print(line.text)
```

### Image Analysis 4.0 (Latest)

```python
from azure.ai.vision.imageanalysis import ImageAnalysisClient
from azure.ai.vision.imageanalysis.models import VisualFeatures
from azure.core.credentials import AzureKeyCredential

client = ImageAnalysisClient(
    endpoint="https://my-vision.cognitiveservices.azure.com",
    credential=AzureKeyCredential("your-key")
)

result = client.analyze(
    image_url=url,
    visual_features=[
        VisualFeatures.CAPTION,
        VisualFeatures.OBJECTS,
        VisualFeatures.TAGS,
        VisualFeatures.PEOPLE
    ]
)

print(f"Caption: {result.caption.text}")
```

## Custom Vision

### Training Workflow

```
1. Create Project → 2. Upload Images → 3. Tag Images → 4. Train → 5. Publish → 6. Predict
```

```python
from azure.cognitiveservices.vision.customvision.training import CustomVisionTrainingClient

# Create project
project = trainer.create_project("MyClassifier", classification_type="Multiclass")

# Add tags
tag1 = trainer.create_tag(project.id, "cats")
tag2 = trainer.create_tag(project.id, "dogs")

# Upload images
with open("cat1.jpg", "rb") as image:
    trainer.create_images_from_data(
        project.id,
        image.read(),
        tag_ids=[tag1.id]
    )

# Train
iteration = trainer.train_project(project.id)

# Publish
trainer.publish_iteration(project.id, iteration.id, "mymodel", prediction_resource_id)
```

## Face API

```python
from azure.cognitiveservices.vision.face import FaceClient

client = FaceClient(endpoint, CognitiveServicesCredentials(key))

# Detect faces
faces = client.face.detect_with_url(
    image_url,
    return_face_attributes=["age", "gender", "emotion", "glasses"]
)

for face in faces:
    print(f"Age: {face.face_attributes.age}")
    print(f"Emotion: {face.face_attributes.emotion}")
```

## Document Intelligence (Form Recognizer)

```python
from azure.ai.formrecognizer import DocumentAnalysisClient
from azure.core.credentials import AzureKeyCredential

client = DocumentAnalysisClient(
    endpoint="https://my-formrecognizer.cognitiveservices.azure.com",
    credential=AzureKeyCredential("your-key")
)

# Analyze document
with open("invoice.pdf", "rb") as f:
    poller = client.begin_analyze_document("prebuilt-invoice", f)
    result = poller.result()

for document in result.documents:
    print(f"Vendor: {document.fields.get('VendorName').value}")
    print(f"Total: {document.fields.get('InvoiceTotal').value}")
```

### Pre-built Models

| Model | Extracts |
|-------|----------|
| prebuilt-invoice | Invoice fields |
| prebuilt-receipt | Receipt fields |
| prebuilt-businessCard | Business card info |
| prebuilt-idDocument | ID card, passport |
| prebuilt-document | General document |

## Speech Services

### Speech-to-Text

```python
import azure.cognitiveservices.speech as speechsdk

speech_config = speechsdk.SpeechConfig(
    subscription="your-key",
    region="eastus"
)

audio_config = speechsdk.audio.AudioConfig(use_default_microphone=True)
recognizer = speechsdk.SpeechRecognizer(speech_config, audio_config)

result = recognizer.recognize_once()
print(f"Recognized: {result.text}")
```

### Text-to-Speech

```python
synthesizer = speechsdk.SpeechSynthesizer(speech_config)

# Simple synthesis
result = synthesizer.speak_text("Hello, world!")

# SSML for control
ssml = """
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis" xml:lang="en-US">
    <voice name="en-US-JennyNeural">
        <prosody rate="slow" pitch="+5%">
            Welcome to Azure Speech Services.
        </prosody>
    </voice>
</speak>
"""
result = synthesizer.speak_ssml(ssml)
```

### Real-time Translation

```python
translation_config = speechsdk.translation.SpeechTranslationConfig(
    subscription="your-key",
    region="eastus",
    speech_recognition_language="en-US",
    target_languages=["fr", "de", "es"]
)

recognizer = speechsdk.translation.TranslationRecognizer(translation_config)
result = recognizer.recognize_once()

for lang, translation in result.translations.items():
    print(f"{lang}: {translation}")
```

## Language Service

### Text Analytics

```python
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

client = TextAnalyticsClient(
    endpoint="https://my-language.cognitiveservices.azure.com",
    credential=AzureKeyCredential("your-key")
)

documents = ["I love Azure AI Services!", "The service was terrible."]

# Sentiment analysis
results = client.analyze_sentiment(documents)
for doc in results:
    print(f"Sentiment: {doc.sentiment}")
    print(f"Scores: +{doc.confidence_scores.positive:.2f} -{doc.confidence_scores.negative:.2f}")

# Key phrases
key_phrases = client.extract_key_phrases(documents)
for doc in key_phrases:
    print(f"Key phrases: {doc.key_phrases}")

# Named entities
entities = client.recognize_entities(documents)
for doc in entities:
    for entity in doc.entities:
        print(f"{entity.text}: {entity.category}")

# Language detection
languages = client.detect_language(documents)
for doc in languages:
    print(f"Language: {doc.primary_language.name}")
```

### Conversational Language Understanding (CLU)

```python
from azure.ai.language.conversations import ConversationAnalysisClient

client = ConversationAnalysisClient(endpoint, credential)

result = client.analyze_conversation(
    task={
        "kind": "Conversation",
        "analysisInput": {
            "conversationItem": {
                "text": "Book a flight to Seattle",
                "participantId": "user1"
            }
        },
        "parameters": {
            "projectName": "FlightBooking",
            "deploymentName": "production"
        }
    }
)

print(f"Intent: {result['result']['prediction']['topIntent']}")
for entity in result['result']['prediction']['entities']:
    print(f"Entity: {entity['category']}: {entity['text']}")
```

## Translator

```python
from azure.ai.translation.text import TextTranslationClient

client = TextTranslationClient(
    credential=TranslatorCredential("your-key", "eastus")
)

result = client.translate(
    body=["Hello, world!"],
    to_language=["fr", "de", "es"]
)

for translation in result:
    for t in translation.translations:
        print(f"{t.to}: {t.text}")
```

## Content Moderator

```python
from azure.ai.contentsafety import ContentSafetyClient

client = ContentSafetyClient(endpoint, credential)

# Analyze text
result = client.analyze_text(
    text={"text": "Some text to analyze"}
)

for category in result.categories_analysis:
    print(f"{category.category}: {category.severity}")
```

## Personalizer

```python
from azure.cognitiveservices.personalizer import PersonalizerClient

client = PersonalizerClient(endpoint, credentials)

# Rank actions
rank_request = {
    "contextFeatures": [{"timeOfDay": "morning", "weather": "sunny"}],
    "actions": [
        {"id": "article1", "features": [{"category": "news"}]},
        {"id": "article2", "features": [{"category": "sports"}]}
    ]
}

response = client.rank(rank_request)
print(f"Recommended: {response.reward_action_id}")

# Send reward
client.reward(response.event_id, {"value": 1.0})
```

## CLI Quick Reference

```bash
# List cognitive services kinds
az cognitiveservices account list-kinds

# Create multi-service resource
az cognitiveservices account create \
  --name my-ai \
  --resource-group myRG \
  --kind CognitiveServices \
  --sku S0

# Get keys
az cognitiveservices account keys list \
  --name my-ai \
  --resource-group myRG

# Get endpoint
az cognitiveservices account show \
  --name my-ai \
  --resource-group myRG \
  --query properties.endpoint

# List SKUs
az cognitiveservices account list-skus \
  --kind ComputerVision \
  --location eastus
```

## Pricing Tiers

| Tier | Description |
|------|-------------|
| Free (F0) | Limited calls, testing |
| Standard (S0) | Production, pay-per-call |
| Standard (S1-S4) | Higher limits, commitment |

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **Multi-service**: One endpoint and key for all services
2. **Computer Vision**: Image analysis, OCR, spatial analysis
3. **Custom Vision**: Train your own image classifiers
4. **Face API**: Detection, identification, verification
5. **Document Intelligence**: Extract data from forms/documents
6. **Speech**: STT, TTS, translation, speaker recognition
7. **Language**: Sentiment, entities, key phrases, CLU
8. **Translator**: 100+ languages, custom dictionaries
9. **Content Safety**: Moderate text and images
10. **Containers**: Run services on-premises

## Gotchas

- Some services require specific regions
- Face API has limited access (apply for approval)
- Custom Vision has training limits per project
- Speech services have different endpoints per region
- Language detection is separate from translation
- Rate limits vary by tier and service
- Some features are preview only
- Container deployment doesn't include all features
- Keys should be rotated regularly
- VNET integration requires private endpoints

## Limits

| Service | Free Tier | Standard |
|---------|-----------|----------|
| Computer Vision | 20 calls/min, 5K/month | 10 TPS |
| Face | 20 calls/min, 30K/month | 10 TPS |
| Speech-to-Text | 5 audio hours/month | Pay per hour |
| Text-to-Speech | 5M chars/month | Pay per char |
| Language | 5K calls/month | Pay per call |
| Translator | 2M chars/month | Pay per char |
| Document Intelligence | 500 pages/month | Pay per page |
