# Linode Managed Databases

> Fully managed database clusters with automatic backups, updates, and high availability.

## Overview

Linode Managed Databases provides fully managed database services for MySQL, PostgreSQL, and MongoDB. Linode handles provisioning, backups, maintenance, and updates while you focus on your application.

## Supported Engines

| Engine | Versions | Cluster Sizes |
|--------|----------|---------------|
| **MySQL** | 8.0 | 1, 3 nodes |
| **PostgreSQL** | 13, 14, 15, 16 | 1, 3 nodes |
| **MongoDB** | 6.0 | 1, 3 nodes (replica set) |

## Key Concepts

| Term | Definition |
|------|------------|
| Cluster | Database instance (1 or 3 nodes) |
| Node | Individual database server |
| Primary | Write-enabled node |
| Standby | Read replica (HA clusters) |
| Engine | Database software (MySQL, PG, Mongo) |
| Plan | CPU, RAM, storage allocation |

## Architecture

### Single Node

```
+---------------------------------------+
|          Database Cluster             |
|                                       |
|  +-------------------------------+    |
|  |        Primary Node           |    |
|  |   +-------+  +--------+       |    |
|  |   | MySQL |  | Data   |       |    |
|  |   | 8.0   |  | Volume |       |    |
|  |   +-------+  +--------+       |    |
|  +-------------------------------+    |
|                                       |
|  Daily Backups: Retained 7 days       |
+---------------------------------------+
```

### High Availability (3 Nodes)

```
+---------------------------------------------------------------+
|                  Database Cluster (HA)                        |
|                                                               |
|  +-------------------+                                        |
|  |   Primary Node    |  Writes                                |
|  |   +-------+       |<--------------+                        |
|  |   | MySQL |       |               |                        |
|  |   +-------+       |               |                        |
|  +---------+---------+               |                        |
|            |                         |                        |
|    Replication                       |                        |
|            |                         |                        |
|  +---------v---------+  +-----------+-----------+             |
|  |  Standby Node 1   |  |    Standby Node 2     |             |
|  |  (sync replica)   |  |    (sync replica)     |             |
|  +-------------------+  +-----------------------+             |
|                                                               |
|  Automatic Failover: ~30 seconds                              |
|  Read Distribution: Optional                                  |
+---------------------------------------------------------------+
```

## Pricing

### Shared CPU Plans

| Plan | vCPU | RAM | Storage | Price/mo |
|------|------|-----|---------|----------|
| DBaaS 1GB | 1 | 1 GB | 15 GB | $15 |
| DBaaS 2GB | 1 | 2 GB | 30 GB | $25 |
| DBaaS 4GB | 2 | 4 GB | 60 GB | $50 |
| DBaaS 8GB | 4 | 8 GB | 120 GB | $100 |
| DBaaS 16GB | 6 | 16 GB | 240 GB | $200 |

### Dedicated CPU Plans

| Plan | vCPU | RAM | Storage | Price/mo |
|------|------|-----|---------|----------|
| DBaaS Dedicated 4GB | 2 | 4 GB | 60 GB | $65 |
| DBaaS Dedicated 8GB | 4 | 8 GB | 120 GB | $130 |
| DBaaS Dedicated 16GB | 8 | 16 GB | 240 GB | $260 |
| DBaaS Dedicated 32GB | 16 | 32 GB | 480 GB | $520 |

**HA Cluster**: 3× single node price

## Create Database

### CLI

```bash
# Create MySQL cluster
linode-cli databases mysql-create \
  --label my-mysql \
  --region us-east \
  --type g6-nanode-1 \
  --engine mysql/8.0.30 \
  --cluster_size 3 \
  --replication_type asynch \
  --ssl_connection true \
  --allow_list '["192.0.2.0/24", "203.0.113.5/32"]'

# Create PostgreSQL cluster
linode-cli databases postgresql-create \
  --label my-postgres \
  --region us-east \
  --type g6-standard-2 \
  --engine postgresql/15.4 \
  --cluster_size 3

# Create MongoDB cluster
linode-cli databases mongodb-create \
  --label my-mongodb \
  --region us-east \
  --type g6-standard-2 \
  --engine mongodb/6.0 \
  --cluster_size 3
```

### Terraform

```hcl
resource "linode_database_mysql" "main" {
  label         = "my-mysql"
  engine_id     = "mysql/8.0.30"
  region        = "us-east"
  type          = "g6-standard-2"
  cluster_size  = 3

  allow_list    = ["192.0.2.0/24"]
  ssl_connection = true

  updates {
    day_of_week   = "sunday"
    duration      = 3
    frequency     = "weekly"
    hour_of_day   = 3
  }
}

output "mysql_host" {
  value = linode_database_mysql.main.host_primary
}
```

## Connection Details

```bash
# Get database details
linode-cli databases mysql-view 12345

# Get credentials
linode-cli databases mysql-credentials 12345

# Output:
# username: linroot
# password: xxxxxxxxxx
```

### Connection Strings

```bash
# MySQL
mysql -h lin-12345-mysql-primary.servers.linodedb.net \
      -u linroot \
      -p \
      --ssl-mode=REQUIRED

# PostgreSQL
psql "host=lin-12345-pg-primary.servers.linodedb.net \
      user=linroot \
      dbname=defaultdb \
      sslmode=require"

# MongoDB
mongosh "mongodb://linroot:password@lin-12345-mongo.servers.linodedb.net:27017/admin?ssl=true&replicaSet=rs0"
```

### Application Configuration

```python
# Python MySQL
import mysql.connector

conn = mysql.connector.connect(
    host="lin-12345-mysql-primary.servers.linodedb.net",
    user="linroot",
    password="your-password",
    database="defaultdb",
    ssl_ca="/path/to/ca-certificate.crt"
)

# Python PostgreSQL
import psycopg2

conn = psycopg2.connect(
    host="lin-12345-pg-primary.servers.linodedb.net",
    user="linroot",
    password="your-password",
    dbname="defaultdb",
    sslmode="require"
)

# Python MongoDB
from pymongo import MongoClient

client = MongoClient(
    "mongodb://linroot:password@lin-12345-mongo.servers.linodedb.net:27017/admin",
    ssl=True,
    replicaSet="rs0"
)
```

## Access Control

### Allow List (IP Whitelist)

```bash
# Update allow list
linode-cli databases mysql-update 12345 \
  --allow_list '["192.0.2.0/24", "203.0.113.0/24"]'

# Allow from specific Linode
linode-cli databases mysql-update 12345 \
  --allow_list '["203.0.113.50/32"]'

# Allow from all (not recommended)
linode-cli databases mysql-update 12345 \
  --allow_list '["0.0.0.0/0"]'
```

### SSL/TLS

```bash
# Download CA certificate
linode-cli databases mysql-ssl 12345 --text

# Require SSL connections (default)
linode-cli databases mysql-update 12345 \
  --ssl_connection true
```

## Backups

### Automatic Backups

```
Backup Schedule:
+-- Daily automatic backups
+-- Retained for 7 days
+-- Point-in-time recovery
+-- Stored in same region
```

```bash
# List backups
linode-cli databases mysql-backups 12345

# Restore from backup (creates new cluster)
linode-cli databases mysql-backup-restore 12345 \
  --backup_id 54321 \
  --label my-mysql-restored \
  --type g6-standard-2
```

## Maintenance

### Maintenance Window

```bash
# Set maintenance window
linode-cli databases mysql-update 12345 \
  --updates.day_of_week sunday \
  --updates.duration 3 \
  --updates.frequency weekly \
  --updates.hour_of_day 3
```

### Engine Updates

```bash
# Update engine version
linode-cli databases mysql-update 12345 \
  --engine mysql/8.0.35

# Patch updates applied automatically during maintenance
```

## Resize Cluster

```bash
# Scale up (increase plan)
linode-cli databases mysql-update 12345 \
  --type g6-standard-4

# Note: Cannot scale down, cannot change cluster_size
```

## Monitoring

### Metrics Available

| Metric | Description |
|--------|-------------|
| CPU | CPU utilization percentage |
| Disk I/O | Read/write operations |
| Memory | RAM usage |
| Connections | Active connections |
| Network | Bytes in/out |

```bash
# Integrate with Linode Longview or external monitoring
# Use application-level monitoring (PMM, pganalyze, etc.)
```

## CLI Quick Reference

```bash
# MySQL
linode-cli databases mysql-list
linode-cli databases mysql-create --label db --region us-east --type g6-standard-2 --engine mysql/8.0.30
linode-cli databases mysql-view 12345
linode-cli databases mysql-credentials 12345
linode-cli databases mysql-ssl 12345
linode-cli databases mysql-update 12345 --allow_list '["10.0.0.0/8"]'
linode-cli databases mysql-delete 12345

# PostgreSQL
linode-cli databases postgresql-list
linode-cli databases postgresql-create --label db --region us-east --type g6-standard-2 --engine postgresql/15
linode-cli databases postgresql-view 12345
linode-cli databases postgresql-credentials 12345
linode-cli databases postgresql-delete 12345

# MongoDB
linode-cli databases mongodb-list
linode-cli databases mongodb-create --label db --region us-east --type g6-standard-2 --engine mongodb/6.0
linode-cli databases mongodb-view 12345
linode-cli databases mongodb-credentials 12345
linode-cli databases mongodb-delete 12345

# Backups
linode-cli databases mysql-backups 12345
linode-cli databases mysql-backup-restore 12345 --backup_id 54321
```

## Best Practices

```
1. Security
   +-- Use strict allow list (avoid 0.0.0.0/0)
   +-- Always enable SSL/TLS
   +-- Connect via private IP when possible
   +-- Rotate credentials periodically

2. High Availability
   +-- Use 3-node clusters for production
   +-- Test failover procedures
   +-- Monitor replication lag

3. Performance
   +-- Right-size your plan
   +-- Use connection pooling
   +-- Monitor slow queries
   +-- Consider read replicas for reads

4. Backups
   +-- Verify backup restores periodically
   +-- Keep application-level backups too
   +-- Document recovery procedures
```

## Gotchas

- Cannot resize to smaller plan
- Cannot change cluster_size after creation
- Default database name is "defaultdb"
- Root user is "linroot" (cannot change)
- Daily backup at consistent time (within maintenance window)
- Allow list changes take a few minutes
- Private networking requires VLAN setup
- MongoDB requires cluster_size 3 (replica set)
- No read replicas beyond 3-node HA
- Cannot customize database parameters (limited)

## Limits

| Resource | Limit |
|----------|-------|
| Databases per account | 25 |
| Allow list entries | 50 per cluster |
| Max connections | Varies by plan |
| Storage | Fixed per plan (auto-allocated) |
| Backup retention | 7 days |
| Max nodes | 3 per cluster |
