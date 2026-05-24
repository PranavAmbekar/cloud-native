# Google Cloud Run

> Fully managed serverless platform for containerized applications that scales automatically.

## Overview

Cloud Run is a managed compute platform that lets you run containers directly on Google's scalable infrastructure. You can deploy code written in any language that can be containerized, without managing servers.

## Key Concepts

| Term | Definition |
|------|------------|
| Service | Deployed application with unique URL |
| Revision | Immutable snapshot of service code and configuration |
| Container | Docker image running your application |
| Concurrency | Requests handled simultaneously per instance |
| Instance | Container running your service |
| Traffic Split | Route traffic between revisions |

## Architecture

```
+---------------------------------------------------------------+
|                      Cloud Run Service                        |
|                                                               |
|  URL: https://my-service-xxx-uc.a.run.app                     |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                     Load Balancer                       |  |
|  |            (Automatic TLS, Global Anycast)              |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|              +---------------+---------------+                |
|              |               |               |                |
|         +----v----+     +----v----+     +----v----+           |
|         |Revision |     |Revision |     |Revision |           |
|         |   v3    |     |   v2    |     |   v1    |           |
|         |  (80%)  |     |  (20%)  |     |  (0%)   |           |
|         +----+----+     +----+----+     +---------+           |
|              |               |                                |
|    +---------+---------+     |                                |
|    |         |         |     |                                |
|  +----+  +----+  +----+   +----+                              |
|  |Inst|  |Inst|  |Inst|   |Inst|   (Auto-scaling 0-N)         |
|  +----+  +----+  +----+   +----+                              |
+---------------------------------------------------------------+
```

## Cloud Run vs Cloud Functions

| Feature | Cloud Run | Cloud Functions |
|---------|-----------|-----------------|
| Code unit | Container | Single function |
| Languages | Any (containerized) | Specific runtimes |
| Trigger | HTTP, Pub/Sub, Eventarc | HTTP, events |
| Timeout | 60 min | 60 min (2nd gen) |
| Concurrency | Up to 1000 | Up to 1000 (2nd gen) |
| WebSockets | Yes | No |
| gRPC | Yes | No |
| Multiple endpoints | Yes | No |

## Deployment

### From Source Code (Buildpacks)

```bash
# Deploy from source (auto-builds container)
gcloud run deploy my-service \
  --source=. \
  --region=us-central1 \
  --allow-unauthenticated
```

### From Container Image

```bash
# Build and push to Artifact Registry
gcloud builds submit --tag gcr.io/my-project/my-service

# Deploy
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-service \
  --region=us-central1 \
  --allow-unauthenticated
```

### Dockerfile Example

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

# Cloud Run uses PORT env variable
ENV PORT=8080
EXPOSE 8080

CMD ["python", "app.py"]
```

```python
# app.py
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello, Cloud Run!'

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

## Configuration

### CPU and Memory

```bash
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-service \
  --memory=2Gi \
  --cpu=2 \
  --concurrency=100 \
  --min-instances=1 \
  --max-instances=100
```

| Setting | Options |
|---------|---------|
| CPU | 1, 2, 4, 6, 8 |
| Memory | 128Mi to 32Gi |
| Concurrency | 1 to 1000 |
| Timeout | 1s to 3600s |

### CPU Allocation

| Mode | Description | Use Case |
|------|-------------|----------|
| **Request-based** | CPU only during request | Web services |
| **Always allocated** | CPU always on | Background work, WebSockets |

```bash
# Always allocated CPU
gcloud run deploy my-service \
  --cpu-boost \
  --no-cpu-throttling
```

## Environment Variables & Secrets

```bash
# Environment variables
gcloud run deploy my-service \
  --set-env-vars="DB_HOST=localhost,DB_PORT=5432"

# Secrets from Secret Manager
gcloud run deploy my-service \
  --set-secrets="API_KEY=my-secret:latest"

# Mount secret as file
gcloud run deploy my-service \
  --set-secrets="/etc/secrets/config.json=config-secret:latest"
```

## Traffic Splitting

```bash
# Deploy new revision without traffic
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-service:v2 \
  --no-traffic

# Gradual rollout
gcloud run services update-traffic my-service \
  --to-revisions=my-service-00002-abc=10

# Full rollout
gcloud run services update-traffic my-service \
  --to-latest

# Rollback
gcloud run services update-traffic my-service \
  --to-revisions=my-service-00001-xyz=100
```

## Custom Domains

```bash
# Map custom domain
gcloud run domain-mappings create \
  --service=my-service \
  --domain=api.example.com \
  --region=us-central1

# Verify domain ownership first
gcloud domains verify example.com
```

## VPC Connectivity

### VPC Connector (Serverless VPC Access)

```bash
# Create connector
gcloud compute networks vpc-access connectors create my-connector \
  --region=us-central1 \
  --subnet=my-subnet \
  --subnet-project=my-project

# Use connector
gcloud run deploy my-service \
  --vpc-connector=my-connector \
  --vpc-egress=all-traffic
```

### Direct VPC Egress (Preview)

```bash
gcloud run deploy my-service \
  --network=my-vpc \
  --subnet=my-subnet \
  --vpc-egress=all-traffic
```

## Authentication

### Public (Unauthenticated)

```bash
gcloud run deploy my-service \
  --allow-unauthenticated
```

### IAM-based

```bash
# Require authentication
gcloud run deploy my-service \
  --no-allow-unauthenticated

# Grant invoker role
gcloud run services add-iam-policy-binding my-service \
  --member=user:alice@example.com \
  --role=roles/run.invoker

# Call authenticated service
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  https://my-service-xxx-uc.a.run.app
```

### Service-to-Service

```python
import google.auth.transport.requests
import google.oauth2.id_token

def call_service():
    audience = "https://target-service-xxx-uc.a.run.app"
    auth_req = google.auth.transport.requests.Request()
    id_token = google.oauth2.id_token.fetch_id_token(auth_req, audience)

    response = requests.get(
        audience,
        headers={"Authorization": f"Bearer {id_token}"}
    )
    return response.json()
```

## Cloud Run Jobs

For batch workloads that run to completion.

```bash
# Create job
gcloud run jobs create my-job \
  --image=gcr.io/my-project/my-batch \
  --tasks=10 \
  --parallelism=5 \
  --max-retries=3

# Execute job
gcloud run jobs execute my-job

# Execute with overrides
gcloud run jobs execute my-job \
  --tasks=20 \
  --update-env-vars="BATCH_ID=123"
```

## Eventarc Triggers

```bash
# Pub/Sub trigger
gcloud eventarc triggers create my-trigger \
  --destination-run-service=my-service \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" \
  --transport-topic=my-topic

# Cloud Storage trigger
gcloud eventarc triggers create storage-trigger \
  --destination-run-service=my-service \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-bucket"
```

## Health Checks

### Startup Probe

```bash
gcloud run deploy my-service \
  --startup-cpu-boost \
  --set-startup-probe="httpGet,path=/health,port=8080,initialDelaySeconds=0,failureThreshold=3"
```

### Liveness Probe

```bash
gcloud run deploy my-service \
  --set-liveness-probe="httpGet,path=/health,port=8080,periodSeconds=10"
```

## CLI Quick Reference

```bash
# Deploy service
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-service \
  --region=us-central1

# List services
gcloud run services list

# Describe service
gcloud run services describe my-service --region=us-central1

# List revisions
gcloud run revisions list --service=my-service

# View logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" --limit=50

# Delete service
gcloud run services delete my-service --region=us-central1

# Get URL
gcloud run services describe my-service --region=us-central1 --format='value(status.url)'

# Update traffic
gcloud run services update-traffic my-service --to-latest

# Execute job
gcloud run jobs execute my-job
```

## Pricing

| Component | Cost |
|-----------|------|
| CPU | $0.00002400 per vCPU-second |
| Memory | $0.00000250 per GiB-second |
| Requests | $0.40 per million requests |
| Free tier | 180,000 vCPU-seconds, 360,000 GiB-seconds, 2M requests/month |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Scale to zero**: Instances scale down to 0 when idle
2. **Concurrency**: Multiple requests per container (default 80)
3. **Min instances**: Prevent cold starts (has cost)
4. **Revisions**: Immutable, enable traffic splitting
5. **PORT**: Container must listen on $PORT env var
6. **Timeout**: 60 minutes max (default 300s)
7. **Jobs**: For batch workloads, run to completion
8. **IAM**: `roles/run.invoker` for authenticated access
9. **VPC Connector**: Required for private resources
10. **Buildpacks**: Auto-detect and containerize source code

## Gotchas

- Container must listen on PORT environment variable
- Default timeout is 300 seconds (not 60 minutes)
- Instances may be terminated anytime
- File system is in-memory (not persistent)
- Minimum 1 request needed to spin up instance
- Cold starts affect first request latency
- WebSockets require "always allocated" CPU
- Service account determines permissions
- Concurrency affects memory requirements
- Traffic splitting only within same region

## Limits

| Resource | Limit |
|----------|-------|
| Request timeout | 3600 seconds |
| Container memory | 32 GiB |
| vCPUs per container | 8 |
| Concurrency per instance | 1000 |
| Max instances per service | 1000 |
| Container image size | 32 GB |
| Request size | 32 MB |
| Response size | 32 MB |
| In-memory filesystem | Based on memory allocation |
| Services per region | 1000 |
| Revisions per service | 1000 |
