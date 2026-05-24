# Amazon Aurora

> MySQL and PostgreSQL-compatible relational database with up to 5x performance of standard MySQL and 3x of PostgreSQL.

---

## Key Features

| Feature | Value |
|---------|-------|
| Compatibility | MySQL 5.7/8.0, PostgreSQL 12-15 |
| Storage | Up to 128 TB, auto-scaling |
| Replicas | Up to 15 read replicas |
| Availability | 99.99% (Multi-AZ) |
| Replication | Synchronous (6 copies across 3 AZs) |
| Failover | < 30 seconds |

---

## Architecture

```
+-------------------------------------------------------------+
|                         Region                               |
|  +-----------------------------------------------------+    |
|  |              Cluster Volume (Shared Storage)         |    |
|  |    +-----+  +-----+  +-----+  +-----+  +-----+     |    |
|  |    |Copy1|  |Copy2|  |Copy3|  |Copy4|  |Copy5| ... |    |
|  |    +-----+  +-----+  +-----+  +-----+  +-----+     |    |
|  |       AZ-a      AZ-b      AZ-c    (6 copies total)  |    |
|  +-----------------------------------------------------+    |
|         ^              ^              ^                      |
|         |              |              |                      |
|  +------+------+ +-----+-----+ +-----+-----+               |
|  |   Primary   | |  Replica  | |  Replica  |               |
|  |   (R/W)     | |  (Read)   | |  (Read)   |               |
|  +-------------+ +-----------+ +-----------+               |
|         |                                                    |
|    Writer Endpoint          Reader Endpoint                  |
|    (always primary)         (load balanced)                  |
+-------------------------------------------------------------+
```

---

## Storage Architecture

### Self-Healing Storage
- 6 copies across 3 AZs
- Survives loss of 2 copies (writes)
- Survives loss of 3 copies (reads)
- Automatic repair using healthy copies
- Continuous backup to S3

### Storage Auto-Scaling
- Starts at 10 GB
- Scales in 10 GB increments
- Up to 128 TB
- No performance impact
- Only pay for what you use

---

## Endpoints

| Endpoint | Purpose |
|----------|---------|
| Cluster (Writer) | Points to primary, for writes |
| Reader | Load balances across replicas |
| Instance | Direct connection to specific instance |
| Custom | User-defined group of instances |

```
# Writer endpoint (always primary)
mydb-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com

# Reader endpoint (load balanced)
mydb-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com
```

---

## Aurora Replicas vs MySQL Read Replicas

| Feature | Aurora Replica | MySQL Read Replica |
|---------|----------------|-------------------|
| Number | Up to 15 | Up to 5 |
| Replication | Milliseconds (shared storage) | Seconds (async) |
| Failover target | Yes (automatic) | Manual promotion |
| Impact on primary | None | Some |

---

## Failover

### Priority Tiers
- Tier 0-15 (lower = higher priority)
- Same tier = largest instance promoted

### Failover Time
- Typically < 30 seconds
- Automatic DNS update
- No data loss (synchronous)

```bash
# Manual failover
aws rds failover-db-cluster --db-cluster-identifier my-cluster
```

---

## Aurora Serverless v2

Auto-scaling compute capacity.

```
+-------------------------------------------------+
| Aurora Serverless v2                            |
|                                                 |
|    Min ACU ---------------------- Max ACU       |
|     0.5                            128          |
|      |                               |          |
|      +---- Scales in seconds --------+          |
|                                                 |
+-------------------------------------------------+
```

### ACU (Aurora Capacity Unit)
- 1 ACU = ~2 GB memory
- Range: 0.5 - 128 ACUs
- Scales in increments of 0.5 ACU

### v1 vs v2
| Feature | v1 | v2 |
|---------|----|----|
| Scaling | Pause/Resume | Continuous |
| Min capacity | 1 ACU | 0.5 ACU |
| Max capacity | 256 ACU | 128 ACU |
| Mixed config | No | Yes (with provisioned) |
| Read replicas | No | Yes |

---

## Aurora Global Database

Cross-region replication with < 1 second lag.

```
+------------------+                    +------------------+
|  Primary Region  |                    | Secondary Region |
|   (us-east-1)    |     < 1 sec lag    |   (eu-west-1)    |
|  +------------+  |                    |  +------------+  |
|  |  Primary   |--|--------------------|->|  Replica   |  |
|  |  Cluster   |  |                    |  |  Cluster   |  |
|  +------------+  |                    |  +------------+  |
+------------------+                    +------------------+
         |                                       |
    Read/Write                              Read Only
                                          (can promote)
```

### Features
- Up to 5 secondary regions
- < 1 second replication lag
- RPO: ~1 second
- RTO: < 1 minute
- Promotes secondary to primary in DR

---

## Aurora Machine Learning

Query ML models directly from SQL.

```sql
-- Sentiment analysis
SELECT aws_comprehend.detect_sentiment(review_text, 'en')
FROM reviews;

-- Custom SageMaker model
SELECT aws_sagemaker.invoke_endpoint(
    'my-endpoint',
    features
) FROM data;
```

Integrates with:
- Amazon Comprehend
- Amazon SageMaker

---

## Backup & Restore

### Automated Backups
- Continuous to S3
- 1-35 days retention
- Point-in-time recovery (PITR)
- No performance impact

### Manual Snapshots
- Retained until deleted
- Can copy cross-region
- Can share with accounts

### Backtrack (MySQL only)
```
Rewind database to point in time WITHOUT restore
-------------------------------------------------> Time
           ^                              ^
           |                              |
      Backtrack to                   Current
      this point                      state
```

- Up to 72 hours
- In-place (no new cluster)
- Pay per change record

```bash
aws rds backtrack-db-cluster \
  --db-cluster-identifier my-cluster \
  --backtrack-to 2024-01-15T10:00:00Z
```

---

## Cloning

Create copy of database in minutes.

```
Original Cluster                Clone
+-----------------+       +-----------------+
|                 |       |                 |
|   Shared Data   |<----->|   Shared Data   |
|    (copy on     |       |    (copy on     |
|     write)      |       |     write)      |
|                 |       |                 |
+-----------------+       +-----------------+
```

- Uses copy-on-write
- Initial clone is nearly instant
- Diverges as changes are made
- Great for testing/development

```bash
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier prod-cluster \
  --db-cluster-identifier test-cluster \
  --restore-type copy-on-write \
  --use-latest-restorable-time
```

---

## Security

### Encryption
- At rest: KMS (AES-256)
- In transit: SSL/TLS
- Must enable at creation

### Network
- VPC only
- Security groups
- No public IP option

### Authentication
- Password
- IAM database authentication
- Kerberos (Active Directory)

---

## Monitoring

### CloudWatch Metrics
| Metric | Description |
|--------|-------------|
| CPUUtilization | CPU usage |
| DatabaseConnections | Active connections |
| FreeableMemory | Available memory |
| VolumeReadIOPs | Read operations |
| VolumeWriteIOPs | Write operations |
| AuroraReplicaLag | Replication delay |
| ServerlessDatabaseCapacity | Current ACUs (Serverless) |

### Performance Insights
- Database load by wait events
- Top SQL queries
- 7 days free retention

### Enhanced Monitoring
- OS-level metrics
- Per-second granularity

---

## CLI Quick Reference

```bash
# Create cluster
aws rds create-db-cluster \
  --db-cluster-identifier my-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --master-username admin \
  --master-user-password mypassword

# Add instance to cluster
aws rds create-db-instance \
  --db-instance-identifier my-instance \
  --db-cluster-identifier my-cluster \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql

# Add read replica
aws rds create-db-instance \
  --db-instance-identifier my-replica \
  --db-cluster-identifier my-cluster \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql

# Create Serverless v2
aws rds create-db-cluster \
  --db-cluster-identifier serverless-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --master-username admin \
  --master-user-password mypassword

# Create global database
aws rds create-global-cluster \
  --global-cluster-identifier my-global \
  --source-db-cluster-identifier my-cluster
```

---

## Pricing

| Component | Cost Basis |
|-----------|------------|
| Instance | Per hour |
| Storage | Per GB/month |
| I/O | Per million requests |
| Backup | Beyond retention period |
| Backtrack | Per change record |
| Global DB | Data transfer |

### Serverless v2
- Per ACU-hour consumed
- Storage charged separately
- I/O charged separately

---

## Aurora vs RDS

| Feature | Aurora | RDS |
|---------|--------|-----|
| Performance | 5x MySQL, 3x PostgreSQL | Standard |
| Storage | Auto-scaling to 128TB | Manual, up to 64TB |
| Replicas | 15 | 5 |
| Failover | < 30 sec | 60-120 sec |
| Replication | Synchronous | Asynchronous |
| Cost | Higher | Lower |
| Backtrack | Yes (MySQL) | No |
| Serverless | Yes | No |
| Global | < 1 sec lag | Higher lag |

---

## Exam Tips

1. **6 copies across 3 AZs** - storage architecture
2. **Up to 15 replicas** - vs 5 for RDS
3. **< 30 second failover** - vs 60-120s for RDS
4. **Shared storage** - replicas don't copy data
5. **Backtrack** - MySQL only, up to 72 hours
6. **Global Database** - < 1 second replication lag
7. **Serverless v2** - scales continuously, mix with provisioned
8. **Clone** - copy-on-write, instant for testing
9. **Reader endpoint** - load balances read traffic
10. **Writer endpoint** - always points to primary
11. **Custom endpoints** - group specific instances
12. **No cross-region replicas** - use Global Database instead
