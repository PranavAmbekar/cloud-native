# AWS Disaster Recovery Guide

> Zero-downtime architecture and failover strategies for business continuity when entire regions fail.

## Real-World Context

In 2024, geopolitical conflicts demonstrated that even major cloud data centers can become inaccessible due to war, natural disasters, or infrastructure attacks. When an AWS region goes offline:

- Applications become unreachable
- Data may be temporarily or permanently lost
- Business operations halt
- Customer trust erodes

**This guide provides a battle-tested approach to ensure your applications survive regional outages with zero downtime.**

## Key Concepts

| Term | Definition |
|------|------------|
| **RTO** | Recovery Time Objective - Maximum acceptable downtime |
| **RPO** | Recovery Point Objective - Maximum acceptable data loss |
| **Failover** | Switching traffic from primary to secondary region |
| **Failback** | Returning to primary region after recovery |
| **Active-Active** | Both regions serve traffic simultaneously |
| **Active-Passive** | Secondary region on standby until failover |
| **Pilot Light** | Minimal resources running in DR region |
| **Warm Standby** | Scaled-down but functional DR environment |

## DR Strategy Comparison

| Strategy | RTO | RPO | Cost | Complexity |
|----------|-----|-----|------|------------|
| Backup & Restore | Hours | Hours | $ | Low |
| Pilot Light | Minutes | Minutes | $$ | Medium |
| Warm Standby | Minutes | Seconds | $$$ | Medium |
| Active-Active | Zero | Zero | $$$$ | High |

## Architecture: Multi-Region Active-Active

```
+------------------------------------------------------------------+
|                        GLOBAL LAYER                               |
|                                                                   |
|  +------------------------------------------------------------+  |
|  |                    Route 53 (Global DNS)                    |  |
|  |    Health Checks + Latency-Based + Failover Routing         |  |
|  +---------------------+--------------------+------------------+  |
|                        |                    |                     |
|          +-------------v------+    +--------v--------------+      |
|          |    Primary Region   |    |   Secondary Region    |      |
|          |     (me-south-1)    |    |     (eu-west-1)       |      |
+----------+---------------------+----+------------------------+-----+

+---------------------------+          +---------------------------+
|      PRIMARY REGION       |          |     SECONDARY REGION      |
|       me-south-1          |          |        eu-west-1          |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |   CloudFront Edge   |  |          |  |   CloudFront Edge   |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |        ALB          |  |          |  |        ALB          |  |
|  |   (Health Checks)   |  |          |  |   (Health Checks)   |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |     ECS / EKS       |  |          |  |     ECS / EKS       |  |
|  |   Auto Scaling      |  |          |  |   Auto Scaling      |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |   Aurora Global     |<-------------->|   Aurora Global     |  |
|  |   (Primary Writer)  |  | Replication |   (Read Replica)    |  |
|  +---------------------+  |          |  +---------------------+  |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |  DynamoDB Global    |<-------------->|  DynamoDB Global    |  |
|  |      Tables         |  |  Sync    |  |      Tables         |  |
|  +---------------------+  |          |  +---------------------+  |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |   S3 (Primary)      |-------------->|   S3 (Replica)       |  |
|  |                     |  |   CRR    |  |                     |  |
|  +---------------------+  |          |  +---------------------+  |
+---------------------------+          +---------------------------+
```

## Component-by-Component DR Setup

---

## 1. DNS & Traffic Management (Route 53)

### Health Check Configuration

```bash
# Create health check for primary region
aws route53 create-health-check \
  --caller-reference "primary-health-$(date +%s)" \
  --health-check-config '{
    "IPAddress": "PRIMARY_ALB_IP",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "app.example.com",
    "RequestInterval": 10,
    "FailureThreshold": 2,
    "EnableSNI": true
  }'

# Create health check for secondary region
aws route53 create-health-check \
  --caller-reference "secondary-health-$(date +%s)" \
  --health-check-config '{
    "IPAddress": "SECONDARY_ALB_IP",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "app.example.com",
    "RequestInterval": 10,
    "FailureThreshold": 2,
    "EnableSNI": true
  }'
```

### Failover DNS Records

```bash
# Primary record (failover routing)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "HealthCheckId": "PRIMARY_HEALTH_CHECK_ID",
        "AliasTarget": {
          "HostedZoneId": "ALB_HOSTED_ZONE_ID",
          "DNSName": "primary-alb.me-south-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Secondary record (failover routing)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "HealthCheckId": "SECONDARY_HEALTH_CHECK_ID",
        "AliasTarget": {
          "HostedZoneId": "ALB_HOSTED_ZONE_ID",
          "DNSName": "secondary-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Terraform: Route 53 Failover

```hcl
# Health checks
resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 2
  request_interval  = 10

  tags = {
    Name = "primary-health-check"
  }
}

resource "aws_route53_health_check" "secondary" {
  fqdn              = aws_lb.secondary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 2
  request_interval  = 10

  tags = {
    Name = "secondary-health-check"
  }
}

# Failover records
resource "aws_route53_record" "primary" {
  zone_id         = aws_route53_zone.main.zone_id
  name            = "app.example.com"
  type            = "A"
  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary" {
  zone_id         = aws_route53_zone.main.zone_id
  name            = "app.example.com"
  type            = "A"
  set_identifier  = "secondary"
  health_check_id = aws_route53_health_check.secondary.id

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

---

## 2. Database Replication

### Aurora Global Database

```
+---------------------------------------------------------------+
|                    Aurora Global Database                      |
|                                                                |
|   Primary Region (me-south-1)      Secondary Region (eu-west-1)|
|   +-------------------------+      +-------------------------+ |
|   |   Primary Cluster       |      |   Secondary Cluster     | |
|   |   +-------+ +-------+   |      |   +-------+ +-------+   | |
|   |   |Writer | |Reader |   | ---> |   |Reader | |Reader |   | |
|   |   +-------+ +-------+   | Async|   +-------+ +-------+   | |
|   |                         | Repl |                         | |
|   |   RPO: ~1 second        |      |   Promotes to Writer    | |
|   |   RTO: ~1 minute        |      |   on failover           | |
|   +-------------------------+      +-------------------------+ |
+---------------------------------------------------------------+
```

```bash
# Create Aurora Global Database
aws rds create-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --source-db-cluster-identifier arn:aws:rds:me-south-1:123456789:cluster:primary-cluster \
  --region me-south-1

# Add secondary region
aws rds create-db-cluster \
  --db-cluster-identifier secondary-cluster \
  --global-cluster-identifier my-global-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.04.0 \
  --region eu-west-1 \
  --db-subnet-group-name secondary-subnet-group
```

### Terraform: Aurora Global Database

```hcl
# Primary region cluster
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "my-global-cluster"
  engine                    = "aurora-mysql"
  engine_version            = "8.0.mysql_aurora.3.04.0"
  database_name             = "myapp"
  storage_encrypted         = true
}

resource "aws_rds_cluster" "primary" {
  provider                  = aws.primary
  cluster_identifier        = "primary-cluster"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = aws_rds_global_cluster.main.engine
  engine_version            = aws_rds_global_cluster.main.engine_version
  database_name             = "myapp"
  master_username           = var.db_username
  master_password           = var.db_password
  db_subnet_group_name      = aws_db_subnet_group.primary.name
  vpc_security_group_ids    = [aws_security_group.db_primary.id]

  backup_retention_period   = 35
  preferred_backup_window   = "02:00-03:00"
  skip_final_snapshot       = false
  final_snapshot_identifier = "primary-final-snapshot"
}

resource "aws_rds_cluster_instance" "primary" {
  count                = 2
  provider             = aws.primary
  identifier           = "primary-instance-${count.index}"
  cluster_identifier   = aws_rds_cluster.primary.id
  instance_class       = "db.r6g.xlarge"
  engine               = aws_rds_global_cluster.main.engine
  engine_version       = aws_rds_global_cluster.main.engine_version
  publicly_accessible  = false
}

# Secondary region cluster
resource "aws_rds_cluster" "secondary" {
  provider                  = aws.secondary
  cluster_identifier        = "secondary-cluster"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = aws_rds_global_cluster.main.engine
  engine_version            = aws_rds_global_cluster.main.engine_version
  db_subnet_group_name      = aws_db_subnet_group.secondary.name
  vpc_security_group_ids    = [aws_security_group.db_secondary.id]

  # Secondary cluster cannot set these
  skip_final_snapshot       = true

  depends_on = [aws_rds_cluster.primary]
}

resource "aws_rds_cluster_instance" "secondary" {
  count                = 2
  provider             = aws.secondary
  identifier           = "secondary-instance-${count.index}"
  cluster_identifier   = aws_rds_cluster.secondary.id
  instance_class       = "db.r6g.xlarge"
  engine               = aws_rds_global_cluster.main.engine
  engine_version       = aws_rds_global_cluster.main.engine_version
  publicly_accessible  = false
}
```

### Aurora Failover Procedure

```bash
# 1. Check replication lag before failover
aws rds describe-global-clusters \
  --global-cluster-identifier my-global-cluster \
  --query 'GlobalClusters[0].GlobalClusterMembers[*].{Cluster:DBClusterArn,IsWriter:IsWriter,Lag:GlobalWriteForwardingStatus}'

# 2. Initiate failover to secondary region
aws rds failover-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789:cluster:secondary-cluster \
  --region eu-west-1

# 3. Monitor failover progress
aws rds describe-global-clusters \
  --global-cluster-identifier my-global-cluster \
  --query 'GlobalClusters[0].Status'

# 4. Update application connection strings (if not using Route 53)
# Secondary cluster is now the writer
```

### DynamoDB Global Tables

```
+---------------------------------------------------------------+
|                    DynamoDB Global Tables                      |
|                                                                |
|   Region A (me-south-1)            Region B (eu-west-1)        |
|   +---------------------+          +---------------------+     |
|   |   Users Table       |  <---->  |   Users Table       |     |
|   |   Orders Table      |  <---->  |   Orders Table      |     |
|   |   Sessions Table    |  <---->  |   Sessions Table    |     |
|   +---------------------+          +---------------------+     |
|                                                                |
|   - Active-Active replication                                  |
|   - RPO: ~1 second (typical)                                   |
|   - RTO: Zero (both regions active)                            |
|   - Conflict resolution: Last writer wins                      |
+---------------------------------------------------------------+
```

```hcl
# Terraform: DynamoDB Global Table
resource "aws_dynamodb_table" "users" {
  provider         = aws.primary
  name             = "users"
  billing_mode     = "PAY_PER_REQUEST"
  hash_key         = "user_id"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  attribute {
    name = "user_id"
    type = "S"
  }

  replica {
    region_name = "eu-west-1"
  }

  replica {
    region_name = "us-east-1"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Environment = "production"
  }
}
```

---

## 3. Storage Replication (S3)

### Cross-Region Replication (CRR)

```
+---------------------------------------------------------------+
|                    S3 Cross-Region Replication                 |
|                                                                |
|   Source Bucket                    Destination Bucket          |
|   (me-south-1)                     (eu-west-1)                 |
|   +---------------------+          +---------------------+     |
|   | app-data-primary    | -------> | app-data-replica    |     |
|   |                     |   CRR    |                     |     |
|   | - User uploads      |          | - Replicated data   |     |
|   | - Application data  |          | - Same structure    |     |
|   | - Backups           |          | - Different bucket  |     |
|   +---------------------+          +---------------------+     |
|                                                                |
|   Replication Time: Usually < 15 minutes (99.99% within 15min) |
|   S3 RTC: 99.99% replicated within 15 minutes (SLA)            |
+---------------------------------------------------------------+
```

```bash
# Enable versioning (required for CRR)
aws s3api put-bucket-versioning \
  --bucket app-data-primary \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-versioning \
  --bucket app-data-replica \
  --versioning-configuration Status=Enabled

# Create replication configuration
aws s3api put-bucket-replication \
  --bucket app-data-primary \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789:role/s3-replication-role",
    "Rules": [{
      "ID": "ReplicateAll",
      "Status": "Enabled",
      "Priority": 1,
      "DeleteMarkerReplication": { "Status": "Enabled" },
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::app-data-replica",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": { "Minutes": 15 }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": { "Minutes": 15 }
        }
      }
    }]
  }'
```

### Terraform: S3 Cross-Region Replication

```hcl
# IAM role for replication
resource "aws_iam_role" "replication" {
  name = "s3-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "s3.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "replication" {
  name = "s3-replication-policy"
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Effect   = "Allow"
        Resource = aws_s3_bucket.primary.arn
      },
      {
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.primary.arn}/*"
      },
      {
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.replica.arn}/*"
      }
    ]
  })
}

# Source bucket
resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "app-data-primary-${var.account_id}"
}

resource "aws_s3_bucket_versioning" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Destination bucket
resource "aws_s3_bucket" "replica" {
  provider = aws.secondary
  bucket   = "app-data-replica-${var.account_id}"
}

resource "aws_s3_bucket_versioning" "replica" {
  provider = aws.secondary
  bucket   = aws_s3_bucket.replica.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Replication configuration
resource "aws_s3_bucket_replication_configuration" "primary" {
  provider   = aws.primary
  depends_on = [aws_s3_bucket_versioning.primary]

  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.primary.id

  rule {
    id     = "replicate-all"
    status = "Enabled"

    delete_marker_replication {
      status = "Enabled"
    }

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD"

      replication_time {
        status = "Enabled"
        time {
          minutes = 15
        }
      }

      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
    }
  }
}
```

---

## 4. Compute Layer DR

### ECS Multi-Region Deployment

```hcl
# Primary region ECS cluster
resource "aws_ecs_cluster" "primary" {
  provider = aws.primary
  name     = "app-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_service" "primary" {
  provider        = aws.primary
  name            = "app-service"
  cluster         = aws_ecs_cluster.primary.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  load_balancer {
    target_group_arn = aws_lb_target_group.primary.arn
    container_name   = "app"
    container_port   = 8080
  }

  network_configuration {
    subnets          = var.primary_private_subnets
    security_groups  = [aws_security_group.ecs_primary.id]
    assign_public_ip = false
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}

# Secondary region ECS cluster (identical setup)
resource "aws_ecs_cluster" "secondary" {
  provider = aws.secondary
  name     = "app-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_service" "secondary" {
  provider        = aws.secondary
  name            = "app-service"
  cluster         = aws_ecs_cluster.secondary.id
  task_definition = aws_ecs_task_definition.app_secondary.arn
  desired_count   = 3  # Keep running for Active-Active, or 0 for Warm Standby
  launch_type     = "FARGATE"

  load_balancer {
    target_group_arn = aws_lb_target_group.secondary.arn
    container_name   = "app"
    container_port   = 8080
  }

  network_configuration {
    subnets          = var.secondary_private_subnets
    security_groups  = [aws_security_group.ecs_secondary.id]
    assign_public_ip = false
  }
}
```

### ECR Cross-Region Replication

```hcl
resource "aws_ecr_replication_configuration" "main" {
  replication_configuration {
    rule {
      destination {
        region      = "eu-west-1"
        registry_id = data.aws_caller_identity.current.account_id
      }

      destination {
        region      = "us-east-1"
        registry_id = data.aws_caller_identity.current.account_id
      }
    }
  }
}
```

---

## 5. Backup Strategy

### AWS Backup Centralized Policy

```
+---------------------------------------------------------------+
|                     AWS Backup Strategy                        |
|                                                                |
|   Daily Backups        Weekly Backups       Monthly Backups    |
|   +-------------+      +-------------+      +-------------+    |
|   | Retain 7    |      | Retain 4    |      | Retain 12   |    |
|   | days        |      | weeks       |      | months      |    |
|   +-------------+      +-------------+      +-------------+    |
|                                                                |
|   Cross-Region Copy: All backups copied to DR region           |
|   Vault Lock: Immutable backups for compliance                 |
+---------------------------------------------------------------+
```

```hcl
# Backup vault in primary region
resource "aws_backup_vault" "primary" {
  provider = aws.primary
  name     = "primary-backup-vault"
}

# Backup vault in DR region
resource "aws_backup_vault" "dr" {
  provider = aws.secondary
  name     = "dr-backup-vault"
}

# Backup plan
resource "aws_backup_plan" "main" {
  provider = aws.primary
  name     = "production-backup-plan"

  # Daily backup - retain 7 days
  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 5 ? * * *)"  # 5 AM UTC daily

    lifecycle {
      delete_after = 7
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle {
        delete_after = 7
      }
    }
  }

  # Weekly backup - retain 4 weeks
  rule {
    rule_name         = "weekly-backup"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 5 ? * 1 *)"  # 5 AM UTC every Monday

    lifecycle {
      delete_after = 28
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle {
        delete_after = 28
      }
    }
  }

  # Monthly backup - retain 12 months
  rule {
    rule_name         = "monthly-backup"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 5 1 * ? *)"  # 5 AM UTC first of month

    lifecycle {
      cold_storage_after = 30
      delete_after       = 365
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle {
        cold_storage_after = 30
        delete_after       = 365
      }
    }
  }
}

# Backup selection - what to backup
resource "aws_backup_selection" "main" {
  provider     = aws.primary
  name         = "production-resources"
  plan_id      = aws_backup_plan.main.id
  iam_role_arn = aws_iam_role.backup.arn

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Backup"
    value = "true"
  }

  resources = [
    aws_rds_cluster.primary.arn,
    aws_dynamodb_table.users.arn,
    aws_efs_file_system.main.arn
  ]
}
```

---

## 6. Automated Failover with Lambda

### Failover Orchestrator

```python
# lambda/failover_orchestrator.py
import boto3
import json
import os

route53 = boto3.client('route53')
rds = boto3.client('rds')
sns = boto3.client('sns')

HOSTED_ZONE_ID = os.environ['HOSTED_ZONE_ID']
GLOBAL_CLUSTER_ID = os.environ['GLOBAL_CLUSTER_ID']
DR_CLUSTER_ARN = os.environ['DR_CLUSTER_ARN']
SNS_TOPIC_ARN = os.environ['SNS_TOPIC_ARN']

def lambda_handler(event, context):
    """
    Triggered by CloudWatch alarm when primary region health check fails.
    Orchestrates failover to DR region.
    """

    try:
        # 1. Verify primary is truly down (avoid false positives)
        if not verify_primary_failure():
            return {'status': 'Primary region recovered, no failover needed'}

        # 2. Promote Aurora secondary to primary
        promote_aurora_secondary()

        # 3. Update Route 53 (if not using automatic failover)
        # This is optional if you have failover routing configured

        # 4. Scale up DR region ECS services
        scale_dr_region()

        # 5. Send notification
        notify_team("FAILOVER COMPLETED", "Successfully failed over to DR region")

        return {'status': 'Failover completed successfully'}

    except Exception as e:
        notify_team("FAILOVER FAILED", str(e))
        raise

def verify_primary_failure():
    """Double-check primary region is actually down"""
    health = route53.get_health_check_status(
        HealthCheckId=os.environ['PRIMARY_HEALTH_CHECK_ID']
    )

    # Check if majority of health checkers report unhealthy
    statuses = health['HealthCheckObservations']
    unhealthy_count = sum(1 for s in statuses if s['StatusReport']['Status'] != 'Success')

    return unhealthy_count > len(statuses) / 2

def promote_aurora_secondary():
    """Promote Aurora secondary cluster to primary"""
    response = rds.failover_global_cluster(
        GlobalClusterIdentifier=GLOBAL_CLUSTER_ID,
        TargetDbClusterIdentifier=DR_CLUSTER_ARN
    )

    # Wait for failover to complete
    waiter = rds.get_waiter('db_cluster_available')
    waiter.wait(
        DBClusterIdentifier=DR_CLUSTER_ARN.split(':')[-1],
        WaiterConfig={'Delay': 30, 'MaxAttempts': 40}
    )

    return response

def scale_dr_region():
    """Scale up ECS services in DR region"""
    ecs = boto3.client('ecs', region_name=os.environ['DR_REGION'])

    ecs.update_service(
        cluster='app-cluster',
        service='app-service',
        desiredCount=int(os.environ['DR_DESIRED_COUNT'])
    )

def notify_team(subject, message):
    """Send SNS notification"""
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"[DR ALERT] {subject}",
        Message=message
    )
```

### CloudWatch Alarm Trigger

```hcl
resource "aws_cloudwatch_metric_alarm" "primary_health" {
  provider            = aws.primary
  alarm_name          = "primary-region-health-alarm"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthCheckStatus"
  namespace           = "AWS/Route53"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1
  alarm_description   = "Primary region health check failed"

  dimensions = {
    HealthCheckId = aws_route53_health_check.primary.id
  }

  alarm_actions = [aws_sns_topic.failover_trigger.arn]
}

resource "aws_sns_topic" "failover_trigger" {
  provider = aws.primary
  name     = "failover-trigger"
}

resource "aws_sns_topic_subscription" "failover_lambda" {
  provider  = aws.primary
  topic_arn = aws_sns_topic.failover_trigger.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.failover_orchestrator.arn
}
```

---

## 7. Zero-Downtime Checklist

### Pre-Disaster Preparation

```
+-- Infrastructure Setup
|   +-- [ ] Multi-region VPC with identical CIDR ranges
|   +-- [ ] Route 53 health checks configured
|   +-- [ ] Failover DNS records created
|   +-- [ ] Aurora Global Database enabled
|   +-- [ ] DynamoDB Global Tables configured
|   +-- [ ] S3 Cross-Region Replication active
|   +-- [ ] ECR replication enabled
|   +-- [ ] AWS Backup cross-region copy enabled
|
+-- Application Readiness
|   +-- [ ] Application supports database failover
|   +-- [ ] Connection strings use DNS (not IP)
|   +-- [ ] Graceful degradation implemented
|   +-- [ ] Session state externalized (Redis/DynamoDB)
|   +-- [ ] Static assets on CloudFront
|
+-- Monitoring & Alerting
|   +-- [ ] CloudWatch alarms for both regions
|   +-- [ ] Replication lag monitoring
|   +-- [ ] Health check dashboards
|   +-- [ ] PagerDuty/OpsGenie integration
|
+-- Documentation & Runbooks
    +-- [ ] Failover runbook documented
    +-- [ ] Failback procedure documented
    +-- [ ] Contact list updated
    +-- [ ] DR drill scheduled quarterly
```

### Failover Execution Steps

```
AUTOMATED FAILOVER (Target: < 5 minutes)
=========================================

1. Detection (0-60 seconds)
   +-- Route 53 health checks detect failure
   +-- CloudWatch alarm triggers
   +-- SNS notification sent

2. DNS Failover (60-120 seconds)
   +-- Route 53 automatically routes to secondary
   +-- TTL: 60 seconds (clients update)
   +-- Users start reaching DR region

3. Database Promotion (60-180 seconds)
   +-- Aurora secondary promoted to writer
   +-- Application reconnects automatically
   +-- DynamoDB Global Tables: instant (already active)

4. Scaling (Parallel)
   +-- ECS/EKS services scale up in DR region
   +-- Auto Scaling adjusts capacity

5. Verification (< 5 minutes total)
   +-- Health checks green in DR region
   +-- Application functional
   +-- Monitoring dashboards updated
```

---

## 8. Testing DR (Quarterly Drill)

### DR Test Procedure

```bash
#!/bin/bash
# dr-test.sh - Quarterly DR drill script

set -e

echo "=== Starting DR Drill ==="
DATE=$(date +%Y-%m-%d)

# 1. Notify team
aws sns publish \
  --topic-arn $SNS_TOPIC \
  --subject "[DR DRILL] Starting quarterly DR test" \
  --message "DR drill starting at $(date). Primary region will be failed over to DR."

# 2. Capture baseline metrics
echo "Capturing baseline metrics..."
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Sum > baseline-metrics.json

# 3. Simulate primary failure (disable health check endpoint)
echo "Simulating primary region failure..."
aws ssm send-command \
  --instance-ids $PRIMARY_INSTANCE_IDS \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["systemctl stop nginx"]'

# 4. Monitor failover
echo "Monitoring failover..."
for i in {1..10}; do
  HEALTH=$(aws route53 get-health-check-status --health-check-id $PRIMARY_HEALTH_CHECK_ID \
    --query 'HealthCheckObservations[0].StatusReport.Status' --output text)
  echo "Health check status: $HEALTH"
  sleep 30
done

# 5. Verify DR region is serving traffic
echo "Verifying DR region..."
curl -s https://app.example.com/health | jq .

# 6. Capture failover metrics
echo "Capturing failover metrics..."
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=$DR_ALB_ARN \
  --start-time $(date -u -v-30M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum > failover-metrics.json

# 7. Restore primary (failback)
echo "Restoring primary region..."
aws ssm send-command \
  --instance-ids $PRIMARY_INSTANCE_IDS \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["systemctl start nginx"]'

# 8. Wait for primary to recover
sleep 120

# 9. Generate report
echo "=== DR Drill Complete ==="
echo "Report saved to dr-drill-report-$DATE.json"
```

---

## 9. Cost Optimization

### DR Cost Strategies

| Strategy | Active-Active | Warm Standby | Pilot Light |
|----------|---------------|--------------|-------------|
| **Compute** | 100% duplicate | 25-50% capacity | Minimal |
| **Database** | Full replicas | Full replicas | Full replicas |
| **Storage** | CRR active | CRR active | Backups only |
| **Monthly Cost** | 2x primary | 1.3-1.5x primary | 1.1-1.2x primary |

### Cost-Saving Tips

```
1. Use Reserved Instances in DR region
   +-- Commit to DR capacity (you'll need it)
   +-- Savings Plans apply cross-region

2. Right-size DR compute
   +-- Warm Standby: Run 25% capacity normally
   +-- Scale up only during failover

3. Use S3 Intelligent-Tiering for replicas
   +-- Automatically moves to cheaper storage
   +-- No retrieval fees

4. Optimize Aurora
   +-- Use smaller instance class in DR for readers
   +-- Scale up during failover

5. Leverage Spot instances for non-critical workloads
   +-- Batch jobs can use Spot in DR region
```

---

## 10. Compliance & Audit

### Compliance Requirements

| Regulation | RTO Requirement | RPO Requirement |
|------------|-----------------|-----------------|
| SOC 2 | Documented | Documented |
| HIPAA | < 72 hours | < 24 hours |
| PCI DSS | < 24 hours | < 1 hour |
| GDPR | < 72 hours (breach notification) | Minimize data loss |
| Financial Services | < 4 hours | < 1 hour |

### Audit Trail

```hcl
# CloudTrail for DR audit
resource "aws_cloudtrail" "dr_audit" {
  name                          = "dr-audit-trail"
  s3_bucket_name                = aws_s3_bucket.audit_logs.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::RDS::DBCluster"
      values = ["arn:aws:rds:*"]
    }
  }
}
```

---

## Summary

### Quick Reference Card

```
+---------------------------------------------------------------+
|                    DR QUICK REFERENCE                          |
+---------------------------------------------------------------+
|                                                                |
|  PRIMARY REGION FAILURE DETECTED                               |
|  ================================                              |
|                                                                |
|  AUTOMATIC (if configured):                                    |
|  1. Route 53 fails over DNS (~60 seconds)                      |
|  2. Traffic routes to DR region                                |
|  3. Monitor: https://health.aws.amazon.com                     |
|                                                                |
|  MANUAL STEPS (if needed):                                     |
|  1. Promote Aurora secondary:                                  |
|     aws rds failover-global-cluster \                          |
|       --global-cluster-identifier my-global-cluster \          |
|       --target-db-cluster-identifier DR_CLUSTER_ARN            |
|                                                                |
|  2. Scale DR compute:                                          |
|     aws ecs update-service --cluster app --service app \       |
|       --desired-count 10 --region eu-west-1                    |
|                                                                |
|  3. Verify application:                                        |
|     curl https://app.example.com/health                        |
|                                                                |
|  CONTACTS:                                                     |
|  - On-call: PagerDuty escalation                               |
|  - AWS Support: Enterprise support case                        |
|  - Status: https://status.aws.amazon.com                       |
|                                                                |
+---------------------------------------------------------------+
```

## Gotchas

- Aurora Global Database failover takes 1-2 minutes (not instant)
- S3 CRR is eventually consistent (not immediate)
- Route 53 health checks have 10-30 second intervals
- DNS TTL affects failover speed (lower TTL = faster failover = more DNS queries)
- DynamoDB Global Tables can have ~1 second replication lag
- Cross-region data transfer costs add up
- DR region must have sufficient service limits
- Some services don't support cross-region replication natively
- Testing DR is essential but often skipped

## Related Documentation

- [AWS Multi-Region Architecture](https://aws.amazon.com/solutions/implementations/multi-region-application-architecture/)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Route 53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-types.html)
- [S3 Cross-Region Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
