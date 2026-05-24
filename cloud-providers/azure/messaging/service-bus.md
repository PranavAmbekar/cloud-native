# Azure Service Bus

> Enterprise message broker with queues and publish-subscribe topics for reliable cloud messaging.

## Overview

Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics. It decouples applications and services, providing reliable message delivery with features like FIFO guarantee, duplicate detection, and transactions.

## Key Concepts

| Term | Definition |
|------|------------|
| Namespace | Container for messaging components |
| Queue | Point-to-point messaging (one consumer) |
| Topic | Publish-subscribe messaging (many consumers) |
| Subscription | Topic subscriber with filter rules |
| Message | Data packet with properties and body |
| Dead-letter Queue | Failed/expired messages storage |
| Session | Ordered message group processing |

## Architecture

```
+------------------------------------------------------------------+
|                   Service Bus Namespace                          |
|                                                                  |
|  Queue (Point-to-Point)                                          |
|  +---------------------------------------------------------+    |
|  |  Producer --> [msg][msg][msg] --> Consumer              |    |
|  |               (FIFO order)                              |    |
|  +---------------------------------------------------------+    |
|                                                                  |
|  Topic (Publish-Subscribe)                                       |
|  +---------------------------------------------------------+    |
|  |  Publisher --> TOPIC                                    |    |
|  |                  |                                      |    |
|  |         +--------+--------+                             |    |
|  |         v        v        v                             |    |
|  |    +------+ +------+ +------+                           |    |
|  |    |Sub 1 | |Sub 2 | |Sub 3 | (filtered subscriptions)  |    |
|  |    +--+---+ +--+---+ +--+---+                           |    |
|  |       v        v        v                               |    |
|  |    Consumer Consumer Consumer                           |    |
|  +---------------------------------------------------------+    |
+------------------------------------------------------------------+
```

## Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Queues | Yes | Yes | Yes |
| Topics/Subscriptions | No | Yes | Yes |
| Message size | 256 KB | 256 KB | 100 MB |
| Throughput | Variable | Variable | Dedicated |
| VNet integration | No | No | Yes |
| Geo-DR | No | Yes | Yes |
| BYOK encryption | No | No | Yes |
| JMS support | No | No | Yes |

## Queues

### Create Queue

```bash
az servicebus queue create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --name myqueue \
  --max-size 5120 \
  --lock-duration PT5M \
  --default-message-time-to-live P14D
```

### Queue Properties

| Property | Description |
|----------|-------------|
| Max Size | 1-80 GB |
| Message TTL | Default message expiration |
| Lock Duration | How long message is locked for processing |
| Duplicate Detection | Time window for detecting duplicates |
| Dead-lettering | Enable dead-letter queue |
| Sessions | Enable ordered message groups |

### Send Message

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage

connection_str = "Endpoint=sb://mynamespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=..."

with ServiceBusClient.from_connection_string(connection_str) as client:
    with client.get_queue_sender("myqueue") as sender:
        message = ServiceBusMessage("Hello, World!")
        message.application_properties = {"priority": "high"}
        sender.send_messages(message)
```

### Receive Message

```python
from azure.servicebus import ServiceBusClient

with ServiceBusClient.from_connection_string(connection_str) as client:
    with client.get_queue_receiver("myqueue") as receiver:
        messages = receiver.receive_messages(max_message_count=10, max_wait_time=30)
        for msg in messages:
            print(str(msg))
            receiver.complete_message(msg)  # Remove from queue
```

## Topics and Subscriptions

### Create Topic and Subscription

```bash
# Create topic
az servicebus topic create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --name mytopic

# Create subscription
az servicebus topic subscription create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --topic-name mytopic \
  --name mysubscription

# Create subscription with filter
az servicebus topic subscription rule create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --topic-name mytopic \
  --subscription-name mysubscription \
  --name myrule \
  --filter-sql-expression "priority = 'high'"
```

### Subscription Filters

| Filter Type | Description | Example |
|-------------|-------------|---------|
| **SQL Filter** | SQL-like WHERE clause | `priority = 'high'` |
| **Correlation Filter** | Match on properties | `CorrelationId = 'order-123'` |
| **True Filter** | Match all messages | Default |
| **False Filter** | Match no messages | Pause subscription |

```python
# SQL Filter examples
"color = 'blue'"
"priority > 5"
"subject LIKE 'order%'"
"NOT EXISTS (user.premium)"
"category IN ('electronics', 'clothing')"
```

### Publish to Topic

```python
with ServiceBusClient.from_connection_string(connection_str) as client:
    with client.get_topic_sender("mytopic") as sender:
        message = ServiceBusMessage("Order created")
        message.subject = "orders"
        message.application_properties = {"priority": "high", "region": "west"}
        sender.send_messages(message)
```

## Message Properties

| Property | Description |
|----------|-------------|
| MessageId | Unique identifier |
| Subject/Label | Message category |
| ContentType | MIME type |
| CorrelationId | For request-reply |
| TimeToLive | Message expiration |
| ScheduledEnqueueTime | Delayed delivery |
| SessionId | Session identifier |
| PartitionKey | Partition assignment |
| ReplyTo | Reply queue/topic |

## Receive Modes

| Mode | Behavior |
|------|----------|
| **Peek Lock** | Lock message, complete/abandon after processing |
| **Receive and Delete** | Delete immediately on receive |

```python
# Peek Lock (default) - safer
receiver = client.get_queue_receiver("myqueue", receive_mode=ServiceBusReceiveMode.PEEK_LOCK)
msg = receiver.receive_messages()[0]
try:
    process(msg)
    receiver.complete_message(msg)  # Success
except:
    receiver.abandon_message(msg)   # Return to queue

# Receive and Delete - faster
receiver = client.get_queue_receiver("myqueue", receive_mode=ServiceBusReceiveMode.RECEIVE_AND_DELETE)
```

## Dead-Letter Queue

```
Queue: myqueue
+-- Active messages
+-- Dead-letter Queue: myqueue/$deadletterqueue
    +-- Expired messages
    +-- Max delivery count exceeded
    +-- Explicit dead-lettering

Access DLQ:
receiver = client.get_queue_receiver("myqueue", sub_queue=ServiceBusSubQueue.DEAD_LETTER)
```

### Reasons for Dead-lettering

| Reason | Description |
|--------|-------------|
| MaxDeliveryCountExceeded | Message delivered too many times |
| TTLExpired | Message expired before delivery |
| HeaderSizeExceeded | Headers too large |
| FilterException | Filter evaluation failed |
| Explicit | Application dead-lettered |

## Sessions (Ordered Processing)

```
Session "order-123":
[msg1] -> [msg2] -> [msg3] -> Consumer (processes in order)

Session "order-456":
[msg4] -> [msg5] -> Consumer (different session)
```

```python
# Send session message
message = ServiceBusMessage("Step 1")
message.session_id = "order-123"
sender.send_messages(message)

# Receive session messages
session_receiver = client.get_queue_receiver(
    "myqueue",
    session_id="order-123"  # or NEXT_AVAILABLE_SESSION
)
```

## Scheduled Messages

```python
from datetime import datetime, timedelta

message = ServiceBusMessage("Future message")
scheduled_time = datetime.utcnow() + timedelta(hours=1)

# Schedule message
sequence_number = sender.schedule_messages(message, scheduled_time)

# Cancel scheduled message
sender.cancel_scheduled_messages(sequence_number)
```

## Transactions

```python
from azure.servicebus import ServiceBusClient

with ServiceBusClient.from_connection_string(connection_str) as client:
    with client.get_queue_sender("queue1") as sender:
        with sender:
            # Start transaction
            sender.send_messages(ServiceBusMessage("msg1"), timeout=10)
            sender.send_messages(ServiceBusMessage("msg2"), timeout=10)
            # Both sent or neither
```

## Geo-Disaster Recovery

```
Primary Namespace                Secondary Namespace
(West US)                        (East US)
+-----------------+              +-----------------+
| Queues/Topics   | -----------> | Queues/Topics   |
| (Active)        |   metadata   | (Standby)       |
+-----------------+    sync      +-----------------+
        |
   +----+----+
   |  Alias  |  --> Failover switches to secondary
   +---------+
```

## CLI Quick Reference

```bash
# Create namespace
az servicebus namespace create \
  --name mynamespace \
  --resource-group myRG \
  --location eastus \
  --sku Standard

# Create queue
az servicebus queue create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --name myqueue

# Create topic
az servicebus topic create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --name mytopic

# Create subscription
az servicebus topic subscription create \
  --namespace-name mynamespace \
  --resource-group myRG \
  --topic-name mytopic \
  --name mysubscription

# Get connection string
az servicebus namespace authorization-rule keys list \
  --namespace-name mynamespace \
  --resource-group myRG \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString

# Send test message
az servicebus queue message send \
  --namespace-name mynamespace \
  --resource-group myRG \
  --queue-name myqueue \
  --body "Test message"
```

## Comparison: Service Bus vs Storage Queues vs Event Hubs

| Feature | Service Bus | Storage Queues | Event Hubs |
|---------|-------------|----------------|------------|
| Protocol | AMQP, HTTP | HTTP | AMQP, HTTP, Kafka |
| Message size | 256 KB - 100 MB | 64 KB | 1 MB |
| Ordering | FIFO (sessions) | No guarantee | Per partition |
| Topics | Yes | No | No |
| Transactions | Yes | No | No |
| Dead-letter | Yes | No | No |
| Use case | Enterprise messaging | Simple queuing | Event streaming |

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **Queue vs Topic**: Queue = one consumer; Topic = many subscribers
2. **Sessions**: Required for FIFO ordering
3. **Dead-letter**: Auto for expired/failed messages
4. **Peek Lock**: Safer, message returned if processing fails
5. **Premium tier**: Required for VNet, large messages, JMS
6. **Duplicate detection**: Based on MessageId within time window
7. **Scheduled messages**: Send now, deliver later
8. **Correlation filters**: Faster than SQL filters
9. **Geo-DR**: Metadata sync only (not messages)
10. **Max delivery count**: Default 10, then dead-letter

## Gotchas

- Basic tier doesn't support topics
- Sessions must be enabled at queue/subscription creation
- Dead-letter messages don't expire automatically
- Duplicate detection has max 7-day window
- Peek Lock has max 5-minute lock duration
- Scheduled messages count toward queue size
- Geo-DR syncs metadata only, not messages
- Premium tier has different pricing model (per messaging unit)
- Message ordering not guaranteed without sessions
- Large messages (> 256 KB) require Premium tier

## Limits

| Resource | Basic | Standard | Premium |
|----------|-------|----------|---------|
| Namespace quota | 100 | 100 | 1,000 |
| Queues/Topics per namespace | 10,000 | 10,000 | 1,000 |
| Message size | 256 KB | 256 KB | 100 MB |
| Queue/Topic size | 1-80 GB | 1-80 GB | 1-80 GB |
| Concurrent connections | 100 | 1,000 | 100,000 |
| Subscriptions per topic | N/A | 2,000 | 2,000 |
| Rules per subscription | N/A | 2,000 | 2,000 |
| Message TTL | 14 days | 14 days | Unlimited |
