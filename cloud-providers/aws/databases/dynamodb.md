# Amazon DynamoDB

> Fully managed NoSQL database with single-digit millisecond performance at any scale.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Table | Collection of items |
| Item | Single data record (like a row) |
| Attribute | Data element (like a column) |
| Primary Key | Unique identifier for items |
| Partition Key (PK) | Hash key for distribution |
| Sort Key (SK) | Optional range key for ordering |
| GSI | Global Secondary Index |
| LSI | Local Secondary Index |

---

## Data Model

### Primary Key Types

**Partition Key Only**
```
+-------------------------------------+
| Table: Users                        |
+-------------+-----------------------+
| user_id(PK) | name  | email         |
+-------------+-------+---------------+
| u001        | Alice | alice@ex.com  |
| u002        | Bob   | bob@ex.com    |
+-------------+-------+---------------+
```

**Partition Key + Sort Key (Composite)**
```
+--------------------------------------------------+
| Table: Orders                                    |
+-----------+-------------+--------+---------------+
| user_id   | order_date  | total  | status        |
| (PK)      | (SK)        |        |               |
+-----------+-------------+--------+---------------+
| u001      | 2024-01-01  | 150.00 | shipped       |
| u001      | 2024-01-15  | 75.00  | delivered     |
| u002      | 2024-01-10  | 200.00 | processing    |
+-----------+-------------+--------+---------------+
```

---

## Capacity Modes

### On-Demand
- Pay per request
- Auto-scales instantly
- Best for: unpredictable, new workloads

### Provisioned
- Specify RCU/WCU
- Lower cost for predictable workloads
- Auto Scaling available

| Unit | Definition |
|------|------------|
| RCU (Read Capacity Unit) | 1 strongly consistent read/sec (up to 4KB) |
| | 2 eventually consistent reads/sec (up to 4KB) |
| WCU (Write Capacity Unit) | 1 write/sec (up to 1KB) |

### Capacity Calculations

```
Reads:
- Item size: 6KB
- Strongly consistent: ceil(6/4) = 2 RCU per read
- Eventually consistent: ceil(6/4) / 2 = 1 RCU per read

Writes:
- Item size: 2.5KB
- WCU needed: ceil(2.5/1) = 3 WCU per write
```

---

## Secondary Indexes

### Global Secondary Index (GSI)
- Different partition key and/or sort key
- Separate provisioned capacity
- Eventually consistent only
- Can add anytime
- Max 20 per table

```
Base Table: PK=user_id, SK=order_date
GSI: PK=status, SK=order_date
-> Query all orders by status
```

### Local Secondary Index (LSI)
- Same partition key, different sort key
- Shares capacity with table
- Strongly or eventually consistent
- Must create at table creation
- Max 5 per table

```
Base Table: PK=user_id, SK=order_date
LSI: PK=user_id, SK=status
-> Query user's orders by status
```

### Index Comparison

| Feature | GSI | LSI |
|---------|-----|-----|
| Partition Key | Different | Same |
| Sort Key | Different | Different |
| Capacity | Separate | Shared |
| Consistency | Eventually | Both |
| Creation | Anytime | Table creation |
| Limit | 20 | 5 |

---

## Query vs Scan

### Query
- Requires partition key
- Efficient, reads specific partition
- Can filter on sort key
- Returns items in sorted order

```python
response = table.query(
    KeyConditionExpression=Key('user_id').eq('u001') &
                          Key('order_date').begins_with('2024-01')
)
```

### Scan
- Reads entire table
- Expensive, consumes more RCU
- Use sparingly
- Can use parallel scan

```python
response = table.scan(
    FilterExpression=Attr('status').eq('shipped')
)
```

---

## Read Consistency

| Type | Description | Use Case |
|------|-------------|----------|
| Eventually Consistent | May not reflect latest write | Default, higher throughput |
| Strongly Consistent | Reflects all writes | When accuracy critical |

```python
# Strongly consistent read
response = table.get_item(
    Key={'user_id': 'u001'},
    ConsistentRead=True
)
```

---

## DynamoDB Streams

Capture item-level changes in real-time.

```
+---------+    +---------+    +---------+
| DynamoDB|--->| Stream  |--->| Lambda  |
|  Table  |    |         |    |         |
+---------+    +---------+    +---------+
                    |
                    v
              +---------+
              | Kinesis |
              |  Data   |
              | Firehose|
              +---------+
```

### Stream View Types
| Type | Contains |
|------|----------|
| KEYS_ONLY | Only key attributes |
| NEW_IMAGE | Entire item after modification |
| OLD_IMAGE | Entire item before modification |
| NEW_AND_OLD_IMAGES | Both before and after |

### Use Cases
- Trigger Lambda on changes
- Replication to other regions
- Analytics pipeline
- Audit logging

---

## Global Tables

Multi-region, multi-active replication.

```
+--------------+        +--------------+
|  us-east-1   |<------>|  eu-west-1   |
|   (active)   |  Sync  |   (active)   |
+--------------+        +--------------+
        ^                      ^
        |                      |
    Writes/Reads           Writes/Reads
```

### Features
- Automatic replication
- Conflict resolution (last writer wins)
- All regions are active
- Sub-second replication
- Requires Streams enabled

---

## DAX (DynamoDB Accelerator)

In-memory cache for DynamoDB.

```
+---------+    +---------+    +---------+
|  App    |--->|   DAX   |--->|DynamoDB |
|         |    | Cluster |    |         |
+---------+    +---------+    +---------+
                 (cache)
```

### Features
| Feature | Value |
|---------|-------|
| Latency | Microseconds |
| Compatibility | API compatible |
| Caching | Item cache + Query cache |
| Write-through | Writes go to DynamoDB |

### When to Use
- Read-heavy workloads
- Repeated reads of same items
- Latency-sensitive applications

### When NOT to Use
- Write-heavy workloads
- Strongly consistent reads required
- Infrequent reads

---

## Transactions

ACID transactions across multiple items/tables.

```python
client.transact_write_items(
    TransactItems=[
        {
            'Put': {
                'TableName': 'Orders',
                'Item': {'order_id': {'S': 'o001'}, ...}
            }
        },
        {
            'Update': {
                'TableName': 'Inventory',
                'Key': {'product_id': {'S': 'p001'}},
                'UpdateExpression': 'SET quantity = quantity - :qty',
                'ExpressionAttributeValues': {':qty': {'N': '1'}}
            }
        }
    ]
)
```

### Limits
- Max 100 items per transaction
- Max 4MB total
- 2x capacity consumed

---

## TTL (Time to Live)

Automatic item deletion.

```python
# Item with TTL
{
    'user_id': 'u001',
    'session_id': 'sess123',
    'expiration_time': 1705363200  # Unix timestamp
}
```

- No additional cost
- Deleted within 48 hours of expiration
- Deletions appear in Streams
- Useful for: sessions, logs, temporary data

---

## Backup & Recovery

### On-Demand Backup
- Full backup anytime
- No performance impact
- Retained until deleted

### Point-in-Time Recovery (PITR)
- Continuous backups
- Restore to any second in last 35 days
- Must be enabled

```bash
# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name MyTable \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

---

## Security

### Encryption
- At rest: AWS owned, AWS managed, or CMK
- In transit: HTTPS/TLS

### Access Control
- IAM policies
- Fine-grained access control (conditions)
- VPC Endpoints

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:Query"],
  "Resource": "arn:aws:dynamodb:*:*:table/Users",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
    }
  }
}
```

---

## CLI Quick Reference

```bash
# Create table
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=user_id,AttributeType=S \
  --key-schema AttributeName=user_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Put item
aws dynamodb put-item \
  --table-name Users \
  --item '{"user_id": {"S": "u001"}, "name": {"S": "Alice"}}'

# Get item
aws dynamodb get-item \
  --table-name Users \
  --key '{"user_id": {"S": "u001"}}'

# Query
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "user_id = :uid" \
  --expression-attribute-values '{":uid": {"S": "u001"}}'

# Scan
aws dynamodb scan --table-name Users

# Delete table
aws dynamodb delete-table --table-name Users
```

---

## Data Modeling Patterns

### Single Table Design
Store multiple entity types in one table.

```
PK          | SK              | Data
------------|-----------------|--------
USER#u001   | PROFILE         | {name, email}
USER#u001   | ORDER#o001      | {total, status}
USER#u001   | ORDER#o002      | {total, status}
PRODUCT#p01 | DETAILS         | {name, price}
```

### Adjacency List
For hierarchical/graph data.

### Sparse Indexes
GSI on attribute that not all items have.

---

## Pricing

| Component | On-Demand | Provisioned |
|-----------|-----------|-------------|
| Reads | $0.25 per 1M RRU | $0.00013 per RCU/hour |
| Writes | $1.25 per 1M WRU | $0.00065 per WCU/hour |
| Storage | $0.25 per GB/month | $0.25 per GB/month |

---

## Exam Tips

1. **Partition Key** - determines data distribution
2. **GSI** - eventually consistent only, add anytime
3. **LSI** - strongly/eventually consistent, must create at table creation
4. **Streams** - enable for triggers, replication, Global Tables
5. **DAX** - microsecond reads, API compatible
6. **Transactions** - 2x capacity consumed
7. **TTL** - free automatic deletion
8. **On-Demand** - unpredictable workloads
9. **Provisioned + Auto Scaling** - predictable workloads, cost savings
10. **Query > Scan** - always prefer Query
11. **Global Tables** - multi-region active-active
12. **Item size limit** - 400KB max
