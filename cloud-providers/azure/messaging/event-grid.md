# Azure Event Grid

> Fully managed event routing service for building event-driven architectures with pub-sub semantics.

## Overview

Azure Event Grid is a fully managed event routing service that enables event-driven, reactive programming. It uses a publish-subscribe model to route events from Azure services, custom applications, and third-party services to handlers.

## Key Concepts

| Term | Definition |
|------|------------|
| Event | What happened (small notification) |
| Event Source | Where the event originated |
| Topic | Endpoint for receiving events |
| Event Subscription | Route events to handlers |
| Event Handler | Where events are processed |
| Schema | Structure of event data |

## Architecture

```
+------------------------------------------------------------------+
|                     Event Sources                                |
|  +---------+ +---------+ +---------+ +---------+ +---------+     |
|  | Storage | | Service | |  Event  | | Custom  | |   IoT   |     |
|  | Account | |   Bus   | |   Hub   | |  Topic  | |   Hub   |     |
|  +----+----+ +----+----+ +----+----+ +----+----+ +----+----+     |
+-------|-----------|-----------|-----------|-----------|---------+
        |           |           |           |           |
        +-----------+-----------+-----+-----+-----------+
                                      |
                               +------v------+
                               | Event Grid  |
                               |   Topic     |
                               +------+------+
                                      |
           +--------------------------+-------------------------+
           |                          |                         |
    +------v------+           +------v------+           +------v------+
    |Subscription |           |Subscription |           |Subscription |
    |  (Filter)   |           |  (Filter)   |           |  (Filter)   |
    +------+------+           +------+------+           +------+------+
           |                          |                         |
    +------v------+           +------v------+           +------v------+
    |  Function   |           |  Logic App  |           |   Webhook   |
    +-------------+           +-------------+           +-------------+
```

## Topic Types

| Type | Description | Events From |
|------|-------------|-------------|
| **System Topic** | Built-in Azure resource events | Azure services |
| **Custom Topic** | Your own events | Custom applications |
| **Partner Topic** | Third-party SaaS events | Auth0, Datadog, etc. |
| **Domain** | Group of custom topics | Multi-tenant scenarios |

## Event Sources (Built-in)

| Source | Event Types |
|--------|-------------|
| **Storage** | Blob created, deleted |
| **Resource Groups** | Resource created, deleted, updated |
| **Service Bus** | Active messages, deadletter |
| **Event Hubs** | Capture file created |
| **Container Registry** | Image pushed, deleted |
| **Key Vault** | Secret expiring, changed |
| **App Configuration** | Key-value modified |
| **IoT Hub** | Device created, deleted, telemetry |

## Event Handlers

| Handler | Description |
|---------|-------------|
| **Azure Functions** | Serverless processing |
| **Logic Apps** | Workflow automation |
| **Event Hubs** | Event streaming |
| **Service Bus** | Queue/topic messaging |
| **Storage Queue** | Simple queuing |
| **Webhooks** | HTTP endpoints |
| **Hybrid Connections** | On-premises delivery |

## Event Schema

### Event Grid Schema (Default)

```json
{
  "topic": "/subscriptions/.../storageAccounts/mystorageaccount",
  "subject": "/blobServices/default/containers/mycontainer/blobs/myfile.txt",
  "eventType": "Microsoft.Storage.BlobCreated",
  "eventTime": "2024-01-15T10:30:00.0000000Z",
  "id": "unique-event-id",
  "data": {
    "api": "PutBlob",
    "contentType": "text/plain",
    "contentLength": 1024,
    "blobType": "BlockBlob",
    "url": "https://mystorageaccount.blob.core.windows.net/mycontainer/myfile.txt"
  },
  "dataVersion": "1.0",
  "metadataVersion": "1"
}
```

### Cloud Events Schema

```json
{
  "specversion": "1.0",
  "type": "Microsoft.Storage.BlobCreated",
  "source": "/subscriptions/.../storageAccounts/mystorageaccount",
  "subject": "/blobServices/default/containers/mycontainer/blobs/myfile.txt",
  "id": "unique-event-id",
  "time": "2024-01-15T10:30:00.0000000Z",
  "datacontenttype": "application/json",
  "data": {
    "api": "PutBlob",
    "contentType": "text/plain"
  }
}
```

## Create System Topic

```bash
# Create system topic for storage account
az eventgrid system-topic create \
  --name mystoragesystemtopic \
  --resource-group myRG \
  --source /subscriptions/.../storageAccounts/mystorageaccount \
  --topic-type Microsoft.Storage.StorageAccounts \
  --location eastus

# Create subscription
az eventgrid system-topic event-subscription create \
  --name mysubscription \
  --system-topic-name mystoragesystemtopic \
  --resource-group myRG \
  --endpoint-type azurefunction \
  --endpoint /subscriptions/.../functions/ProcessBlobEvent
```

## Create Custom Topic

```bash
# Create custom topic
az eventgrid topic create \
  --name mycustomtopic \
  --resource-group myRG \
  --location eastus

# Get endpoint and key
az eventgrid topic show --name mycustomtopic --resource-group myRG --query endpoint
az eventgrid topic key list --name mycustomtopic --resource-group myRG

# Create subscription
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.azurewebsites.net/api/events
```

## Publish Custom Events

```python
from azure.eventgrid import EventGridPublisherClient
from azure.core.credentials import AzureKeyCredential
from azure.eventgrid import EventGridEvent
from datetime import datetime

endpoint = "https://mycustomtopic.eastus-1.eventgrid.azure.net/api/events"
key = "your-topic-key"

client = EventGridPublisherClient(endpoint, AzureKeyCredential(key))

event = EventGridEvent(
    event_type="MyApp.Order.Created",
    subject="/orders/12345",
    data={
        "orderId": "12345",
        "customer": "John Doe",
        "total": 99.99
    },
    data_version="1.0"
)

client.send([event])
```

## Event Filtering

### Event Type Filtering

```bash
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.com/api/orders \
  --included-event-types MyApp.Order.Created MyApp.Order.Updated
```

### Subject Filtering

```bash
# Prefix filter
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.com/api/images \
  --subject-begins-with /images/

# Suffix filter
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.com/api/jpg \
  --subject-ends-with .jpg
```

### Advanced Filtering

```bash
# Filter on data properties
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.com/api/premium \
  --advanced-filter data.priority StringIn high critical
```

### Advanced Filter Operators

| Operator | Description |
|----------|-------------|
| NumberIn | Number in list |
| NumberNotIn | Number not in list |
| NumberLessThan | < |
| NumberGreaterThan | > |
| StringIn | String in list |
| StringNotIn | String not in list |
| StringBeginsWith | Prefix match |
| StringEndsWith | Suffix match |
| StringContains | Contains substring |
| BoolEquals | Boolean match |
| IsNullOrUndefined | Null check |

## Event Domains

Multi-tenant event publishing.

```
Event Domain: contoso-domain
+-- Topic: tenant-1
|   +-- Subscription -> tenant1-handler
+-- Topic: tenant-2
|   +-- Subscription -> tenant2-handler
+-- Topic: tenant-3
    +-- Subscription -> tenant3-handler
```

```bash
# Create domain
az eventgrid domain create \
  --name myeventdomain \
  --resource-group myRG \
  --location eastus

# Publish to domain topic
endpoint="https://myeventdomain.eastus-1.eventgrid.azure.net/api/events"
# Include "topic" in event for routing
```

## Dead-Lettering and Retry

```
Event --> Handler (fail) --> Retry (exponential backoff)
                              | (after max retries)
                              v
                        Dead-letter Storage
```

### Retry Policy

| Setting | Default | Range |
|---------|---------|-------|
| Max delivery attempts | 30 | 1-30 |
| Event TTL | 1440 min (24 hr) | 1-1440 |
| Retry delay | 30 seconds | Exponential backoff |

```bash
# Configure dead-letter and retry
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.com/api/events \
  --deadletter-endpoint /subscriptions/.../storageAccounts/mystorageaccount/blobServices/default/containers/deadletter \
  --max-delivery-attempts 10 \
  --event-ttl 60
```

## Webhook Validation

Event Grid validates webhook endpoints.

### Synchronous Validation

```json
// Event Grid sends:
{
  "validationCode": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6",
  "validationUrl": "https://..."
}

// Your endpoint returns:
{
  "validationResponse": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6"
}
```

### Asynchronous Validation

```python
# Manual validation via GET to validationUrl
# Used when endpoint can't respond synchronously
```

## Azure Function Handler

```csharp
// Azure Functions binding
[Function("ProcessEvent")]
public void Run([EventGridTrigger] EventGridEvent eventGridEvent)
{
    _logger.LogInformation($"Event Type: {eventGridEvent.EventType}");
    _logger.LogInformation($"Subject: {eventGridEvent.Subject}");
    _logger.LogInformation($"Data: {eventGridEvent.Data}");
}
```

```python
# Python Azure Function
import azure.functions as func
import logging

def main(event: func.EventGridEvent):
    logging.info(f"Event Type: {event.event_type}")
    logging.info(f"Subject: {event.subject}")
    logging.info(f"Data: {event.get_json()}")
```

## CLI Quick Reference

```bash
# List system topic types
az eventgrid topic-type list --output table

# Create system topic
az eventgrid system-topic create \
  --name mysystemtopic \
  --resource-group myRG \
  --source /subscriptions/.../storageAccounts/mysa \
  --topic-type Microsoft.Storage.StorageAccounts

# Create custom topic
az eventgrid topic create \
  --name mycustomtopic \
  --resource-group myRG

# Create subscription to Function
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint-type azurefunction \
  --endpoint /subscriptions/.../functions/MyFunction

# Create subscription to webhook
az eventgrid event-subscription create \
  --name mysubscription \
  --source-resource-id /subscriptions/.../topics/mycustomtopic \
  --endpoint https://myapp.com/api/events

# List subscriptions
az eventgrid event-subscription list \
  --source-resource-id /subscriptions/.../topics/mycustomtopic

# Get topic key
az eventgrid topic key list --name mycustomtopic --resource-group myRG
```

## Comparison: Event Grid vs Event Hubs vs Service Bus

| Feature | Event Grid | Event Hubs | Service Bus |
|---------|------------|------------|-------------|
| Pattern | Push (reactive) | Pull (stream) | Push/Pull (message) |
| Message size | 1 MB | 1 MB | 256 KB - 100 MB |
| Throughput | Millions/sec | Millions/sec | Lower |
| Ordering | No guarantee | Per partition | FIFO (sessions) |
| Dead-letter | Yes | No | Yes |
| Use case | Events, reactions | Streaming, analytics | Messaging, workflows |

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **System vs Custom topics**: System = Azure resources; Custom = your events
2. **Push model**: Event Grid pushes to handlers
3. **At-least-once delivery**: Events may be delivered multiple times
4. **Filtering**: Event type, subject, advanced (data properties)
5. **Dead-letter**: Requires Storage account blob container
6. **Webhook validation**: Required for custom endpoints
7. **Event schemas**: Event Grid schema vs Cloud Events
8. **Retry policy**: Max 30 attempts, 24-hour TTL
9. **Event Domains**: Multi-tenant event publishing
10. **Partner Events**: Third-party integrations

## Gotchas

- Event size limited to 1 MB
- No guaranteed ordering of events
- At-least-once delivery (handlers must be idempotent)
- Webhook endpoints must validate subscription
- System topics auto-created or explicit creation depends on service
- Custom topic events require topic name in routing
- Dead-letter requires storage account in same region
- Advanced filters limited to 25 per subscription
- Event TTL maximum is 24 hours
- Retry backoff is exponential (30s, 60s, 120s...)

## Limits

| Resource | Limit |
|----------|-------|
| Topics per subscription | 100 |
| Event subscriptions per topic | 500 |
| Event subscriptions per system topic | 500 |
| Events per second per topic | 5,000 (can increase) |
| Event size | 1 MB |
| Custom topics per region | 100 |
| Domains per region | 100 |
| Topics per domain | 100,000 |
| Advanced filters per subscription | 25 |
| Filter values per filter | 25 |
| Batch publish | 1 MB total |
