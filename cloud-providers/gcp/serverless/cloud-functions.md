# Google Cloud Functions

> Event-driven serverless compute platform for building and deploying single-purpose functions.

## Overview

Cloud Functions is a serverless execution environment for building and connecting cloud services. You write simple, single-purpose functions that are attached to events and automatically scale.

## Key Concepts

| Term | Definition |
|------|------------|
| Function | Single-purpose code responding to events |
| Trigger | Event that invokes the function |
| Runtime | Language execution environment |
| Generation | Function version (1st gen or 2nd gen) |
| Concurrency | Simultaneous executions per instance |
| Cold Start | Initial instance startup latency |

## Generations

| Feature | 1st Gen | 2nd Gen |
|---------|---------|---------|
| Runtime | Node, Python, Go, Java, Ruby, .NET | Same + more |
| Timeout | 9 minutes | 60 minutes |
| Memory | 8 GB max | 32 GB max |
| Concurrency | 1 request per instance | Up to 1000 per instance |
| Min instances | Yes | Yes |
| VPC Connector | Yes | Yes |
| Traffic splitting | No | Yes |
| Built on | Proprietary | Cloud Run |

**Recommendation**: Use 2nd gen for new projects.

## Architecture

```
+---------------------------------------------------------------+
|                        Event Sources                          |
|  +---------+ +---------+ +---------+ +---------+ +---------+  |
|  |  HTTP   | | Pub/Sub | | Storage | |Firestore| |Eventarc |  |
|  | Request | | Message | |  Event  | | Trigger | |  Event  |  |
|  +----+----+ +----+----+ +----+----+ +----+----+ +----+----+  |
+-------+----------+----------+-----------+-----------+----------+
        |          |          |           |           |
        +----------+----------+-----+-----+-----------+
                                    |
                    +---------------v---------------+
                    |       Cloud Functions         |
                    |                               |
                    |  +-------------------------+  |
                    |  |        Instance         |  |
                    |  |  +-------------------+  |  |
                    |  |  |    Function       |  |  |
                    |  |  |      Code         |  |  |
                    |  |  +-------------------+  |  |
                    |  +-------------------------+  |
                    |         ^ (scales)            |
                    |         v                     |
                    |  +-------------------------+  |
                    |  |        Instance         |  |
                    |  +-------------------------+  |
                    +-------------------------------+
```

## Supported Runtimes

| Runtime | Versions |
|---------|----------|
| Node.js | 16, 18, 20 |
| Python | 3.9, 3.10, 3.11, 3.12 |
| Go | 1.19, 1.20, 1.21, 1.22 |
| Java | 11, 17, 21 |
| Ruby | 3.0, 3.2 |
| .NET | 6.0, 8.0 |
| PHP | 8.1, 8.2 |

## Triggers

### HTTP Trigger

```python
# main.py
import functions_framework

@functions_framework.http
def hello_http(request):
    name = request.args.get('name', 'World')
    return f'Hello, {name}!'
```

```bash
# Deploy HTTP function
gcloud functions deploy hello-http \
  --gen2 \
  --runtime=python312 \
  --trigger-http \
  --allow-unauthenticated \
  --region=us-central1 \
  --entry-point=hello_http
```

### Pub/Sub Trigger

```python
import base64
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def process_pubsub(cloud_event: CloudEvent):
    data = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    print(f"Received message: {data}")
```

```bash
gcloud functions deploy process-pubsub \
  --gen2 \
  --runtime=python312 \
  --trigger-topic=my-topic \
  --region=us-central1
```

### Cloud Storage Trigger

```python
@functions_framework.cloud_event
def process_gcs(cloud_event: CloudEvent):
    data = cloud_event.data
    bucket = data["bucket"]
    name = data["name"]
    print(f"File {name} uploaded to {bucket}")
```

```bash
gcloud functions deploy process-gcs \
  --gen2 \
  --runtime=python312 \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=my-bucket" \
  --region=us-central1
```

### Firestore Trigger

```python
@functions_framework.cloud_event
def process_firestore(cloud_event: CloudEvent):
    data = cloud_event.data
    document = data["value"]["fields"]
    print(f"Document updated: {document}")
```

```bash
gcloud functions deploy process-firestore \
  --gen2 \
  --runtime=python312 \
  --trigger-event-filters="type=google.cloud.firestore.document.v1.written" \
  --trigger-event-filters="database=(default)" \
  --trigger-event-filters-path-pattern="document=users/{userId}" \
  --region=us-central1
```

## Environment Variables & Secrets

```bash
# Set environment variables
gcloud functions deploy my-function \
  --set-env-vars DB_HOST=localhost,DB_PORT=5432

# Use secrets from Secret Manager
gcloud functions deploy my-function \
  --set-secrets 'API_KEY=my-secret:latest'

# Mount secret as file
gcloud functions deploy my-function \
  --set-secrets '/etc/secrets/config.json=config-secret:latest'
```

```python
import os

def my_function(request):
    api_key = os.environ.get('API_KEY')
    db_host = os.environ.get('DB_HOST')
```

## Concurrency (2nd Gen)

```
1st Gen: 1 request per instance
+---------+     +---------+     +---------+
|Instance |     |Instance |     |Instance |
| Req 1   |     | Req 2   |     | Req 3   |
+---------+     +---------+     +---------+

2nd Gen: Multiple requests per instance
+-----------------------------------------+
|              Instance                    |
|  Req 1   Req 2   Req 3   ...   Req N    |
|  (up to 1000 concurrent requests)        |
+-----------------------------------------+
```

```bash
# Set concurrency (2nd gen only)
gcloud functions deploy my-function \
  --gen2 \
  --concurrency=100 \
  --cpu=2
```

## Cold Start Mitigation

### Min Instances

```bash
# Keep instances warm
gcloud functions deploy my-function \
  --gen2 \
  --min-instances=1 \
  --max-instances=100
```

### Best Practices

```python
# Initialize outside handler (reused across invocations)
from google.cloud import storage
client = storage.Client()  # Reused

@functions_framework.http
def my_function(request):
    # Use pre-initialized client
    bucket = client.bucket('my-bucket')
    return 'OK'
```

## VPC Connectivity

```bash
# Connect to VPC
gcloud functions deploy my-function \
  --gen2 \
  --vpc-connector=my-connector \
  --egress-settings=all-traffic
```

### Egress Settings

| Setting | Description |
|---------|-------------|
| `private-ranges-only` | VPC for private IPs, internet for public |
| `all-traffic` | All traffic through VPC |

## Testing Locally

```bash
# Install Functions Framework
pip install functions-framework

# Run locally
functions-framework --target=hello_http --port=8080

# Test
curl http://localhost:8080?name=World
```

## Deployment with Source

```bash
# From current directory
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --trigger-http \
  --source=.

# From GCS
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --trigger-http \
  --source=gs://my-bucket/function-source.zip

# From repository
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --trigger-http \
  --source=https://source.developers.google.com/projects/my-project/repos/my-repo
```

## Requirements File

```
# requirements.txt
functions-framework==3.*
google-cloud-storage==2.*
google-cloud-pubsub==2.*
requests==2.*
```

## CLI Quick Reference

```bash
# Deploy function (2nd gen)
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --trigger-http \
  --region=us-central1 \
  --entry-point=main

# List functions
gcloud functions list

# Describe function
gcloud functions describe my-function --region=us-central1

# View logs
gcloud functions logs read my-function --region=us-central1

# Call function
gcloud functions call my-function --region=us-central1 --data='{"name":"World"}'

# Delete function
gcloud functions delete my-function --region=us-central1

# Get URL
gcloud functions describe my-function --region=us-central1 --format='value(url)'

# Set IAM policy
gcloud functions add-iam-policy-binding my-function \
  --region=us-central1 \
  --member=allUsers \
  --role=roles/cloudfunctions.invoker
```

## Pricing

| Component | Cost |
|-----------|------|
| Invocations | $0.40 per million |
| Compute (GB-s) | $0.0000025 per GB-s |
| Compute (GHz-s) | $0.00001 per GHz-s |
| Networking | Egress charges apply |
| Free tier | 2M invocations, 400K GB-s, 200K GHz-s/month |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **2nd Gen**: Built on Cloud Run, more features
2. **Concurrency**: 2nd gen supports multiple requests per instance
3. **Cold starts**: Use min instances for latency-sensitive
4. **Timeout**: 1st gen = 9 min, 2nd gen = 60 min
5. **Memory**: 1st gen = 8 GB max, 2nd gen = 32 GB max
6. **Eventarc**: Event routing for 2nd gen
7. **VPC Connector**: Required for private resources
8. **IAM**: `roles/cloudfunctions.invoker` for invocation
9. **Secrets**: Use Secret Manager, not env vars for sensitive data
10. **Traffic splitting**: Only available in 2nd gen

## Gotchas

- 1st gen and 2nd gen have different capabilities
- Background functions (1st gen) vs CloudEvents (2nd gen)
- `--allow-unauthenticated` required for public access
- Environment variables visible in console (use secrets)
- VPC Connector has its own billing
- Min instances incur charges even when idle
- Function name must be unique per region
- Timeout starts when function begins executing
- File system is read-only except /tmp
- /tmp is per-instance, not shared

## Limits

| Resource | 1st Gen | 2nd Gen |
|----------|---------|---------|
| Timeout | 540s (9 min) | 3600s (60 min) |
| Memory | 8 GB | 32 GB |
| vCPUs | 4.8 | 8 |
| Concurrency | 1 | 1000 |
| Max instances | 3000 | 1000 |
| Deployment size | 500 MB (compressed) | 500 MB |
| HTTP request size | 10 MB | 32 MB |
| HTTP response size | 10 MB | 32 MB |
| /tmp storage | 512 MB | 512 MB - 32 GB |
