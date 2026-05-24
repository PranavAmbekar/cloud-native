# Azure SQL Database

> Fully managed relational database with built-in intelligence, high availability, and elastic scaling.

## Overview

Azure SQL Database is a fully managed PaaS database engine that handles most database management functions. It's based on the latest stable version of Microsoft SQL Server and provides built-in high availability, automated backups, and intelligent performance optimization.

## Key Concepts

| Term | Definition |
|------|------------|
| Logical Server | Management container for databases |
| Database | Individual SQL database |
| Elastic Pool | Shared resources across multiple databases |
| DTU | Database Transaction Unit (bundled compute metric) |
| vCore | Virtual core (granular compute configuration) |
| Service Tier | Performance/feature level |
| Read Replica | Read-only copy for scaling reads |

## Deployment Options

| Option | Description | Use Case |
|--------|-------------|----------|
| **Single Database** | Isolated database with dedicated resources | Most workloads |
| **Elastic Pool** | Multiple databases sharing resources | SaaS, multi-tenant |
| **Managed Instance** | Full SQL Server compatibility | Lift-and-shift |
| **Hyperscale** | Massive scale (100 TB+) | Large databases |

## Purchasing Models

### DTU Model

```
DTU = CPU + Memory + I/O (bundled)

Basic:    5 DTUs, 2 GB max
Standard: 10-3000 DTUs, 1 TB max
Premium:  125-4000 DTUs, 4 TB max
```

### vCore Model

```
General Purpose:  2-128 vCores
Business Critical: 2-128 vCores
Hyperscale:       2-128 vCores

+ Choose: Memory, Storage, IOPS independently
```

## Service Tiers

| Tier | Use Case | Availability | Features |
|------|----------|--------------|----------|
| **Basic** | Light workloads | 99.99% | 5 DTUs, 2 GB |
| **Standard** | General workloads | 99.99% | Scalable DTUs |
| **Premium** | High I/O, low latency | 99.99% | In-memory OLTP |
| **General Purpose** | Budget-friendly | 99.99% | Remote storage |
| **Business Critical** | High performance | 99.99% | Local SSD, read replica |
| **Hyperscale** | Very large databases | 99.99% | 100 TB, fast scaling |

### vCore Tier Comparison

| Feature | General Purpose | Business Critical | Hyperscale |
|---------|-----------------|-------------------|------------|
| Storage | Remote | Local SSD | Distributed |
| Max size | 16 TB | 4 TB | 100 TB |
| Read replicas | No | 1 free | 0-4 |
| IOPS | Standard | High | Very high |
| Zone redundancy | Optional | Optional | Optional |
| Serverless | Yes | No | Yes |

## Compute Tiers (vCore)

| Tier | Description |
|------|-------------|
| **Provisioned** | Fixed compute, always on |
| **Serverless** | Auto-scale, auto-pause, pay per use |

### Serverless

```
┌─────────────────────────────────────────────────┐
│              Serverless Compute                  │
│                                                  │
│  Min vCores: 0.5 ─────────────▶ Max vCores: 16 │
│                                                  │
│  Auto-pause after: 1 hour (configurable)        │
│  Resume time: ~1 minute                         │
│                                                  │
│  Billing: Per second of compute used            │
└─────────────────────────────────────────────────┘
```

## High Availability

### General Purpose

```
┌─────────────────────────────────────────────────────┐
│                 Compute (Stateless)                  │
│  ┌─────┐  ┌─────┐  ┌─────┐                         │
│  │ VM1 │  │ VM2 │  │ VM3 │  (Failover replicas)   │
│  └──┬──┘  └──┬──┘  └──┬──┘                         │
└─────┼───────┼───────┼───────────────────────────────┘
      │       │       │
      ▼       ▼       ▼
┌─────────────────────────────────────────────────────┐
│           Azure Premium Storage (LRS/ZRS)           │
│              (Data + Logs + Backups)                │
└─────────────────────────────────────────────────────┘
```

### Business Critical

```
┌─────────────────────────────────────────────────────┐
│  Always On Availability Group (synchronous)         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐          │
│  │Primary  │◄─►│Secondary│◄─►│Secondary│          │
│  │(R/W)    │   │(standby)│   │(standby)│          │
│  │+ Local  │   │+ Local  │   │+ Local  │          │
│  │  SSD    │   │  SSD    │   │  SSD    │          │
│  └─────────┘   └─────────┘   └─────────┘          │
│       │                                             │
│       └─────────────────────────────────────────┐  │
│                                                  ▼  │
│                              ┌──────────────────┐  │
│                              │  Read Replica    │  │
│                              │  (free, R/O)     │  │
│                              └──────────────────┘  │
└─────────────────────────────────────────────────────┘
```

## Elastic Pools

Share resources across multiple databases.

```
┌─────────────────────────────────────────────────────┐
│              Elastic Pool (500 eDTUs)               │
│                                                      │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐ │
│  │ DB1  │  │ DB2  │  │ DB3  │  │ DB4  │  │ DB5  │ │
│  │ 50   │  │ 200  │  │ 100  │  │ 50   │  │ 100  │ │
│  │ eDTU │  │ eDTU │  │ eDTU │  │ eDTU │  │ eDTU │ │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘ │
│                                                      │
│  Min eDTU per DB: 0    Max eDTU per DB: 250        │
└─────────────────────────────────────────────────────┘
```

Use cases:
- SaaS applications with many tenant databases
- Unpredictable usage patterns
- Cost optimization for multiple databases

## Backup and Restore

### Automated Backups

| Backup Type | Frequency | Retention |
|-------------|-----------|-----------|
| Full | Weekly | 7-35 days (configurable) |
| Differential | Every 12-24 hours | Same as above |
| Transaction log | Every 5-10 minutes | Same as above |

### Point-in-Time Restore (PITR)

```
Timeline:
─────────────────────────────────────────────────▶
   │      │      │              │              │
  Full   Diff   Log            Log           Log
   │                            ▲
   └────────────────────────────┘
        Restore to any point
```

### Long-term Retention (LTR)

Store backups for years (compliance).

```
Weekly:  Keep 4 backups
Monthly: Keep 12 backups
Yearly:  Keep 10 backups
```

## Geo-Replication

### Active Geo-Replication

```
Primary (East US)              Secondary (West US)
┌─────────────┐    async      ┌─────────────┐
│   Database  │──────────────▶│  Read-only  │
│    (R/W)    │  replication  │   replica   │
└─────────────┘               └─────────────┘
                                     │
                              Can be promoted
                              to primary
```

### Auto-Failover Groups

```
┌────────────────────────────────────────────────────┐
│              Failover Group                         │
│                                                     │
│  Primary Server          Secondary Server          │
│  ┌──────────────┐       ┌──────────────┐          │
│  │ Database 1   │──────▶│ Database 1   │          │
│  │ Database 2   │──────▶│ Database 2   │          │
│  │ Database 3   │──────▶│ Database 3   │          │
│  └──────────────┘       └──────────────┘          │
│                                                     │
│  Listener: <group-name>.database.windows.net       │
│  R/W: <group-name>.database.windows.net            │
│  R/O: <group-name>.secondary.database.windows.net  │
└────────────────────────────────────────────────────┘
```

## Security

### Authentication

| Method | Description |
|--------|-------------|
| **SQL Authentication** | Username/password |
| **Azure AD** | Azure AD users/groups/managed identity |
| **Azure AD MFA** | Multi-factor authentication |

### Network Security

| Feature | Description |
|---------|-------------|
| **Firewall rules** | IP-based access control |
| **VNet service endpoints** | Access from VNet subnets |
| **Private endpoints** | Private IP in your VNet |
| **Minimum TLS version** | Enforce TLS 1.2+ |

### Data Security

| Feature | Description |
|---------|-------------|
| **TDE** | Transparent Data Encryption (default) |
| **Always Encrypted** | Client-side column encryption |
| **Dynamic Data Masking** | Mask sensitive data |
| **Row-Level Security** | Filter rows by user |
| **Auditing** | Track database events |

```sql
-- Dynamic Data Masking example
ALTER TABLE Users
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');

ALTER TABLE Users
ALTER COLUMN SSN ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)');
```

## Performance Tuning

### Automatic Tuning

| Feature | Description |
|---------|-------------|
| **Force last good plan** | Revert to better query plan |
| **Create index** | Auto-create missing indexes |
| **Drop index** | Remove unused indexes |

### Query Performance Insight

```
Top Resource Consuming Queries
├── Query 1: 45% DTU
├── Query 2: 23% DTU
└── Query 3: 12% DTU

Recommendations:
- Add index on Column X
- Update statistics on Table Y
```

## CLI Quick Reference

```bash
# Create logical server
az sql server create \
  --name mysqlserver \
  --resource-group myRG \
  --location eastus \
  --admin-user adminuser \
  --admin-password 'SecureP@ssw0rd!'

# Create database
az sql db create \
  --name mydb \
  --resource-group myRG \
  --server mysqlserver \
  --service-objective S0

# Create serverless database
az sql db create \
  --name mydb \
  --resource-group myRG \
  --server mysqlserver \
  --edition GeneralPurpose \
  --compute-model Serverless \
  --auto-pause-delay 60 \
  --min-capacity 0.5 \
  --max-capacity 4

# Create elastic pool
az sql elastic-pool create \
  --name mypool \
  --resource-group myRG \
  --server mysqlserver \
  --edition Standard \
  --dtu 100

# Add firewall rule
az sql server firewall-rule create \
  --resource-group myRG \
  --server mysqlserver \
  --name AllowMyIP \
  --start-ip-address 1.2.3.4 \
  --end-ip-address 1.2.3.4

# Create geo-replica
az sql db replica create \
  --name mydb \
  --resource-group myRG \
  --server mysqlserver \
  --partner-server mysqlserver-secondary \
  --partner-resource-group myRG

# Restore to point in time
az sql db restore \
  --dest-name mydb-restored \
  --resource-group myRG \
  --server mysqlserver \
  --name mydb \
  --time "2024-01-15T10:00:00Z"
```

## Connection String

```
Server=tcp:<server>.database.windows.net,1433;
Initial Catalog=<database>;
Persist Security Info=False;
User ID=<user>;
Password=<password>;
MultipleActiveResultSets=False;
Encrypt=True;
TrustServerCertificate=False;
Connection Timeout=30;
```

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **DTU vs vCore**: DTU = bundled; vCore = granular control
2. **Business Critical**: Local SSD, free read replica, zone redundancy available
3. **Serverless**: Auto-pause saves money, ~1 min resume time
4. **Elastic pools**: Share DTUs/vCores across databases
5. **Geo-replication**: Up to 4 readable secondaries
6. **Failover groups**: Automatic failover with listener endpoints
7. **PITR**: Restore to any point within retention period (7-35 days)
8. **LTR**: Weekly/monthly/yearly backups for compliance
9. **TDE**: Enabled by default, encrypts data at rest
10. **Private endpoints**: Most secure network access

## Gotchas

- Logical server name must be globally unique
- Deleting server deletes all databases
- Serverless has cold start latency (~1 minute)
- Elastic pool databases share resources (noisy neighbor possible)
- Geo-replication is async (potential data loss on failover)
- Max database size varies by tier (4 TB Business Critical, 100 TB Hyperscale)
- DTU Basic tier doesn't support geo-replication
- Always Encrypted requires driver support
- TDE uses service-managed keys by default (can use CMK)
- Firewall rules take a few minutes to propagate

## Limits

| Resource | Limit |
|----------|-------|
| Databases per server | 5000 |
| Servers per subscription | 200 per region |
| Elastic pools per server | 500 |
| Max database size (General Purpose) | 16 TB |
| Max database size (Business Critical) | 4 TB |
| Max database size (Hyperscale) | 100 TB |
| DTU per database | 4000 (Premium P15) |
| vCores per database | 128 |
| Geo-replicas per database | 4 |
| Backup retention | 7-35 days (PITR), 10 years (LTR) |
