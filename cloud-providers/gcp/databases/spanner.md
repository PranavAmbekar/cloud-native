# Google Cloud Spanner

> Fully managed, scalable, relational database service with unlimited scale and strong consistency.

## Overview

Cloud Spanner is a fully managed relational database that combines the benefits of relational structure with non-relational horizontal scale. It provides ACID transactions, SQL semantics, and automatic synchronous replication for high availability.

## Key Concepts

| Term | Definition |
|------|------------|
| Instance | Container for databases with compute capacity |
| Database | Contains tables, indexes, and data |
| Node | Unit of compute and storage capacity |
| Processing Unit | Smaller compute unit (1 node = 1000 PUs) |
| Split | Horizontal partition of data |
| Interleaved Table | Child table co-located with parent |

## Architecture

```
+---------------------------------------------------------------+
|                    Cloud Spanner Instance                     |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                 Regional / Multi-Regional               |  |
|  |                                                         |  |
|  |   +---------+   +---------+   +---------+               |  |
|  |   |  Zone A |   |  Zone B |   |  Zone C |               |  |
|  |   | +-----+ |   | +-----+ |   | +-----+ |               |  |
|  |   | |Split| |   | |Split| |   | |Split| |               |  |
|  |   | |  1  | |   | |  1  | |   | |  1  | |               |  |
|  |   | +-----+ |   | +-----+ |   | +-----+ |               |  |
|  |   | +-----+ |   | +-----+ |   | +-----+ |               |  |
|  |   | |Split| |   | |Split| |   | |Split| |               |  |
|  |   | |  2  | |   | |  2  | |   | |  2  | |               |  |
|  |   | +-----+ |   | +-----+ |   | +-----+ |               |  |
|  |   +---------+   +---------+   +---------+               |  |
|  |              Synchronous Replication                    |  |
|  +---------------------------------------------------------+  |
|                                                               |
|  Capacity: 3 Nodes (3000 Processing Units)                   |
|  SLA: 99.999% (Multi-Regional)                               |
+---------------------------------------------------------------+
```

## Instance Configurations

| Type | Description | SLA |
|------|-------------|-----|
| **Regional** | Single region, 3 zones | 99.99% |
| **Multi-Regional** | Multiple regions | 99.999% |

### Multi-Regional Configurations

| Configuration | Regions |
|---------------|---------|
| nam3 | us-east1, us-east4, us-central1 |
| nam6 | us-east4, us-west1, us-central1 |
| nam-eur-asia1 | US, Europe, Asia |
| eur5 | europe-west1, europe-west4 |

## Capacity Planning

### Processing Units (Recommended)

```
Processing Units (PU):
+-- Minimum: 100 PU
+-- 1 Node = 1000 PU
+-- Scale in 100 PU increments (< 1000)
+-- Scale in 1000 PU increments (>= 1000)

Capacity per 1000 PU (1 Node):
+-- ~10,000 QPS reads
+-- ~2,000 QPS writes
+-- ~2 TB storage
```

```bash
# Create instance with processing units
gcloud spanner instances create my-instance \
  --config=regional-us-central1 \
  --processing-units=100 \
  --description="Development instance"

# Scale up
gcloud spanner instances update my-instance \
  --processing-units=500
```

## Schema Design

### Table Definition

```sql
CREATE TABLE Users (
  UserId     INT64 NOT NULL,
  FirstName  STRING(100),
  LastName   STRING(100),
  Email      STRING(200) NOT NULL,
  CreatedAt  TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
) PRIMARY KEY (UserId);

CREATE TABLE Orders (
  UserId     INT64 NOT NULL,
  OrderId    INT64 NOT NULL,
  Amount     FLOAT64,
  Status     STRING(50),
  CreatedAt  TIMESTAMP NOT NULL,
) PRIMARY KEY (UserId, OrderId),
  INTERLEAVE IN PARENT Users ON DELETE CASCADE;
```

### Interleaved Tables

```
Parent: Users
+-- UserId: 1
|   +-- User data
|   +-- Orders (interleaved)
|       +-- OrderId: 1001
|       +-- OrderId: 1002
|       +-- OrderId: 1003
+-- UserId: 2
|   +-- User data
|   +-- Orders (interleaved)
|       +-- OrderId: 2001

Benefits:
- Co-located on same splits
- Efficient joins
- Better locality
```

### Indexes

```sql
-- Secondary index
CREATE INDEX UsersByEmail ON Users(Email);

-- Storing index (includes columns)
CREATE INDEX UsersByLastName ON Users(LastName) STORING (FirstName, Email);

-- Null-filtered index
CREATE NULL_FILTERED INDEX ActiveUsers ON Users(Status)
WHERE Status IS NOT NULL;

-- Interleaved index
CREATE INDEX OrdersByDate ON Orders(CreatedAt DESC),
INTERLEAVE IN Users;
```

## Data Types

| Type | Description |
|------|-------------|
| BOOL | Boolean |
| INT64 | 64-bit integer |
| FLOAT64 | 64-bit floating point |
| NUMERIC | Exact numeric |
| STRING(N) | Variable-length string |
| BYTES(N) | Variable-length bytes |
| DATE | Date |
| TIMESTAMP | Timestamp with timezone |
| JSON | JSON document |
| ARRAY | Ordered list |

## Queries and DML

### Read Operations

```python
from google.cloud import spanner

client = spanner.Client()
instance = client.instance("my-instance")
database = instance.database("my-database")

# Read with SQL
with database.snapshot() as snapshot:
    results = snapshot.execute_sql(
        "SELECT UserId, FirstName, LastName FROM Users WHERE UserId = @user_id",
        params={"user_id": 123},
        param_types={"user_id": spanner.param_types.INT64}
    )
    for row in results:
        print(row)

# Read specific rows (faster than SQL)
with database.snapshot() as snapshot:
    results = snapshot.read(
        table="Users",
        columns=["UserId", "FirstName", "LastName"],
        keyset=spanner.KeySet(keys=[[123], [456]])
    )
```

### Write Operations

```python
# Single DML
def update_user(transaction):
    transaction.execute_update(
        "UPDATE Users SET LastName = @name WHERE UserId = @id",
        params={"name": "Smith", "id": 123},
        param_types={
            "name": spanner.param_types.STRING,
            "id": spanner.param_types.INT64
        }
    )

database.run_in_transaction(update_user)

# Batch DML
def batch_updates(transaction):
    statements = [
        ("UPDATE Users SET Status = 'active' WHERE UserId = 1", {}, {}),
        ("UPDATE Users SET Status = 'active' WHERE UserId = 2", {}, {}),
    ]
    row_counts = transaction.batch_update(statements)

database.run_in_transaction(batch_updates)

# Mutations (faster for bulk)
with database.batch() as batch:
    batch.insert(
        "Users",
        columns=["UserId", "FirstName", "LastName", "Email", "CreatedAt"],
        values=[
            [1, "Alice", "Smith", "alice@example.com", spanner.COMMIT_TIMESTAMP],
            [2, "Bob", "Jones", "bob@example.com", spanner.COMMIT_TIMESTAMP],
        ]
    )
```

## Transactions

### Read-Write Transaction

```python
def transfer_funds(transaction):
    # Read
    results = transaction.execute_sql(
        "SELECT Balance FROM Accounts WHERE AccountId = @id",
        params={"id": 1},
        param_types={"id": spanner.param_types.INT64}
    )
    balance = list(results)[0][0]

    # Write
    transaction.execute_update(
        "UPDATE Accounts SET Balance = @balance WHERE AccountId = @id",
        params={"balance": balance - 100, "id": 1},
        param_types={
            "balance": spanner.param_types.FLOAT64,
            "id": spanner.param_types.INT64
        }
    )

database.run_in_transaction(transfer_funds)
```

### Read-Only Transaction

```python
# Strong read (most recent data)
with database.snapshot() as snapshot:
    results = snapshot.execute_sql("SELECT * FROM Users")

# Stale read (bounded staleness)
import datetime
staleness = datetime.timedelta(seconds=10)
with database.snapshot(exact_staleness=staleness) as snapshot:
    results = snapshot.execute_sql("SELECT * FROM Users")

# Read at specific timestamp
with database.snapshot(read_timestamp=timestamp) as snapshot:
    results = snapshot.execute_sql("SELECT * FROM Users")
```

## Best Practices

### Primary Key Design

```sql
-- Good: UUID or hash-based (distributed)
CREATE TABLE Events (
  EventId STRING(36) NOT NULL,  -- UUID
  ...
) PRIMARY KEY (EventId);

-- Good: Reverse timestamp (recent data distributed)
CREATE TABLE Logs (
  ReverseTimestamp INT64 NOT NULL,  -- MAX_INT - timestamp
  LogId INT64 NOT NULL,
  ...
) PRIMARY KEY (ReverseTimestamp, LogId);

-- Bad: Sequential (creates hotspots)
CREATE TABLE BadTable (
  Id INT64 NOT NULL,  -- Auto-incrementing
  ...
) PRIMARY KEY (Id);
```

### Avoid Hotspots

| Problem | Solution |
|---------|----------|
| Sequential keys | Use UUIDs or hash prefixes |
| Timestamp-first keys | Reverse timestamp or add hash |
| Popular rows | Shard data or use caching |
| Recent data | Use reverse timestamp |

## CLI Quick Reference

```bash
# Create instance
gcloud spanner instances create my-instance \
  --config=regional-us-central1 \
  --processing-units=100 \
  --description="My instance"

# Create database
gcloud spanner databases create my-database \
  --instance=my-instance

# Execute DDL
gcloud spanner databases ddl update my-database \
  --instance=my-instance \
  --ddl='CREATE TABLE Users (UserId INT64 NOT NULL) PRIMARY KEY (UserId)'

# Execute query
gcloud spanner databases execute-sql my-database \
  --instance=my-instance \
  --sql='SELECT * FROM Users'

# List instances
gcloud spanner instances list

# Describe database
gcloud spanner databases describe my-database --instance=my-instance

# Create backup
gcloud spanner backups create my-backup \
  --instance=my-instance \
  --database=my-database \
  --retention-period=7d \
  --expiration-date=2024-12-31

# Delete instance
gcloud spanner instances delete my-instance
```

## Pricing

| Component | Cost |
|-----------|------|
| Processing Unit (regional) | ~$0.90/PU/month |
| Processing Unit (multi-regional) | ~$3.00/PU/month |
| Storage | $0.30/GB/month |
| Network | Standard GCP egress |
| Backups | $0.30/GB/month |

## Exam Tips (Professional Cloud Architect, Database Engineer)

1. **Processing Units**: 100 PU minimum, scale in 100/1000 increments
2. **Multi-regional**: 99.999% SLA, higher cost
3. **Interleaved tables**: Co-locate parent-child data
4. **Hotspots**: Avoid sequential/timestamp primary keys
5. **STORING indexes**: Include columns for index-only queries
6. **Stale reads**: Lower latency, use for analytics
7. **Mutations vs DML**: Mutations faster for bulk ops
8. **Commit timestamp**: Use `allow_commit_timestamp` option
9. **Backups**: Point-in-time recovery available
10. **TrueTime**: Enables external consistency

## Gotchas

- Cannot change instance configuration (regional <-> multi-regional)
- Primary key design critical for performance
- Sequential keys cause hotspots
- Schema changes can be slow for large tables
- No auto-incrementing IDs (use UUIDs)
- INTERLEAVE requires same primary key prefix
- Maximum 7 interleave levels
- Stale reads not allowed in read-write transactions
- Processing units affect both compute and storage capacity
- Multi-regional replication is synchronous (latency impact)

## Limits

| Resource | Limit |
|----------|-------|
| Databases per instance | 100 |
| Tables per database | 5,000 |
| Columns per table | 1,024 |
| Indexes per table | 100 |
| Interleave depth | 7 levels |
| Row size | 10 MB |
| Transaction size | 100 MB |
| Mutations per transaction | 80,000 |
| Processing units | 10,000+ (with support) |
| Storage per node | ~2 TB |
