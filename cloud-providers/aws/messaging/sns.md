# Amazon SNS (Simple Notification Service)

> Fully managed pub/sub messaging service for event-driven architectures.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Topic | Communication channel |
| Publisher | Sends messages to topic |
| Subscriber | Receives messages from topic |
| Subscription | Endpoint connection to topic |
| Message | Data sent to topic (up to 256 KB) |
| Fan-out | One message to many subscribers |

---

## Architecture

```
                    ┌─────────────────┐
                    │    SNS Topic    │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    ┌─────────┐        ┌─────────┐        ┌─────────┐
    │   SQS   │        │ Lambda  │        │  HTTP   │
    │  Queue  │        │         │        │Endpoint │
    └─────────┘        └─────────┘        └─────────┘
         │                   │                   │
         ▼                   ▼                   ▼
    [Consumer]          [Process]          [Webhook]
```

---

## Topic Types

### Standard Topics
| Feature | Value |
|---------|-------|
| Throughput | Unlimited |
| Ordering | Best effort |
| Delivery | At least once |
| Subscribers | All types |

### FIFO Topics
| Feature | Value |
|---------|-------|
| Throughput | 300 msg/s (3000 batched) |
| Ordering | Strict (per message group) |
| Delivery | Exactly once |
| Subscribers | SQS FIFO only |
| Name | Must end in `.fifo` |

---

## Supported Subscribers

| Endpoint Type | Use Case |
|---------------|----------|
| SQS | Queue for processing |
| Lambda | Serverless processing |
| HTTP/HTTPS | Webhooks, external services |
| Email | Notifications |
| Email-JSON | Structured email |
| SMS | Text messages |
| Mobile Push | iOS, Android, Fire OS |
| Kinesis Data Firehose | Streaming to S3, Redshift |

---

## Message Flow

```
Publisher                SNS Topic              Subscribers
    │                        │                       │
    │   1. Publish           │                       │
    │──────────────────────▶ │                       │
    │                        │   2. Fan-out          │
    │                        │──────────────────────▶│ SQS
    │                        │──────────────────────▶│ Lambda
    │                        │──────────────────────▶│ HTTP
    │                        │──────────────────────▶│ Email
    │                        │                       │
```

---

## Message Filtering

Subscribers receive only messages they want.

```json
// Subscription Filter Policy
{
  "store": ["example_corp"],
  "event": ["order_placed"],
  "customer_interests": ["rugby", "football"]
}
```

```json
// Message Attributes
{
  "store": {"Type": "String", "Value": "example_corp"},
  "event": {"Type": "String", "Value": "order_placed"},
  "customer_interests": {"Type": "String.Array", "Value": "[\"rugby\", \"tennis\"]"}
}
```

### Filter Operators
| Operator | Example |
|----------|---------|
| Exact match | `"event": ["order_placed"]` |
| Prefix | `"event": [{"prefix": "order_"}]` |
| Numeric | `"price": [{"numeric": [">", 100]}]` |
| Exists | `"customer": [{"exists": true}]` |
| OR logic | `"event": ["placed", "shipped"]` |

---

## Fan-Out Pattern

### SNS + SQS
```
             ┌─────────────────┐
             │    SNS Topic    │
             └────────┬────────┘
                      │
      ┌───────────────┼───────────────┐
      │               │               │
      ▼               ▼               ▼
 ┌─────────┐    ┌─────────┐    ┌─────────┐
 │  SQS 1  │    │  SQS 2  │    │  SQS 3  │
 │ Orders  │    │Analytics│    │  Audit  │
 └─────────┘    └─────────┘    └─────────┘
```

Benefits:
- Decoupled processing
- Independent scaling
- Different processing rates
- Retry per queue

### SNS + Kinesis Firehose
```
SNS Topic → Kinesis Firehose → S3 / Redshift / OpenSearch
```

For analytics and archival.

---

## Message Delivery

### Retry Policy (HTTP/HTTPS)
| Phase | Retries | Interval |
|-------|---------|----------|
| Immediate | 3 | 0 seconds |
| Pre-backoff | 2 | 1 second |
| Backoff | 10 | 1-20 seconds |
| Post-backoff | 38 | 20 seconds |

Total: 23 hours of retries.

### Dead Letter Queue
```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789:my-dlq"
}
```

Messages that fail delivery go to DLQ.

---

## Message Attributes

```python
import boto3

sns = boto3.client('sns')

sns.publish(
    TopicArn='arn:aws:sns:us-east-1:123456789:my-topic',
    Message='Order placed',
    MessageAttributes={
        'event': {
            'DataType': 'String',
            'StringValue': 'order_placed'
        },
        'store_id': {
            'DataType': 'Number',
            'StringValue': '12345'
        }
    }
)
```

---

## Message Structure

### Single message to all
```python
sns.publish(
    TopicArn=topic_arn,
    Message='Hello everyone!'
)
```

### Protocol-specific messages
```python
sns.publish(
    TopicArn=topic_arn,
    Message=json.dumps({
        'default': 'Default message',
        'email': 'Email-specific message',
        'sqs': 'SQS-specific message',
        'lambda': 'Lambda-specific message'
    }),
    MessageStructure='json'
)
```

---

## Raw Message Delivery

Skip SNS metadata wrapper for SQS/HTTP.

### Without Raw Delivery
```json
{
  "Type": "Notification",
  "MessageId": "xxx",
  "TopicArn": "arn:aws:sns:...",
  "Message": "Your actual message",
  "Timestamp": "...",
  ...
}
```

### With Raw Delivery
```json
"Your actual message"
```

Enable per subscription for cleaner payloads.

---

## Mobile Push Notifications

```
                  ┌─────────┐
                  │   SNS   │
                  │Platform │
                  │  App    │
                  └────┬────┘
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
   ┌───────┐      ┌───────┐      ┌───────┐
   │  APNs │      │  FCM  │      │  ADM  │
   │(Apple)│      │(Google│      │(Amazon│
   │       │      │       │      │       │
   └───────┘      └───────┘      └───────┘
       │               │               │
       ▼               ▼               ▼
    iPhone          Android        Kindle
```

Supported platforms:
- APNs (Apple)
- FCM (Firebase/Google)
- ADM (Amazon)
- Baidu, WNS (Windows)

---

## Security

### Access Control
```json
// Topic Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "s3.amazonaws.com"},
    "Action": "sns:Publish",
    "Resource": "arn:aws:sns:us-east-1:123456789:my-topic",
    "Condition": {
      "ArnLike": {"aws:SourceArn": "arn:aws:s3:::my-bucket"}
    }
  }]
}
```

### Encryption
- At rest: AWS KMS
- In transit: HTTPS

### VPC Endpoints
- Private connectivity without internet

---

## FIFO Topics

Ordered, exactly-once delivery.

```python
sns.publish(
    TopicArn='arn:aws:sns:us-east-1:123456789:my-topic.fifo',
    Message='Order 1',
    MessageGroupId='order-group-1',
    MessageDeduplicationId='unique-id-1'  # or content-based
)
```

| Feature | Requirement |
|---------|-------------|
| Subscribers | SQS FIFO queues only |
| Topic name | Must end in `.fifo` |
| Message Group ID | Required |
| Deduplication | Required (ID or content-based) |

---

## CLI Quick Reference

```bash
# Create topic
aws sns create-topic --name my-topic

# Create FIFO topic
aws sns create-topic \
  --name my-topic.fifo \
  --attributes FifoTopic=true

# Subscribe SQS
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:my-topic \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789:my-queue

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:my-topic \
  --protocol email \
  --notification-endpoint user@example.com

# Publish message
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789:my-topic \
  --message "Hello World"

# Publish with attributes
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789:my-topic \
  --message "Order placed" \
  --message-attributes '{"event":{"DataType":"String","StringValue":"order_placed"}}'

# List subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456789:my-topic

# Set filter policy
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789:my-topic:xxx \
  --attribute-name FilterPolicy \
  --attribute-value '{"event":["order_placed"]}'

# Delete topic
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:123456789:my-topic
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Publishes | $0.50 per 1M (Standard) |
| | $0.50 per 1M (FIFO) |
| Deliveries | Varies by protocol |
| | SQS: Free |
| | HTTP: $0.60 per 1M |
| | Email: $2.00 per 100K |
| | SMS: Varies by country |
| Data transfer | Standard rates |

---

## Limits

| Resource | Limit |
|----------|-------|
| Message size | 256 KB |
| Topics per account | 100,000 |
| Subscriptions per topic | 12,500,000 |
| Filter policies per topic | 200 |
| FIFO throughput | 300 msg/s (3000 batched) |

---

## Common Patterns

### Event Notification
```
S3 Event → SNS → Lambda, SQS, Email
```

### Application Integration
```
App A → SNS → App B, App C, App D
```

### Alert System
```
CloudWatch Alarm → SNS → Email, SMS, Slack (via Lambda)
```

---

## Exam Tips

1. **Fan-out** - one message to multiple subscribers
2. **Push model** - SNS pushes to subscribers
3. **Message filtering** - subscribers get only matching messages
4. **No persistence** - messages not stored (unlike SQS)
5. **FIFO topics** - SQS FIFO subscribers only
6. **DLQ support** - for failed HTTP/Lambda deliveries
7. **Raw delivery** - skip SNS wrapper for cleaner messages
8. **Cross-account** - use topic policy for permissions
9. **Mobile push** - supports APNs, FCM, ADM
10. **SNS + SQS** - common pattern for fan-out with queuing
11. **Filter policy** - JSON policy on subscription
12. **Message attributes** - used for filtering, up to 10
