# AWS AI Services

> Pre-trained AI APIs for vision, speech, language, and more - no ML expertise required.

---

## Service Overview

| Service | Category | Purpose |
|---------|----------|---------|
| Rekognition | Vision | Image and video analysis |
| Textract | Vision | Document text extraction |
| Comprehend | Language | NLP and text analysis |
| Translate | Language | Language translation |
| Transcribe | Speech | Speech to text |
| Polly | Speech | Text to speech |
| Lex | Conversational | Chatbots and voice bots |
| Kendra | Search | Intelligent enterprise search |
| Personalize | Recommendations | Personalized recommendations |
| Forecast | Time Series | Time series forecasting |

---

## Amazon Rekognition

Image and video analysis.

### Capabilities

```
┌─────────────────────────────────────────────────────────────────┐
│                      Rekognition                                │
│                                                                 │
│  Image Analysis                    Video Analysis               │
│  ├── Object/Scene detection        ├── People tracking          │
│  ├── Face detection/comparison     ├── Face search              │
│  ├── Celebrity recognition         ├── Path tracking            │
│  ├── Text in images (OCR)          ├── Activity detection       │
│  ├── Content moderation            ├── Segment detection        │
│  └── Custom labels                 └── Content moderation       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Face Detection

```python
import boto3

rekognition = boto3.client('rekognition')

response = rekognition.detect_faces(
    Image={'S3Object': {'Bucket': 'my-bucket', 'Name': 'photo.jpg'}},
    Attributes=['ALL']
)

for face in response['FaceDetails']:
    print(f"Age: {face['AgeRange']}")
    print(f"Emotions: {face['Emotions']}")
    print(f"Smile: {face['Smile']['Value']}")
```

### Face Comparison

```python
response = rekognition.compare_faces(
    SourceImage={'S3Object': {'Bucket': 'bucket', 'Name': 'source.jpg'}},
    TargetImage={'S3Object': {'Bucket': 'bucket', 'Name': 'target.jpg'}},
    SimilarityThreshold=80
)

for match in response['FaceMatches']:
    print(f"Similarity: {match['Similarity']}%")
```

### Custom Labels
Train custom object detection models.

```
Your Images → Label → Train → Deploy → Detect custom objects
```

### Content Moderation

```python
response = rekognition.detect_moderation_labels(
    Image={'S3Object': {'Bucket': 'bucket', 'Name': 'image.jpg'}}
)

# Returns: Nudity, Violence, Drugs, etc.
```

---

## Amazon Textract

Extract text, forms, and tables from documents.

### Capabilities

```
┌─────────────────────────────────────────────────────────────────┐
│                        Textract                                 │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Detect     │  │   Analyze    │  │   Analyze    │          │
│  │   Text       │  │   Document   │  │   Expense    │          │
│  │              │  │              │  │              │          │
│  │  Raw text    │  │  Forms +     │  │  Receipts +  │          │
│  │  extraction  │  │  Tables      │  │  Invoices    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │   Analyze    │  │   Lending    │                            │
│  │   ID         │  │   Document   │                            │
│  │              │  │              │                            │
│  │  Passports,  │  │  Mortgage,   │                            │
│  │  Driver lic  │  │  loan docs   │                            │
│  └──────────────┘  └──────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Document Analysis

```python
import boto3

textract = boto3.client('textract')

# Analyze document with forms and tables
response = textract.analyze_document(
    Document={'S3Object': {'Bucket': 'bucket', 'Name': 'form.pdf'}},
    FeatureTypes=['FORMS', 'TABLES']
)

# Extract key-value pairs (forms)
for block in response['Blocks']:
    if block['BlockType'] == 'KEY_VALUE_SET':
        # Process form fields
        pass
```

### Async Processing (Large Documents)

```python
# Start async job
response = textract.start_document_analysis(
    DocumentLocation={'S3Object': {'Bucket': 'bucket', 'Name': 'large.pdf'}},
    FeatureTypes=['FORMS', 'TABLES']
)

job_id = response['JobId']

# Check status
status = textract.get_document_analysis(JobId=job_id)
```

---

## Amazon Comprehend

Natural Language Processing (NLP).

### Capabilities

| Feature | Description |
|---------|-------------|
| Entity Recognition | People, places, dates, organizations |
| Sentiment Analysis | Positive, negative, neutral, mixed |
| Key Phrases | Important phrases extraction |
| Language Detection | Identify language |
| Syntax Analysis | Parts of speech |
| Topic Modeling | Discover topics in documents |
| Custom Classification | Train custom classifiers |
| Custom Entities | Train custom entity recognizers |
| PII Detection | Detect/redact personal info |

### Sentiment Analysis

```python
import boto3

comprehend = boto3.client('comprehend')

response = comprehend.detect_sentiment(
    Text="I love this product! It's amazing.",
    LanguageCode='en'
)

print(response['Sentiment'])  # POSITIVE
print(response['SentimentScore'])  # {Positive: 0.99, ...}
```

### Entity Recognition

```python
response = comprehend.detect_entities(
    Text="John Smith works at Amazon in Seattle.",
    LanguageCode='en'
)

# Returns:
# John Smith - PERSON
# Amazon - ORGANIZATION
# Seattle - LOCATION
```

### PII Detection

```python
response = comprehend.detect_pii_entities(
    Text="Contact John at john@email.com or 555-123-4567",
    LanguageCode='en'
)

# Returns: EMAIL, PHONE entities with positions
```

### Comprehend Medical
Specialized for medical text.
- Medical entities (medications, conditions)
- PHI detection
- ICD-10-CM/RxNorm linking

---

## Amazon Translate

Neural machine translation.

```python
import boto3

translate = boto3.client('translate')

response = translate.translate_text(
    Text="Hello, how are you?",
    SourceLanguageCode='en',
    TargetLanguageCode='es'
)

print(response['TranslatedText'])  # "Hola, ¿cómo estás?"
```

### Features
- 75+ languages
- Custom terminology
- Active Custom Translation (domain adaptation)
- Real-time and batch translation
- Formality control

### Batch Translation

```python
response = translate.start_text_translation_job(
    InputDataConfig={
        'S3Uri': 's3://bucket/input/',
        'ContentType': 'text/plain'
    },
    OutputDataConfig={'S3Uri': 's3://bucket/output/'},
    DataAccessRoleArn='arn:aws:iam::xxx:role/TranslateRole',
    SourceLanguageCode='en',
    TargetLanguageCodes=['es', 'fr', 'de']
)
```

---

## Amazon Transcribe

Speech to text.

### Features

```
┌─────────────────────────────────────────────────────────────────┐
│                       Transcribe                                │
│                                                                 │
│  ├── Real-time streaming                                       │
│  ├── Batch transcription                                       │
│  ├── Speaker identification (up to 10)                         │
│  ├── Channel identification (call centers)                     │
│  ├── Custom vocabulary                                         │
│  ├── Automatic language detection                              │
│  ├── PII redaction                                             │
│  ├── Toxicity detection                                        │
│  └── Subtitles (SRT, VTT)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Batch Transcription

```python
import boto3

transcribe = boto3.client('transcribe')

response = transcribe.start_transcription_job(
    TranscriptionJobName='my-job',
    Media={'MediaFileUri': 's3://bucket/audio.mp3'},
    MediaFormat='mp3',
    LanguageCode='en-US',
    Settings={
        'ShowSpeakerLabels': True,
        'MaxSpeakerLabels': 2
    }
)
```

### Streaming Transcription

```python
from amazon_transcribe.client import TranscribeStreamingClient

async def transcribe_stream():
    client = TranscribeStreamingClient(region='us-east-1')
    stream = await client.start_stream_transcription(
        language_code='en-US',
        media_sample_rate_hz=16000,
        media_encoding='pcm'
    )
    # Stream audio and receive transcripts
```

### Transcribe Medical
- Medical conversations
- HIPAA eligible
- Medical vocabulary

### Transcribe Call Analytics
- Call center analytics
- Sentiment per speaker
- Call categorization
- Issues detection

---

## Amazon Polly

Text to speech.

### Voices

| Type | Description |
|------|-------------|
| Standard | Basic TTS |
| Neural | More natural, lifelike |
| Long-form | Optimized for articles |
| Generative | Most natural (preview) |

### Speech Synthesis

```python
import boto3

polly = boto3.client('polly')

response = polly.synthesize_speech(
    Text='Hello, welcome to AWS Polly.',
    OutputFormat='mp3',
    VoiceId='Joanna',
    Engine='neural'
)

# Save audio
with open('output.mp3', 'wb') as f:
    f.write(response['AudioStream'].read())
```

### SSML Support

```python
response = polly.synthesize_speech(
    Text='''
    <speak>
        Hello <break time="1s"/>
        <prosody rate="slow">This is slower.</prosody>
        <emphasis level="strong">This is important!</emphasis>
    </speak>
    ''',
    TextType='ssml',
    OutputFormat='mp3',
    VoiceId='Joanna'
)
```

### Speech Marks
Get word/sentence timing for lip-sync.

---

## Amazon Lex

Conversational AI for chatbots.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          Lex Bot                                │
│                                                                 │
│   User: "I want to book a flight to London"                    │
│                     │                                           │
│                     ▼                                           │
│   ┌─────────────────────────────────────────┐                  │
│   │              Intent: BookFlight          │                  │
│   │                                          │                  │
│   │  Slots:                                  │                  │
│   │  ├── destination: London                │                  │
│   │  ├── departure_date: (ask user)         │                  │
│   │  └── return_date: (ask user)            │                  │
│   │                                          │                  │
│   │  Fulfillment: Lambda function            │                  │
│   └─────────────────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Integrations
- Amazon Connect (call center)
- Facebook Messenger
- Slack
- Twilio SMS
- Web/mobile apps

### Lex V2
- Improved NLU
- Multiple languages per bot
- Streaming conversations

---

## Amazon Kendra

Intelligent enterprise search.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kendra Index                             │
│                                                                 │
│   Data Sources:                                                 │
│   ├── S3                                                       │
│   ├── SharePoint                                               │
│   ├── Salesforce                                               │
│   ├── ServiceNow                                               │
│   ├── Databases                                                │
│   └── Custom connectors                                        │
│                                                                 │
│   Query: "What is our vacation policy?"                        │
│                     │                                           │
│                     ▼                                           │
│   ┌─────────────────────────────────────────┐                  │
│   │  Results:                                │                  │
│   │  1. [Answer] Employees get 20 days...   │                  │
│   │  2. [Document] HR_Vacation_Policy.pdf   │                  │
│   │  3. [FAQ] Common vacation questions     │                  │
│   └─────────────────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Features
- Natural language queries
- FAQ matching
- Document ranking
- Access control
- Custom synonyms
- Relevance tuning

---

## Amazon Personalize

Real-time personalization and recommendations.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Personalize                                │
│                                                                 │
│   1. Import Data                                               │
│      ├── User interactions (clicks, purchases)                 │
│      ├── User metadata                                         │
│      └── Item metadata                                         │
│                                                                 │
│   2. Create Solution                                           │
│      └── Recipe: User-Personalization, Similar-Items, etc.     │
│                                                                 │
│   3. Deploy Campaign                                           │
│      └── Real-time recommendations API                         │
│                                                                 │
│   API: GetRecommendations(userId) → [item1, item2, ...]        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Recipes
- User-Personalization
- Similar-Items
- Personalized-Ranking
- Trending-Now
- Next-Best-Action

---

## Amazon Forecast

Time series forecasting.

```
Historical Data → Forecast → Predictions
(sales, demand,     │
 traffic, etc.)     │
                    ▼
              ┌──────────┐
              │ Forecast │
              │          │
              │ P10 ─────│─── Lower bound
              │ P50 ─────│─── Most likely
              │ P90 ─────│─── Upper bound
              └──────────┘
```

### AutoML
Automatically selects best algorithm:
- DeepAR+
- Prophet
- NPTS
- ARIMA
- ETS

---

## Pricing Summary

| Service | Pricing Model |
|---------|---------------|
| Rekognition | Per image/video minute |
| Textract | Per page |
| Comprehend | Per unit (100 chars) |
| Translate | Per character |
| Transcribe | Per second |
| Polly | Per character |
| Lex | Per request |
| Kendra | Per hour + queries |
| Personalize | Per data GB + TPS |
| Forecast | Per data GB + forecasts |

---

## Exam Tips

1. **Rekognition** - faces, objects, moderation, custom labels
2. **Textract** - OCR + forms + tables (beyond OCR)
3. **Comprehend** - sentiment, entities, key phrases, PII
4. **Comprehend Medical** - medical NLP, PHI detection
5. **Translate** - 75+ languages, custom terminology
6. **Transcribe** - speech-to-text, speaker ID, medical variant
7. **Polly** - text-to-speech, neural voices, SSML
8. **Lex** - chatbots, intents, slots, Lambda fulfillment
9. **Kendra** - enterprise search, natural language
10. **Personalize** - recommendations, real-time
11. **Forecast** - time series predictions, AutoML
12. **All services** - can process S3 data, integrate with Lambda
