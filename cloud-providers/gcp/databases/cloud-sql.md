# Google Cloud SQL

> Fully managed relational database service for MySQL, PostgreSQL, and SQL Server.

## Overview

Cloud SQL is a fully managed relational database service that makes it easy to set up, maintain, manage, and administer MySQL, PostgreSQL, and SQL Server databases on Google Cloud.

## Key Concepts

| Term | Definition |
|------|------------|
| Instance | A Cloud SQL database server |
| Database | Individual database within an instance |
| Connection | Method to access the database |
| Replica | Read copy of the primary instance |
| Backup | Point-in-time snapshot |
| Maintenance Window | Scheduled update period |

## Supported Databases

| Database | Versions |
|----------|----------|
| **MySQL** | 5.6, 5.7, 8.0 |
| **PostgreSQL** | 11, 12, 13, 14, 15, 16 |
| **SQL Server** | 2017, 2019, 2022 (Express, Web, Standard, Enterprise) |

## Architecture

```
+---------------------------------------------------------------+
|                      Cloud SQL Instance                       |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                    Primary Instance                     |  |
|  |  +---------------+   +-----------------------------+    |  |
|  |  |   Database    |   |       Configuration         |    |  |
|  |  |   Engine      |   |  * vCPUs: 4                 |    |  |
|  |  |  (MySQL/PG/   |   |  * Memory: 16 GB            |    |  |
|  |  |   SQL Server) |   |  * Storage: 100 GB SSD      |    |  |
|  |  +---------------+   |  * HA: Regional             |    |  |
|  |                      +-----------------------------+    |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                    Synchronous Replication                    |
|                              |                                |
|  +---------------------------------------------------------+  |
|  |                    Standby Instance                     |  |
|  |              (Different Zone, Auto-failover)            |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                   Asynchronous Replication                    |
|                              |                                |
|  +---------------------------------------------------------+  |
|  |                     Read Replicas                       |  |
|  |   +---------+   +---------+   +---------+               |  |
|  |   |Replica 1|   |Replica 2|   |Replica 3|               |  |
|  |   +---------+   +---------+   +---------+               |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

## Machine Types

| Type | vCPUs | Memory | Use Case |
|------|-------|--------|----------|
| **Shared-core** | 0.6-1 | 0.6-3.75 GB | Dev/test, low-traffic |
| **Lightweight** | 1-8 | 3.75-52 GB | Small production |
| **Standard** | 1-96 | 3.75-624 GB | General production |
| **High Memory** | 1-96 | 6.5-624 GB | Memory-intensive |
| **Custom** | Flexible | Flexible | Specific requirements |

## High Availability

### Regional HA

```
Primary Zone (us-central1-a)        Standby Zone (us-central1-b)
+-----------------------+           +-----------------------+
|   Primary Instance    |           |   Standby Instance    |
|   +---------------+   |           |   +---------------+   |
|   |   Database    |   | --sync--> |   |   Database    |   |
|   +---------------+   |           |   +---------------+   |
|   +---------------+   |           |   +---------------+   |
|   |  Persistent   |   |           |   |  Persistent   |   |
|   |    Disk       |   |           |   |    Disk       |   |
|   +---------------+   |           |   +---------------+   |
+-----------------------+           +-----------------------+
         |                                    ^
         | Automatic failover (~1-2 minutes)  |
         +------------------------------------+
```

```bash
# Create HA instance
gcloud sql instances create my-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100GB
```

## Read Replicas

```bash
# Create read replica
gcloud sql instances create my-replica \
  --master-instance-name=my-instance \
  --region=us-central1

# Create cross-region replica
gcloud sql instances create my-replica \
  --master-instance-name=my-instance \
  --region=europe-west1
```

### Read Replica Use Cases

- Read scaling (offload read queries)
- Analytics workloads
- Geographic distribution
- Disaster recovery (cross-region)

## Storage

| Type | IOPS | Use Case |
|------|------|----------|
| **SSD** | Higher | Production workloads |
| **HDD** | Lower | Cost-sensitive, low I/O |

### Storage Features

| Feature | Description |
|---------|-------------|
| Auto-resize | Automatically increase storage |
| Max size | 64 TB |
| Storage increase | Cannot decrease after increase |

```bash
# Enable automatic storage increase
gcloud sql instances patch my-instance \
  --storage-auto-increase \
  --storage-auto-increase-limit=500GB
```

## Connectivity

### Connection Methods

| Method | Description | Use Case |
|--------|-------------|----------|
| **Public IP** | Direct internet access | Quick setup, testing |
| **Private IP** | VPC-only access | Production, security |
| **Cloud SQL Proxy** | Encrypted tunnel | Secure without VPN |
| **Cloud SQL Connector** | Native library | Application integration |

### Private IP (VPC)

```bash
# Create instance with private IP
gcloud sql instances create my-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --network=projects/my-project/global/networks/my-vpc \
  --no-assign-ip
```

### Cloud SQL Auth Proxy

```bash
# Download proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64

# Connect
./cloud-sql-proxy --port 5432 my-project:us-central1:my-instance

# Connect from application
psql -h 127.0.0.1 -U postgres -d mydb
```

### Python with Cloud SQL Connector

```python
from google.cloud.sql.connector import Connector
import pg8000

connector = Connector()

def get_connection():
    return connector.connect(
        "my-project:us-central1:my-instance",
        "pg8000",
        user="postgres",
        password="password",
        db="mydb"
    )

# Use with SQLAlchemy
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+pg8000://",
    creator=get_connection
)
```

## Backups

### Automated Backups

```bash
# Configure automated backups
gcloud sql instances patch my-instance \
  --backup-start-time=02:00 \
  --backup-location=us

# Retention: 7 days (default), up to 365 days
gcloud sql instances patch my-instance \
  --retained-backups-count=30
```

### Point-in-Time Recovery (PITR)

```bash
# Enable PITR
gcloud sql instances patch my-instance \
  --enable-point-in-time-recovery

# Restore to point in time
gcloud sql instances clone my-instance my-restored-instance \
  --point-in-time="2024-01-15T10:00:00.000Z"
```

### On-demand Backup

```bash
# Create backup
gcloud sql backups create --instance=my-instance

# List backups
gcloud sql backups list --instance=my-instance

# Restore from backup
gcloud sql backups restore <backup-id> --restore-instance=my-instance
```

## Security

### Authorized Networks

```bash
# Allow specific IP
gcloud sql instances patch my-instance \
  --authorized-networks=1.2.3.4/32

# Allow multiple IPs
gcloud sql instances patch my-instance \
  --authorized-networks=1.2.3.4/32,5.6.7.8/32
```

### SSL/TLS

```bash
# Require SSL
gcloud sql instances patch my-instance \
  --require-ssl

# Create client certificate
gcloud sql ssl client-certs create my-cert \
  --instance=my-instance

# Download certificates
gcloud sql ssl client-certs describe my-cert \
  --instance=my-instance \
  --format="value(cert)" > client-cert.pem
```

### IAM Database Authentication

```bash
# Enable IAM authentication
gcloud sql instances patch my-instance \
  --database-flags=cloudsql.iam_authentication=on

# Create IAM user (PostgreSQL)
CREATE USER "user@example.com" WITH LOGIN;
GRANT ALL PRIVILEGES ON DATABASE mydb TO "user@example.com";
```

## Flags and Configuration

```bash
# Set database flags
gcloud sql instances patch my-instance \
  --database-flags=max_connections=500,log_min_duration_statement=1000

# List available flags
gcloud sql flags list --database-version=POSTGRES_15
```

### Common PostgreSQL Flags

| Flag | Description |
|------|-------------|
| max_connections | Maximum concurrent connections |
| log_min_duration_statement | Log slow queries (ms) |
| autovacuum | Enable autovacuum |
| work_mem | Memory for query operations |

## CLI Quick Reference

```bash
# Create instance
gcloud sql instances create my-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1

# List instances
gcloud sql instances list

# Describe instance
gcloud sql instances describe my-instance

# Connect to instance
gcloud sql connect my-instance --user=postgres

# Create database
gcloud sql databases create mydb --instance=my-instance

# Create user
gcloud sql users create myuser \
  --instance=my-instance \
  --password=mypassword

# Delete instance
gcloud sql instances delete my-instance

# Restart instance
gcloud sql instances restart my-instance

# Clone instance
gcloud sql instances clone my-instance my-clone
```

## Pricing Components

| Component | Description |
|-----------|-------------|
| Instance | Per vCPU and memory (per second) |
| Storage | Per GB (SSD or HDD) |
| Network | Egress charges |
| HA | ~2x instance cost |
| Backups | Per GB stored |
| IP addresses | Static IP charges |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Regional HA**: Automatic failover, ~2x cost
2. **Read replicas**: Async replication, read scaling
3. **Private IP**: Requires VPC, more secure
4. **Cloud SQL Proxy**: Encrypted tunnel, handles auth
5. **PITR**: Requires binary logging enabled
6. **Maintenance window**: Configure to minimize impact
7. **IAM auth**: Use service accounts for applications
8. **Storage auto-resize**: Cannot decrease after increase
9. **Cross-region replica**: Can be promoted for DR
10. **Connection limits**: Based on machine type

## Gotchas

- Cannot decrease storage size after increase
- Regional HA doubles the cost
- Maintenance updates may cause brief downtime
- Read replicas cannot be used for HA failover
- Private IP cannot be removed after adding
- Some flags require instance restart
- IAM authentication requires flag and user setup
- Connection pooling recommended for many connections
- Binary logging required for PITR and replication
- Instance names cannot be reused for 7 days after deletion

## Limits

| Resource | Limit |
|----------|-------|
| Storage per instance | 64 TB |
| Read replicas per primary | 10 |
| Connections per instance | Varies by tier |
| Databases per instance | No hard limit |
| Backups per instance | 365 |
| Maximum vCPUs | 96 |
| Maximum memory | 624 GB |
| Instances per project | ~100 (can increase) |
