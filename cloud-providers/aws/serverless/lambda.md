# AWS Lambda

> Run code without provisioning or managing servers. Pay only for compute time consumed.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Function | Your code + configuration |
| Handler | Entry point method (e.g., `index.handler`) |
| Runtime | Language environment (Python, Node.js, Java, etc.) |
| Trigger | Event source that invokes the function |
| Layer | Reusable code/dependencies shared across functions |
| Concurrency | Number of simultaneous executions |
| Cold Start | Initial setup time when new instance spins up |

---

## Supported Runtimes

| Runtime | Versions |
|---------|----------|
| Node.js | 18.x, 20.x |
| Python | 3.9, 3.10, 3.11, 3.12 |
| Java | 11, 17, 21 |
| .NET | 6, 8 |
| Ruby | 3.2, 3.3 |
| Go | Provided (al2023) |
| Rust | Provided (al2023) |
| Custom | Any language via custom runtime |

---

## Configuration Limits

| Resource | Limit |
|----------|-------|
| Memory | 128 MB - 10,240 MB (10 GB) |
| Timeout | 1 sec - 15 minutes |
| Package size (zipped) | 50 MB |
| Package size (unzipped) | 250 MB |
| Container image | 10 GB |
| Environment variables | 4 KB total |
| /tmp storage | 512 MB - 10,240 MB |
| Concurrent executions | 1,000 (default, can increase) |
| Layers per function | 5 |

---

## Invocation Models

### Synchronous
```
Client → Lambda → Response
         ↓
      Waits for completion
```
- API Gateway, ALB, Cognito
- Client waits for response
- Errors returned to caller

### Asynchronous
```
Client → Lambda → 202 Accepted
              ↓
         Queued for processing
```
- S3, SNS, EventBridge, CloudWatch Events
- Automatic retry (2 retries)
- Dead Letter Queue (DLQ) for failures

### Event Source Mapping
```
Source → Lambda polls → Processes batches
(SQS, Kinesis, DynamoDB Streams)
```
- Lambda polls the source
- Batch processing
- Configurable batch size and window

---

## Event Sources (Triggers)

| Category | Services |
|----------|----------|
| API | API Gateway, ALB, Function URLs |
| Storage | S3, EFS |
| Database | DynamoDB Streams, RDS (via EventBridge) |
| Messaging | SQS, SNS, Kinesis, MSK (Kafka) |
| Orchestration | Step Functions, EventBridge |
| Other | CloudWatch Events, Cognito, Alexa, IoT |

---

## Execution Environment

```
┌─────────────────────────────────────────┐
│         Execution Environment           │
│  ┌───────────────────────────────────┐  │
│  │         /tmp (ephemeral)          │  │
│  │         512 MB - 10 GB            │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │         Runtime                   │  │
│  │    ┌─────────────────────────┐    │  │
│  │    │     Your Function       │    │  │
│  │    │    Code + Layers        │    │  │
│  │    └─────────────────────────┘    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
     │
     │ Reused for subsequent invocations
     │ (warm start)
```

---

## Cold Start vs Warm Start

| Type | Description | Latency |
|------|-------------|---------|
| Cold Start | New execution environment | 100ms - 5s+ |
| Warm Start | Reusing existing environment | <100ms |

### Reducing Cold Starts
- Use smaller packages
- Provisioned Concurrency
- Keep functions warm (scheduled pings)
- Use lighter runtimes (Python, Node.js)
- Initialize outside handler

```python
# Good - initialized once, reused
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('my-table')

def handler(event, context):
    # Uses pre-initialized table
    return table.get_item(Key={'id': event['id']})
```

---

## Concurrency

### Types
| Type | Description |
|------|-------------|
| Unreserved | Shared pool (default 1000) |
| Reserved | Guaranteed for specific function |
| Provisioned | Pre-initialized, no cold starts |

### Formula
```
Concurrency = (requests/sec) × (avg duration in sec)

Example:
100 requests/sec × 0.5 sec = 50 concurrent executions
```

---

## Layers

Reusable components containing libraries, custom runtimes, or dependencies.

```
Function
    ├── /opt/python (Python libraries)
    ├── /opt/nodejs (Node.js modules)
    └── /opt/bin (executables)
```

```bash
# Create layer
zip -r layer.zip python/
aws lambda publish-layer-version \
  --layer-name my-layer \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.11
```

---

## Function URLs

Direct HTTPS endpoint without API Gateway.

```
https://<url-id>.lambda-url.<region>.on.aws
```

| Feature | Function URL | API Gateway |
|---------|--------------|-------------|
| Cost | Free | Per request |
| Auth | IAM or None | IAM, Cognito, API Keys |
| Features | Basic | Full (throttling, caching, etc.) |
| Custom domain | No (use CloudFront) | Yes |

---

## Environment Variables

```bash
# Set via CLI
aws lambda update-function-configuration \
  --function-name my-function \
  --environment "Variables={DB_HOST=localhost,DB_PORT=5432}"
```

```python
import os

def handler(event, context):
    db_host = os.environ['DB_HOST']
    db_port = os.environ['DB_PORT']
```

### Encryption
- Encrypted at rest by default (AWS managed key)
- Use KMS CMK for additional control
- Decrypt in function for sensitive values

---

## VPC Integration

```
┌─────────────────────────────────────┐
│              VPC                    │
│  ┌─────────────────────────────┐   │
│  │      Private Subnet         │   │
│  │  ┌───────┐    ┌─────────┐  │   │
│  │  │Lambda │────│   RDS   │  │   │
│  │  │ (ENI) │    └─────────┘  │   │
│  │  └───────┘                  │   │
│  └─────────────────────────────┘   │
│               │                     │
│               ▼                     │
│  ┌─────────────────────────────┐   │
│  │      NAT Gateway            │   │ → Internet
│  │   (for internet access)     │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

- Required for accessing VPC resources (RDS, ElastiCache)
- Adds cold start latency (~1s with Hyperplane)
- Needs NAT Gateway for internet access
- Use VPC Endpoints for AWS services

---

## IAM Permissions

### Execution Role (what Lambda can do)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### Resource Policy (who can invoke Lambda)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "s3.amazonaws.com"},
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:my-func",
      "Condition": {
        "ArnLike": {"AWS:SourceArn": "arn:aws:s3:::my-bucket"}
      }
    }
  ]
}
```

---

## Error Handling & Retries

| Invocation Type | Retry Behavior |
|-----------------|----------------|
| Synchronous | No automatic retry (caller handles) |
| Asynchronous | 2 retries with delays |
| Event Source (stream) | Retry until success or data expires |
| Event Source (SQS) | Returns to queue, retry based on visibility timeout |

### Dead Letter Queue (DLQ)
```
Failed async invocation → DLQ (SQS or SNS)
```

### Destinations
```
Success → SQS, SNS, Lambda, EventBridge
Failure → SQS, SNS, Lambda, EventBridge
```

---

## Monitoring

### CloudWatch Metrics
- Invocations, Errors, Duration
- Throttles, ConcurrentExecutions
- IteratorAge (for streams)

### CloudWatch Logs
```
/aws/lambda/<function-name>
```

### X-Ray Tracing
```python
from aws_xray_sdk.core import patch_all
patch_all()  # Instrument AWS SDK calls
```

---

## CLI Quick Reference

```bash
# Create function
aws lambda create-function \
  --function-name my-function \
  --runtime python3.11 \
  --role arn:aws:iam::123456789:role/lambda-role \
  --handler index.handler \
  --zip-file fileb://function.zip

# Invoke
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  output.json

# Update code
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# View logs
aws logs tail /aws/lambda/my-function --follow

# Set concurrency
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 100
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Requests | $0.20 per 1M requests |
| Duration | $0.0000166667 per GB-second |
| Provisioned Concurrency | $0.000004646 per GB-second |
| Free tier | 1M requests, 400,000 GB-seconds/month |

### Cost Formula
```
Cost = (Requests × $0.20/1M) + (GB-seconds × $0.0000166667)

Example: 1M requests, 1GB memory, 500ms avg
= $0.20 + (1,000,000 × 1 × 0.5 × $0.0000166667)
= $0.20 + $8.33 = $8.53
```

---

## Best Practices

1. **Keep functions small and focused** - single responsibility
2. **Minimize package size** - faster cold starts
3. **Initialize outside handler** - reuse connections
4. **Use environment variables** - for configuration
5. **Set appropriate timeout** - not too high, not too low
6. **Use Provisioned Concurrency** - for latency-sensitive workloads
7. **Implement proper error handling** - with DLQ/Destinations
8. **Use Layers** - for shared dependencies
9. **Monitor with X-Ray** - for debugging
10. **Apply least privilege IAM** - minimal permissions

---

## Common Patterns

### API Backend
```
API Gateway → Lambda → DynamoDB
```

### Event Processing
```
S3 Upload → Lambda → Process → Store in DynamoDB
```

### Scheduled Tasks
```
EventBridge (cron) → Lambda → Cleanup/Reports
```

### Stream Processing
```
Kinesis/DynamoDB Streams → Lambda → Transform → S3/Redshift
```

### Fan-out
```
SNS → Lambda (multiple subscribers)
```

---

## Exam Tips

1. **15 min max timeout** - for longer tasks, use Step Functions or ECS
2. **Async invocation** - 2 automatic retries
3. **VPC Lambda** - needs NAT for internet, use VPC endpoints for AWS services
4. **Provisioned Concurrency** - eliminates cold starts
5. **Reserved Concurrency** - limits AND guarantees concurrency
6. **Layers** - max 5 per function, 250MB total unzipped
7. **Function URLs** - free alternative to API Gateway (basic features)
8. **X-Ray** - enable active tracing for debugging
9. **Environment variables** - 4KB limit, encrypted at rest
10. **Container images** - up to 10GB, ECR only
