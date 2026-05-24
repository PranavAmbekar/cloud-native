# Amazon RDS (Relational Database Service)

> Managed relational database service for MySQL, PostgreSQL, MariaDB, Oracle, SQL Server.

---

## Supported Engines

| Engine | Versions | Use Case |
|--------|----------|----------|
| MySQL | 5.7, 8.0 | General purpose, web apps |
| PostgreSQL | 12-16 | Complex queries, GIS, JSON |
| MariaDB | 10.x | MySQL alternative |
| Oracle | 12c-19c | Enterprise, legacy apps |
| SQL Server | 2016-2022 | Windows/.NET apps |
| Aurora | MySQL/PostgreSQL compatible | High performance |

---

## Key Concepts

| Term | Definition |
|------|------------|
| DB Instance | Database environment in the cloud |
| DB Instance Class | CPU, memory configuration (db.t3, db.r5, etc.) |
| Storage | EBS-based (gp2, gp3, io1, magnetic) |
| Multi-AZ | Synchronous standby in another AZ |
| Read Replica | Asynchronous read-only copy |
| Parameter Group | Engine configuration settings |
| Option Group | Additional engine features |
| Subnet Group | Subnets where RDS can be deployed |

---

## Instance Classes

| Family | Type | Use Case |
|--------|------|----------|
| db.t3/t4g | Burstable | Dev/test, small workloads |
| db.m5/m6g | General | Production, balanced workloads |
| db.r5/r6g | Memory | Memory-intensive workloads |
| db.x2g | Memory | Extreme memory (up to 4TB) |

### Graviton (ARM)
- `db.t4g`, `db.m6g`, `db.r6g`
- ~20% better price/performance

---

## Storage Types

| Type | IOPS | Use Case |
|------|------|----------|
| gp2 | 3 IOPS/GB (burst to 3,000) | General purpose |
| gp3 | 3,000 baseline, up to 16,000 | Cost-effective, predictable |
| io1/io2 | Up to 64,000 | I/O intensive, critical |
| Magnetic | Variable | Legacy, infrequent access |

### Storage Autoscaling
- Automatically increases storage
- Set maximum storage threshold
- Scales when free storage < 10%

---

## Multi-AZ Deployment

```
+-----------------------------------------------------+
|                      Region                          |
|  +------------------+    +------------------+       |
|  |      AZ-a        |    |      AZ-b        |       |
|  |  +------------+  |    |  +------------+  |       |
|  |  |  Primary   |----------|  Standby   |  |       |
|  |  |    RDS     |  Sync |  |    RDS     |  |       |
|  |  +------------+  Repl |  +------------+  |       |
|  |       ^          |    |       |          |       |
|  +-------|----------+    +-------|----------+       |
|          |                       |                   |
|      DNS Endpoint           (Auto failover)          |
|          |                       |                   |
|      Application <---------------+                   |
|                   (on failure)                       |
+-----------------------------------------------------+
```

### Multi-AZ Features
- Automatic failover (60-120 seconds)
- Synchronous replication
- Same endpoint (DNS failover)
- Automatic backups from standby
- Zero data loss
- **NOT for read scaling** (use Read Replicas)

---

## Read Replicas

```
+-----------------------------------------------------+
|  +----------+     Async      +----------+          |
|  |  Primary |--------------->| Replica  | (same AZ)|
|  |   (R/W)  |       |        |  (Read)  |          |
|  +----------+       |        +----------+          |
|                     |                               |
|                     |        +----------+          |
|                     +------->| Replica  | (cross AZ)
|                     |        |  (Read)  |          |
|                     |        +----------+          |
|                     |                               |
|                     |        +----------+          |
|                     +------->| Replica  | (cross region)
|                              |  (Read)  |          |
|                              +----------+          |
+-----------------------------------------------------+
```

### Read Replica Features
| Feature | Value |
|---------|-------|
| Max replicas | 5 per primary |
| Cross-AZ | Yes (free within region) |
| Cross-region | Yes (data transfer charges) |
| Replication | Asynchronous |
| Promotion | Can be promoted to standalone |
| Read endpoint | Each replica has unique endpoint |

### Replication Lag
- Asynchronous = eventual consistency
- Monitor with `ReplicaLag` metric
- Consider for read-after-write scenarios

---

## Backup & Recovery

### Automated Backups
- Daily full backup during maintenance window
- Transaction logs every 5 minutes
- Retention: 0-35 days
- Point-in-time recovery (PITR)
- Stored in S3 (managed by AWS)

### Manual Snapshots
- User-initiated
- Retained until deleted
- Can copy cross-region
- Can share with other accounts

### Restore
- Restores to NEW instance (not in-place)
- PITR: restore to any second within retention
- Snapshot: restore to snapshot time

```bash
# Point-in-time restore
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mydb \
  --target-db-instance-identifier mydb-restored \
  --restore-time 2024-01-15T10:00:00Z
```

---

## Security

### Network
- Deploy in VPC (private subnets recommended)
- Security Groups for access control
- No public access by default

### Encryption at Rest
- AES-256 encryption
- AWS KMS (managed or CMK)
- Must enable at creation (cannot enable later)
- Encrypts: storage, backups, snapshots, replicas

### Encryption in Transit
- SSL/TLS connections
- Download RDS CA certificate
- Force SSL via parameter group

```sql
-- MySQL: Force SSL
GRANT USAGE ON *.* TO 'user'@'%' REQUIRE SSL;
```

### IAM Authentication
- Token-based authentication
- No password management
- MySQL and PostgreSQL only

```bash
# Generate auth token
aws rds generate-db-auth-token \
  --hostname mydb.xxxxx.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --username admin
```

---

## Parameter Groups

Engine configuration settings.

| Type | Description |
|------|-------------|
| Static | Requires reboot |
| Dynamic | Applied immediately |

```bash
# Create parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name my-params \
  --db-parameter-group-family mysql8.0 \
  --description "Custom MySQL params"

# Modify parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-params \
  --parameters "ParameterName=max_connections,ParameterValue=500,ApplyMethod=immediate"
```

---

## Maintenance & Updates

### Maintenance Window
- 30-minute weekly window
- OS patching, minor engine updates
- Can defer non-required updates

### Engine Upgrades
- Minor: automatic or manual
- Major: manual only, may require downtime
- Test on replica first

---

## Monitoring

### CloudWatch Metrics
| Metric | Description |
|--------|-------------|
| CPUUtilization | CPU percentage |
| FreeableMemory | Available RAM |
| FreeStorageSpace | Available storage |
| ReadIOPS / WriteIOPS | I/O operations |
| DatabaseConnections | Active connections |
| ReplicaLag | Replication delay (seconds) |

### Enhanced Monitoring
- OS-level metrics (per second)
- Process list
- Additional cost

### Performance Insights
- Database load analysis
- Wait event analysis
- Top SQL queries
- 7 days free retention

---

## CLI Quick Reference

```bash
# Create instance
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password mypassword \
  --allocated-storage 20

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-replica \
  --source-db-instance-identifier mydb

# Enable Multi-AZ
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --multi-az \
  --apply-immediately

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-snapshot

# Describe instances
aws rds describe-db-instances

# Delete instance
aws rds delete-db-instance \
  --db-instance-identifier mydb \
  --skip-final-snapshot
```

---

## Pricing Components

| Component | Charged For |
|-----------|-------------|
| Instance | Per hour (running) |
| Storage | Per GB-month |
| I/O (io1/io2) | Per IOPS provisioned |
| Backup | Beyond allocated storage |
| Data Transfer | Cross-region, internet |
| Snapshots | Per GB-month |

---

## RDS vs Aurora

| Feature | RDS | Aurora |
|---------|-----|--------|
| Storage limit | 64 TB | 128 TB |
| Replicas | 5 | 15 |
| Replication | Async | Sync (6 copies) |
| Failover | 60-120s | <30s |
| Storage scaling | Manual/Auto | Automatic |
| Cost | Lower | Higher (but more performant) |

---

## Best Practices

1. **Use Multi-AZ** for production
2. **Enable automated backups** with appropriate retention
3. **Use Read Replicas** for read-heavy workloads
4. **Enable encryption** at creation
5. **Use Parameter Groups** for tuning
6. **Monitor with Performance Insights**
7. **Place in private subnets**
8. **Use IAM authentication** where possible
9. **Enable Enhanced Monitoring** for detailed metrics
10. **Test failover** periodically

---

## Exam Tips

1. **Multi-AZ = HA** (failover), **Read Replicas = scalability** (read performance)
2. **Multi-AZ is synchronous**, Read Replicas are asynchronous
3. **Encryption must be enabled at creation** (cannot encrypt existing)
4. **Read Replica can be promoted** to standalone (breaks replication)
5. **Cross-region Read Replica** - data transfer charges apply
6. **Automated backups** - max 35 days retention
7. **PITR** - restore to any second within retention period
8. **Restore creates NEW instance** - not in-place
9. **IAM auth** - MySQL and PostgreSQL only
10. **Storage autoscaling** - automatically increases, never decreases
