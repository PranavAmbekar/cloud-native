# Amazon SQS (Simple Queue Service)

> Fully managed message queuing service for decoupling and scaling microservices.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Queue | Buffer for messages |
| Message | Data (up to 256 KB) |
| Producer | Sends messages to queue |
| Consumer | Receives messages from queue |
| Visibility Timeout | Time message is hidden after receive |
| Dead Letter Queue | Queue for failed messages |
| Long Polling | Efficient polling (wait for messages) |

---

## Queue Types

### Standard Queue
```
Producer ──▶ [msg3][msg1][msg2] ──▶ Consumer
             (best-effort ordering)
```

| Feature | Standard |
|---------|----------|
| Throughput | Unlimited |
| Ordering | Best effort |
| Delivery | At least once |
| Duplicates | Possible |

### FIFO Queue
```
Producer ──▶ [msg1][msg2][msg3] ──▶ Consumer
             (exactly ordered)
```

| Feature | FIFO |
|---------|------|
| Throughput | 300 msg/s (3000 with batching) |
| Ordering | Guaranteed (per group) |
| Delivery | Exactly once |
| Duplicates | None |
| Name | Must end in `.fifo` |

---

## Message Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. SEND         2. RECEIVE       3. PROCESS      4. DELETE│
│                                                             │
│  Producer ──▶ Queue ──▶ Consumer ──▶ Process ──▶ Delete    │
│                  │                      │                   │
│                  │    Visibility        │                   │
│                  │◄───Timeout───────────┘                   │
│                  │    (hidden)                              │
│                  │                                          │
│                  ▼                                          │
│            If not deleted, message returns to queue         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Visibility Timeout

Time message is invisible after being received.

```
Message received at T=0
Visibility Timeout = 30 seconds

T=0  ──────────── T=30 ──────────── T=60
│                   │                 │
│   Message         │   Message       │
│   INVISIBLE       │   VISIBLE       │
│   (processing)    │   (reappears)   │
```

- Default: 30 seconds
- Range: 0 seconds - 12 hours
- Can extend with `ChangeMessageVisibility`

```python
# Extend visibility timeout
sqs.change_message_visibility(
    QueueUrl=queue_url,
    ReceiptHandle=receipt_handle,
    VisibilityTimeout=60  # extend by 60 seconds
)
```

---

## Long Polling vs Short Polling

### Short Polling (default)
- Returns immediately (even if empty)
- More API calls
- Higher cost

### Long Polling
- Waits for messages (up to 20 seconds)
- Fewer empty responses
- Lower cost

```python
# Long polling
response = sqs.receive_message(
    QueueUrl=queue_url,
    WaitTimeSeconds=20  # Long poll for 20 seconds
)
```

Enable at queue level or per request.

---

## Dead Letter Queue (DLQ)

Queue for messages that fail processing.

```
Main Queue                 Dead Letter Queue
┌─────────────┐           ┌─────────────┐
│  [message]  │──fail──▶  │  [message]  │
│             │  (after   │             │
│             │  N tries) │             │
└─────────────┘           └─────────────┘

maxReceiveCount = 3
→ After 3 failed receives, move to DLQ
```

### Redrive Policy
```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789:my-dlq",
  "maxReceiveCount": 3
}
```

### DLQ Redrive
Move messages back to source queue for reprocessing.

---

## Message Attributes

Metadata attached to messages.

```python
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='Order data',
    MessageAttributes={
        'OrderType': {
            'DataType': 'String',
            'StringValue': 'priority'
        },
        'OrderId': {
            'DataType': 'Number',
            'StringValue': '12345'
        }
    }
)
```

- Up to 10 attributes
- Name, Type, Value
- Used for filtering (with SNS)

---

## FIFO Features

### Message Group ID
```
Group A: [msg1] [msg3] [msg5]  → Ordered within group
Group B: [msg2] [msg4] [msg6]  → Ordered within group
```

Messages in same group processed in order.

### Deduplication ID
Prevents duplicate messages within 5-minute window.

```python
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='data',
    MessageGroupId='group-1',
    MessageDeduplicationId='unique-id-123'  # or enable content-based
)
```

### Content-Based Deduplication
- SHA-256 hash of message body
- Enable at queue level
- No need for DeduplicationId

---

## Delay Queues

Delay message delivery.

```
Message sent at T=0
Delay = 60 seconds

T=0 ──────────── T=60 ──────────── T=120
│                 │                  │
│   Message       │   Message        │
│   DELAYED       │   AVAILABLE      │
│   (invisible)   │   (can receive)  │
```

- Queue-level: 0-15 minutes default delay
- Message-level: Override with `DelaySeconds`

```python
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='delayed message',
    DelaySeconds=300  # 5 minutes
)
```

---

## Large Messages

Messages > 256 KB using S3.

```
┌──────────┐      ┌─────┐      ┌─────────┐
│ Producer │─────▶│ S3  │      │   SQS   │
│          │      │     │      │         │
│ (large   │      │ ┌───┴───┐  │ [ptr]   │
│  payload)│      │ │payload│  │         │
└──────────┘      │ └───────┘  └────┬────┘
                  │      ▲          │
                  │      │          │
                  │      └──────────┘
                  │        pointer
                  └─────────────────────────▶ Consumer
```

Use **Amazon SQS Extended Client Library**.

---

## Access Control

### IAM Policies
```json
{
  "Effect": "Allow",
  "Action": [
    "sqs:SendMessage",
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage"
  ],
  "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue"
}
```

### Queue Policies (Resource-based)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "sns.amazonaws.com"},
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:sns:us-east-1:123456789:my-topic"
      }
    }
  }]
}
```

---

## Encryption

### At Rest
- SSE-SQS (AWS managed)
- SSE-KMS (customer managed)

### In Transit
- HTTPS endpoints

```bash
# Enable SSE-KMS
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --attributes KmsMasterKeyId=alias/my-key
```

---

## Integration with Lambda

```
┌─────────┐    Event Source    ┌─────────┐
│   SQS   │───────Mapping─────▶│ Lambda  │
│  Queue  │                    │         │
└─────────┘                    └─────────┘
```

- Lambda polls SQS
- Batch size: 1-10,000 messages
- Lambda deletes on success
- Failed messages return to queue (or DLQ)

---

## CLI Quick Reference

```bash
# Create standard queue
aws sqs create-queue --queue-name my-queue

# Create FIFO queue
aws sqs create-queue \
  --queue-name my-queue.fifo \
  --attributes FifoQueue=true

# Send message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --message-body "Hello World"

# Receive message
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --wait-time-seconds 20

# Delete message
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --receipt-handle "AQEBwJn..."

# Purge queue
aws sqs purge-queue \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue

# Get queue attributes
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --attribute-names All
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Requests | $0.40 per 1M (Standard) |
| | $0.50 per 1M (FIFO) |
| Data transfer | Standard rates |
| Free tier | 1M requests/month |

Batch operations count as single request (up to 10 messages).

---

## Limits

| Resource | Limit |
|----------|-------|
| Message size | 256 KB |
| Message retention | 1 minute - 14 days (default 4 days) |
| Visibility timeout | 0 sec - 12 hours |
| Delay | 0 - 15 minutes |
| Long poll wait | 0 - 20 seconds |
| Batch size | 10 messages |
| In-flight messages | 120,000 (Standard), 20,000 (FIFO) |
| FIFO throughput | 300/s (3,000 with batching) |

---

## SQS vs SNS vs Kinesis

| Feature | SQS | SNS | Kinesis |
|---------|-----|-----|---------|
| Model | Queue | Pub/Sub | Stream |
| Consumers | Pull | Push | Pull |
| Retention | 14 days | None | 365 days |
| Ordering | FIFO available | No | Per shard |
| Replay | No | No | Yes |
| Throughput | Unlimited/300 | High | Per shard |

---

## Exam Tips

1. **Standard = unlimited throughput**, at-least-once, best-effort ordering
2. **FIFO = 300 msg/s** (3000 batched), exactly-once, ordered
3. **Visibility Timeout** - prevents reprocessing, extend if needed
4. **Long Polling** - reduces cost, set WaitTimeSeconds > 0
5. **DLQ** - configure maxReceiveCount for failed messages
6. **Delay Queues** - up to 15 minutes delay
7. **Message retention** - 1 min to 14 days
8. **Lambda integration** - event source mapping, auto-delete on success
9. **FIFO queue name** - must end in `.fifo`
10. **Message Group ID** - for ordering within FIFO
11. **Deduplication** - 5-minute window in FIFO
12. **Large messages** - use S3 with Extended Client Library
