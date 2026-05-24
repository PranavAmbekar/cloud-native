# Azure Cosmos DB

> Globally distributed, multi-model database service for any scale with guaranteed low latency.

## Overview

Azure Cosmos DB is a fully managed NoSQL database with single-digit millisecond response times, automatic scaling, and global distribution. It supports multiple data models and APIs.

## Key Concepts

| Term | Definition |
|------|------------|
| Account | Top-level resource containing databases |
| Database | Namespace for containers |
| Container | Collection of items (table, collection, graph) |
| Item | Individual document/record |
| Partition Key | Property used to distribute data |
| Request Unit (RU) | Currency for throughput (CPU, IOPS, memory) |
| Consistency Level | Balance between consistency and performance |

## Supported APIs

| API | Data Model | Use Case |
|-----|------------|----------|
| **NoSQL (Core)** | Document (JSON) | Most common, full feature access |
| **MongoDB** | Document (BSON) | MongoDB compatibility |
| **Cassandra** | Wide-column | Cassandra workloads |
| **Gremlin** | Graph | Graph relationships |
| **Table** | Key-value | Azure Table Storage migration |
| **PostgreSQL** | Relational | Distributed PostgreSQL |

## Architecture

```
+-------------------------------------------------------------+
|                      Cosmos DB Account                      |
|  +-------------------------------------------------------+  |
|  |                     Database                          |  |
|  |  +---------------------------------------------------+|  |
|  |  |                  Container                        ||  |
|  |  |  +-----------------------------------------------+||  |
|  |  |  | Logical Partition (partition key value)       |||  |
|  |  |  |   +------+  +------+  +------+                |||  |
|  |  |  |   | Item |  | Item |  | Item |                |||  |
|  |  |  |   +------+  +------+  +------+                |||  |
|  |  |  +-----------------------------------------------+||  |
|  |  +---------------------------------------------------+|  |
|  +-------------------------------------------------------+  |
|                                                             |
|  Physical Partitions: Automatic, managed by Cosmos DB      |
+-------------------------------------------------------------+
```

## Partition Key

Critical design decision affecting performance and scalability.

### Good Partition Key Properties

- High cardinality (many unique values)
- Even distribution of data
- Even distribution of requests
- Used in most queries

### Examples

| Scenario | Good Partition Key | Bad Partition Key |
|----------|-------------------|-------------------|
| E-commerce orders | /customerId | /orderDate |
| IoT telemetry | /deviceId | /status |
| Multi-tenant app | /tenantId | /region |
| Social media posts | /userId | /postType |

```json
// Example item with partition key
{
    "id": "order-123",
    "customerId": "cust-456",  // Partition key
    "items": [...],
    "total": 99.99
}
```

## Request Units (RU)

```
1 RU = Cost to read 1 KB item by ID

Operations cost more RUs:
- Point read (1 KB): 1 RU
- Write (1 KB): 5 RUs
- Query (depends on complexity)
- Cross-partition query: Higher cost
```

### RU Estimation

| Operation | Approximate RU Cost |
|-----------|---------------------|
| Point read (1 KB) | 1 RU |
| Point read (64 KB) | 64 RU |
| Write (1 KB) | ~5 RU |
| Write (64 KB) | ~64 RU |
| Simple query | 3-10 RU |
| Complex query | 10-100+ RU |

## Throughput Models

| Model | Description | Use Case |
|-------|-------------|----------|
| **Provisioned** | Fixed RU/s capacity | Predictable workloads |
| **Autoscale** | 10-100% of max RU/s | Variable workloads |
| **Serverless** | Per-request billing | Sporadic, dev/test |

### Throughput Options

```
Provisioned: 400 RU/s - unlimited (in 100 RU/s increments)
Autoscale:   1,000 - 1,000,000 RU/s (max you set, scales 10-100%)
Serverless:  No provisioning, pay per RU consumed

Database-level: Shared across containers (up to 25 containers)
Container-level: Dedicated to single container
```

## Consistency Levels

| Level | Description | Latency | Availability |
|-------|-------------|---------|--------------|
| **Strong** | Linearizable, global order | Higher | Lower |
| **Bounded Staleness** | Consistent within K versions or T time | Medium | Medium |
| **Session** | Consistent within a session | Low | High |
| **Consistent Prefix** | Reads never see out-of-order writes | Low | High |
| **Eventual** | No ordering guarantees | Lowest | Highest |

```
Strong <- Bounded Staleness <- Session <- Consistent Prefix <- Eventual
(Strongest)                                                (Weakest)

Default: Session (good balance)
Most common: Session or Eventual
```

## Global Distribution

```
+-------------+    +-------------+    +-------------+
|  East US    |<-->|  West US    |<-->|  Europe     |
|  (Primary)  |    |  (Replica)  |    |  (Replica)  |
+-------------+    +-------------+    +-------------+
       |                  |                  |
       +------------------+------------------+
                          |
              Multi-master writes or
              Single-write with failover
```

### Multi-Region Features

| Feature | Description |
|---------|-------------|
| **Automatic Failover** | Failover to replica if primary fails |
| **Manual Failover** | Trigger failover for testing/DR |
| **Multi-region Writes** | Write to any region (conflict resolution) |
| **Service-managed Failover** | Automatic failover priority |

## Indexing

### Default: Automatic indexing of all properties

```json
// Indexing policy
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {"path": "/*"}
    ],
    "excludedPaths": [
        {"path": "/description/*"}
    ],
    "compositeIndexes": [
        [
            {"path": "/name", "order": "ascending"},
            {"path": "/age", "order": "descending"}
        ]
    ]
}
```

### Index Types

| Type | Use Case |
|------|----------|
| **Range** | =, >, <, >=, <=, ORDER BY |
| **Spatial** | ST_DISTANCE, ST_WITHIN |
| **Composite** | Multi-property ORDER BY, complex filters |

## Change Feed

Stream of changes to process in real-time.

```
Container
    |
    v
Change Feed --> Azure Functions
            --> Stream processing
            --> Event-driven architecture
```

```csharp
// Azure Functions with Change Feed trigger
[Function("ProcessChanges")]
public void Run([CosmosDBTrigger(
    databaseName: "mydb",
    containerName: "items",
    Connection = "CosmosConnection",
    LeaseContainerName = "leases")] IReadOnlyList<Item> changes)
{
    foreach (var item in changes)
    {
        _logger.LogInformation($"Changed: {item.Id}");
    }
}
```

## Stored Procedures, Triggers, UDFs

```javascript
// Stored procedure (JavaScript, runs in partition)
function createItems(items) {
    var context = getContext();
    var container = context.getCollection();
    var response = context.getResponse();

    for (var i = 0; i < items.length; i++) {
        container.createDocument(container.getSelfLink(), items[i]);
    }

    response.setBody({created: items.length});
}

// Pre-trigger
function validateItem() {
    var item = getContext().getRequest().getBody();
    if (!item.timestamp) {
        item.timestamp = new Date().toISOString();
    }
    getContext().getRequest().setBody(item);
}
```

## SQL Query Examples (NoSQL API)

```sql
-- Basic query
SELECT * FROM c WHERE c.category = "electronics"

-- Projection
SELECT c.name, c.price FROM c WHERE c.price > 100

-- JOIN (within document)
SELECT c.name, t.name AS tagName
FROM c JOIN t IN c.tags

-- Aggregate
SELECT VALUE COUNT(1) FROM c WHERE c.status = "active"

-- Cross-partition query (uses more RUs)
SELECT * FROM c ORDER BY c.createdAt DESC
```

## CLI Quick Reference

```bash
# Create Cosmos DB account
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myRG \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1

# Create database
az cosmosdb sql database create \
  --account-name mycosmosaccount \
  --resource-group myRG \
  --name mydb

# Create container
az cosmosdb sql container create \
  --account-name mycosmosaccount \
  --resource-group myRG \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/customerId" \
  --throughput 400

# Enable autoscale
az cosmosdb sql container throughput migrate \
  --account-name mycosmosaccount \
  --resource-group myRG \
  --database-name mydb \
  --name mycontainer \
  --throughput-type autoscale

# Add region
az cosmosdb update \
  --name mycosmosaccount \
  --resource-group myRG \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1 \
  --locations regionName=northeurope failoverPriority=2
```

## Pricing Components

| Component | Cost Factor |
|-----------|-------------|
| **Throughput** | RU/s provisioned (hourly) |
| **Storage** | GB per month |
| **Regions** | Multiply by number of regions |
| **Multi-region writes** | 2x write RU cost |
| **Backup** | Free (periodic), paid (continuous) |

### Cost Estimation

```
Monthly cost example:
- 1000 RU/s provisioned
- 100 GB storage
- 2 regions
- Single-write

Throughput: $0.008/hr × 1000/100 × 730 hrs × 2 regions = ~$116
Storage: $0.25/GB × 100 GB × 2 regions = $50
Total: ~$166/month
```

## Exam Tips (AZ-204, AZ-305)

1. **Partition key**: Critical design decision, cannot change after creation
2. **Consistency levels**: Strong uses 2x RUs; Session is default and most common
3. **RU consumption**: Point reads are cheapest; cross-partition queries are expensive
4. **Change feed**: Process changes in real-time (like DynamoDB Streams)
5. **Serverless**: Max 5000 RU/s per container, no SLA
6. **Autoscale**: Scales between 10% and 100% of max RU/s
7. **Global distribution**: Add regions for read latency, multi-master for write availability
8. **Stored procedures**: Run in single partition only
9. **TTL**: Automatic deletion based on _ts field + ttl
10. **Synthetic partition key**: Combine properties for better distribution

## Gotchas

- Partition key cannot be changed after container creation
- Logical partition limit: 20 GB
- Stored procedures execute in single partition only
- Cross-partition queries consume more RUs
- Strong consistency requires 2x write RUs
- Serverless has no SLA and 5000 RU/s limit
- Change feed doesn't capture deletes (use soft delete)
- Item ID must be unique within a partition
- Multi-region writes require conflict resolution policy
- Minimum throughput: 400 RU/s (provisioned), 1000 RU/s (autoscale max)

## Limits

| Resource | Limit |
|----------|-------|
| Databases per account | Unlimited |
| Containers per database | Unlimited |
| Item size | 2 MB |
| Partition key value | 2 KB |
| Logical partition size | 20 GB |
| Stored procedure execution | 5 seconds |
| Throughput per container | Unlimited (provisioned) |
| Regions per account | Unlimited |
| Indexes per container | 500 |
| Composite index properties | 8 |
