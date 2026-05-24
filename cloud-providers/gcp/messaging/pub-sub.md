# Google Cloud Pub/Sub

> Scalable, real-time messaging service for asynchronous event-driven architectures.

## Overview

Cloud Pub/Sub is an asynchronous messaging service that decouples services producing events from services processing events. It's designed for high-durability and at-least-once delivery at scale.

## Key Concepts

| Term | Definition |
|------|------------|
| Topic | Named resource for publishing messages |
| Subscription | Named resource for consuming messages |
| Message | Data published to a topic |
| Publisher | Client that sends messages |
| Subscriber | Client that receives messages |
| Acknowledgment | Confirmation of message processing |

## Architecture

```
+---------------------------------------------------------------+
|                           Pub/Sub                             |
|                                                               |
|  Publishers                         Subscribers               |
|  +-----+ +-----+                   +-----------------------+  |
|  |App 1| |App 2|                   | Push Subscription     |  |
|  +--+--+ +--+--+                   |  -> Cloud Run         |  |
|     |       |                      |  -> Cloud Functions   |  |
|     +---+---+                      |  -> HTTP endpoint     |  |
|         |                          +-----------------------+  |
|         v                                                     |
|  +--------------+                  +-----------------------+  |
|  |    Topic     |----------------->| Pull Subscription     |  |
|  |  (messages)  |                  |  <- Subscriber pulls  |  |
|  +--------------+                  |  <- Dataflow          |  |
|                                    |  <- Compute Engine    |  |
|                                    +-----------------------+  |
|                                                               |
|                                    +-----------------------+  |
|                                    | BigQuery Subscription |  |
|                                    |  -> Direct to BigQuery|  |
|                                    +-----------------------+  |
+---------------------------------------------------------------+
```

## Topics

### Create Topic

```bash
# Create topic
gcloud pubsub topics create my-topic

# Create with schema
gcloud pubsub topics create my-topic \
  --schema=my-schema \
  --message-encoding=JSON

# Create with retention
gcloud pubsub topics create my-topic \
  --message-retention-duration=7d
```

### Topic Settings

| Setting | Description |
|---------|-------------|
| Message retention | Store messages (up to 31 days) |
| Schema | Enforce message structure |
| CMK encryption | Custom encryption key |
| Dead letter | Handle failed messages |

## Subscriptions

### Subscription Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Pull** | Subscriber pulls messages | Flexible processing |
| **Push** | Pub/Sub pushes to endpoint | Event-driven |
| **BigQuery** | Direct write to BigQuery | Analytics |
| **Cloud Storage** | Write to GCS | Archival |

### Create Subscription

```bash
# Pull subscription
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --ack-deadline=60

# Push subscription
gcloud pubsub subscriptions create my-push-sub \
  --topic=my-topic \
  --push-endpoint="https://my-service.run.app/push" \
  --push-auth-service-account=push-sa@my-project.iam.gserviceaccount.com

# BigQuery subscription
gcloud pubsub subscriptions create my-bq-sub \
  --topic=my-topic \
  --bigquery-table=my-project:dataset.table

# With dead letter topic
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --dead-letter-topic=dead-letter-topic \
  --max-delivery-attempts=5
```

### Subscription Settings

| Setting | Description | Default |
|---------|-------------|---------|
| Ack deadline | Time to acknowledge | 10 seconds |
| Message retention | How long to keep unacked | 7 days |
| Expiration | Subscription expiry if unused | 31 days |
| Filter | Message attribute filter | None |
| Ordering | Enable message ordering | Disabled |
| Exactly once | Exactly-once delivery | Disabled |

## Publishing Messages

### gcloud

```bash
# Publish message
gcloud pubsub topics publish my-topic --message="Hello, World!"

# With attributes
gcloud pubsub topics publish my-topic \
  --message="Order created" \
  --attribute="type=order,priority=high"

# With ordering key
gcloud pubsub topics publish my-topic \
  --message="Event 1" \
  --ordering-key="user-123"
```

### Python SDK

```python
from google.cloud import pubsub_v1
from concurrent import futures

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "my-topic")

# Simple publish
future = publisher.publish(topic_path, b"Hello, World!")
message_id = future.result()
print(f"Published message ID: {message_id}")

# With attributes
future = publisher.publish(
    topic_path,
    b"Order created",
    type="order",
    priority="high"
)

# Batch publishing
def callback(future):
    message_id = future.result()
    print(f"Published: {message_id}")

for i in range(100):
    future = publisher.publish(topic_path, f"Message {i}".encode())
    future.add_done_callback(callback)

# With ordering
publisher = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(
        enable_message_ordering=True
    )
)
publisher.publish(topic_path, b"Event 1", ordering_key="user-123")
```

## Receiving Messages

### Pull (Synchronous)

```python
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "my-sub")

# Pull messages
response = subscriber.pull(
    request={"subscription": subscription_path, "max_messages": 10}
)

for msg in response.received_messages:
    print(f"Data: {msg.message.data.decode()}")
    print(f"Attributes: {msg.message.attributes}")

# Acknowledge
ack_ids = [msg.ack_id for msg in response.received_messages]
subscriber.acknowledge(request={"subscription": subscription_path, "ack_ids": ack_ids})
```

### Pull (Streaming - Recommended)

```python
from google.cloud import pubsub_v1
from concurrent.futures import TimeoutError

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "my-sub")

def callback(message):
    print(f"Received: {message.data.decode()}")
    print(f"Attributes: {message.attributes}")
    message.ack()  # Acknowledge

# Start streaming
streaming_pull_future = subscriber.subscribe(subscription_path, callback=callback)

try:
    streaming_pull_future.result(timeout=60)  # Block for 60 seconds
except TimeoutError:
    streaming_pull_future.cancel()
    streaming_pull_future.result()
```

### Push (HTTP Endpoint)

```python
# Cloud Function or Cloud Run handler
from flask import Flask, request
import base64

app = Flask(__name__)

@app.route('/push', methods=['POST'])
def handle_push():
    envelope = request.get_json()
    if not envelope:
        return 'Bad Request', 400

    message = envelope['message']
    data = base64.b64decode(message['data']).decode()
    attributes = message.get('attributes', {})

    print(f"Received: {data}")
    print(f"Attributes: {attributes}")

    return 'OK', 200
```

## Message Ordering

```python
# Publisher with ordering
publisher = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(
        enable_message_ordering=True
    )
)

# All messages with same ordering key delivered in order
publisher.publish(topic_path, b"Event 1", ordering_key="user-123")
publisher.publish(topic_path, b"Event 2", ordering_key="user-123")
publisher.publish(topic_path, b"Event 3", ordering_key="user-123")
```

```bash
# Enable ordering on subscription
gcloud pubsub subscriptions create my-ordered-sub \
  --topic=my-topic \
  --enable-message-ordering
```

## Filtering

```bash
# Create subscription with filter
gcloud pubsub subscriptions create filtered-sub \
  --topic=my-topic \
  --filter='attributes.type = "order"'

# Multiple conditions
gcloud pubsub subscriptions create filtered-sub \
  --topic=my-topic \
  --filter='attributes.priority = "high" AND attributes.region = "us"'
```

### Filter Syntax

```
attributes.type = "order"
attributes:type                    # Has attribute
NOT attributes:type                # Missing attribute
attributes.priority = "high"
hasPrefix(attributes.region, "us")
```

## Dead Letter Topics

```bash
# Create dead letter topic
gcloud pubsub topics create dead-letter

# Create subscription with dead letter
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --dead-letter-topic=dead-letter \
  --max-delivery-attempts=5

# Grant Pub/Sub service account access
gcloud pubsub topics add-iam-policy-binding dead-letter \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

## Schemas

```bash
# Create Avro schema
gcloud pubsub schemas create my-schema \
  --type=AVRO \
  --definition='{
    "type": "record",
    "name": "Order",
    "fields": [
      {"name": "order_id", "type": "string"},
      {"name": "amount", "type": "double"}
    ]
  }'

# Create topic with schema
gcloud pubsub topics create orders \
  --schema=my-schema \
  --message-encoding=JSON
```

## Exactly-Once Delivery

```bash
# Enable exactly-once on subscription
gcloud pubsub subscriptions create exactly-once-sub \
  --topic=my-topic \
  --enable-exactly-once-delivery
```

## CLI Quick Reference

```bash
# Create topic
gcloud pubsub topics create my-topic

# Create subscription
gcloud pubsub subscriptions create my-sub --topic=my-topic

# Publish message
gcloud pubsub topics publish my-topic --message="Hello"

# Pull messages
gcloud pubsub subscriptions pull my-sub --auto-ack

# List topics
gcloud pubsub topics list

# List subscriptions
gcloud pubsub subscriptions list

# Describe topic
gcloud pubsub topics describe my-topic

# Delete topic
gcloud pubsub topics delete my-topic

# Delete subscription
gcloud pubsub subscriptions delete my-sub

# Seek to time
gcloud pubsub subscriptions seek my-sub --time="2024-01-15T10:00:00Z"

# Create snapshot
gcloud pubsub snapshots create my-snapshot --subscription=my-sub
```

## Pricing

| Component | Cost |
|-----------|------|
| Message delivery | $40 per TiB |
| Seek operations | $0.10 per GiB |
| Snapshot storage | $0.026 per GiB/month |
| Topic message storage | $0.27 per GiB/month |
| Free tier | 10 GiB per month |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **At-least-once**: Default delivery guarantee
2. **Exactly-once**: Available, higher overhead
3. **Pull vs Push**: Pull for flexible, Push for event-driven
4. **Ack deadline**: Must ack within deadline
5. **Dead letter**: Handle poison messages
6. **Ordering**: Requires ordering key, per-key ordering
7. **Filtering**: Reduce unnecessary processing
8. **BigQuery subscription**: Direct streaming
9. **Message retention**: Up to 31 days on topic
10. **Seek**: Replay or skip messages

## Gotchas

- Messages may be delivered more than once (default)
- Ack deadline starts when message delivered
- Unacked messages redelivered after deadline
- Message retention is separate from subscription
- Ordering only guaranteed within same ordering key
- Push requires publicly accessible endpoint
- Schema changes require new revision
- Dead letter needs IAM permissions
- Filter changes require new subscription
- Exactly-once has higher latency

## Limits

| Resource | Limit |
|----------|-------|
| Topics per project | 10,000 |
| Subscriptions per topic | 10,000 |
| Subscriptions per project | 10,000 |
| Message size | 10 MB |
| Attributes per message | 100 |
| Attribute key size | 256 bytes |
| Attribute value size | 1024 bytes |
| Message retention | 31 days |
| Ack deadline | 10-600 seconds |
| Throughput per region | 1 GiB/s (can increase) |
