# GCP Disaster Recovery Guide

> Zero-downtime architecture and failover strategies for business continuity when entire regions fail.

## Real-World Context

In 2026, during the US-Israel-Iran conflict, Iranian forces bombed cloud data centers in the Middle East region. While this specific incident targeted AWS, it demonstrated that **any cloud provider's regional infrastructure can be destroyed** by military action, natural disasters, or infrastructure attacks.

Companies relying on single-region deployments faced:

- Complete application outages lasting days or weeks
- Permanent data loss for single-region deployments
- Business shutdowns and customer exodus
- Existential threats without backup strategies

**This guide provides a battle-tested approach using GCP services to ensure your applications survive regional outages with zero downtime.**

## Key Concepts

| Term | Definition |
|------|------------|
| **RTO** | Recovery Time Objective - Maximum acceptable downtime |
| **RPO** | Recovery Point Objective - Maximum acceptable data loss |
| **Failover** | Switching traffic from primary to secondary region |
| **Failback** | Returning to primary region after recovery |
| **Active-Active** | Both regions serve traffic simultaneously |
| **Active-Passive** | Secondary region on standby until failover |
| **Multi-Regional** | Resources spanning multiple regions |
| **Dual-Region** | Storage replicated across 2 specific regions |

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
|  |              Cloud Load Balancing (Global)                  |  |
|  |        Health Checks + Geo-Routing + Failover               |  |
|  +---------------------+--------------------+------------------+  |
|                        |                    |                     |
|          +-------------v------+    +--------v--------------+      |
|          |   Primary Region    |    |  Secondary Region     |      |
|          |  (me-central1)      |    |  (europe-west1)       |      |
+----------+---------------------+----+------------------------+-----+

+---------------------------+          +---------------------------+
|      PRIMARY REGION       |          |     SECONDARY REGION      |
|       me-central1         |          |       europe-west1        |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |    Cloud CDN        |  |          |  |    Cloud CDN        |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |  Regional LB        |  |          |  |  Regional LB        |  |
|  |  (Health Checks)    |  |          |  |  (Health Checks)    |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |      GKE / GCE      |  |          |  |      GKE / GCE      |  |
|  |   Auto Scaling      |  |          |  |   Auto Scaling      |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |  Cloud SQL          |<-------------->|  Cloud SQL          |  |
|  |  (Primary)          |  | Cross-Rgn |  (Read Replica)      |  |
|  +---------------------+  | Replica   |  +---------------------+  |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |  Cloud Spanner      |<-------------->|  Cloud Spanner      |  |
|  |  (Multi-Region)     |  |  Sync    |  |  (Multi-Region)     |  |
|  +---------------------+  |          |  +---------------------+  |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |  GCS (Primary)      |-------------->|  GCS (Replica)       |  |
|  |  Dual-Region        |  |  Turbo   |  |  Dual-Region        |  |
|  +---------------------+  |          |  +---------------------+  |
+---------------------------+          +---------------------------+
```

## Component-by-Component DR Setup

---

## 1. Traffic Management (Cloud Load Balancing)

### Global HTTP(S) Load Balancer

```
+---------------------------------------------------------------+
|              Global HTTP(S) Load Balancer                      |
|                                                                |
|   Anycast IP (Single Global IP)                                |
|             |                                                  |
|             v                                                  |
|   +-------------------+    +-------------------+               |
|   | Backend Service   |    | Backend Service   |               |
|   | (me-central1)     |    | (europe-west1)    |               |
|   |                   |    |                   |               |
|   | Health Check: OK  |    | Health Check: OK  |               |
|   | Capacity: 100%    |    | Capacity: 100%    |               |
|   +-------------------+    +-------------------+               |
|                                                                |
|   - Routes to nearest healthy backend                          |
|   - Automatic failover if region unhealthy                     |
|   - Cross-region overflow for capacity                         |
+---------------------------------------------------------------+
```

### gcloud: Global Load Balancer Setup

```bash
# Create health check
gcloud compute health-checks create http app-health-check \
  --port=8080 \
  --request-path="/health" \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# Create backend service
gcloud compute backend-services create app-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=app-health-check \
  --global

# Add primary region backend (MIG)
gcloud compute backend-services add-backend app-backend \
  --instance-group=app-mig-primary \
  --instance-group-region=me-central1 \
  --balancing-mode=UTILIZATION \
  --max-utilization=0.8 \
  --capacity-scaler=1.0 \
  --global

# Add secondary region backend (MIG)
gcloud compute backend-services add-backend app-backend \
  --instance-group=app-mig-secondary \
  --instance-group-region=europe-west1 \
  --balancing-mode=UTILIZATION \
  --max-utilization=0.8 \
  --capacity-scaler=1.0 \
  --global

# Create URL map
gcloud compute url-maps create app-url-map \
  --default-service=app-backend

# Create HTTPS proxy
gcloud compute target-https-proxies create app-https-proxy \
  --url-map=app-url-map \
  --ssl-certificates=app-ssl-cert

# Create global forwarding rule
gcloud compute forwarding-rules create app-https-rule \
  --global \
  --target-https-proxy=app-https-proxy \
  --ports=443 \
  --ip-address=app-global-ip
```

### Terraform: Global Load Balancer with Failover

```hcl
# Health check
resource "google_compute_health_check" "app" {
  name               = "app-health-check"
  check_interval_sec = 10
  timeout_sec        = 5
  healthy_threshold  = 2
  unhealthy_threshold = 3

  http_health_check {
    port         = 8080
    request_path = "/health"
  }
}

# Backend service with multiple regions
resource "google_compute_backend_service" "app" {
  name                  = "app-backend"
  protocol              = "HTTP"
  port_name             = "http"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.app.id]
  load_balancing_scheme = "EXTERNAL"

  # Primary region backend
  backend {
    group           = google_compute_region_instance_group_manager.primary.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
    capacity_scaler = 1.0
  }

  # Secondary region backend
  backend {
    group           = google_compute_region_instance_group_manager.secondary.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
    capacity_scaler = 1.0
  }

  # Enable CDN
  enable_cdn = true
  cdn_policy {
    cache_mode        = "CACHE_ALL_STATIC"
    default_ttl       = 3600
    max_ttl           = 86400
    negative_caching  = true
  }
}

# URL map
resource "google_compute_url_map" "app" {
  name            = "app-url-map"
  default_service = google_compute_backend_service.app.id
}

# HTTPS proxy
resource "google_compute_target_https_proxy" "app" {
  name             = "app-https-proxy"
  url_map          = google_compute_url_map.app.id
  ssl_certificates = [google_compute_managed_ssl_certificate.app.id]
}

# Global IP
resource "google_compute_global_address" "app" {
  name = "app-global-ip"
}

# Forwarding rule
resource "google_compute_global_forwarding_rule" "app" {
  name       = "app-https-rule"
  target     = google_compute_target_https_proxy.app.id
  port_range = "443"
  ip_address = google_compute_global_address.app.address
}

# Managed SSL certificate
resource "google_compute_managed_ssl_certificate" "app" {
  name = "app-ssl-cert"

  managed {
    domains = ["app.example.com"]
  }
}
```

---

## 2. Database Replication

### Cloud SQL Cross-Region Replicas

```
+---------------------------------------------------------------+
|               Cloud SQL Cross-Region Replication               |
|                                                                |
|   Primary Region (me-central1)   Secondary Region (europe-west1)|
|   +-------------------------+    +-------------------------+   |
|   |   Primary Instance      |    |   Read Replica          |   |
|   |   +-------+ +-------+   |    |   +-------+ +-------+   |   |
|   |   |Read/  | |Compute|   |--->|   | Read  | |Compute|   |   |
|   |   |Write  | |       |   |Async|   | Only  | |       |   |   |
|   |   +-------+ +-------+   |Repl|   +-------+ +-------+   |   |
|   |                         |    |                         |   |
|   |   HA: Regional          |    |   Promotes to Primary   |   |
|   |   Backups: Automatic    |    |   on failover           |   |
|   +-------------------------+    +-------------------------+   |
|                                                                |
|   RPO: Seconds to minutes (async replication)                  |
|   RTO: Minutes (manual promotion required)                     |
+---------------------------------------------------------------+
```

### gcloud: Cloud SQL with Cross-Region Replica

```bash
# Create primary instance with HA
gcloud sql instances create myapp-db-primary \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-16384 \
  --region=me-central1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100GB \
  --storage-auto-increase \
  --backup-start-time=02:00 \
  --enable-point-in-time-recovery \
  --retained-backups-count=14 \
  --retained-transaction-log-days=7

# Create cross-region read replica
gcloud sql instances create myapp-db-replica \
  --master-instance-name=myapp-db-primary \
  --region=europe-west1 \
  --tier=db-custom-4-16384 \
  --availability-type=REGIONAL

# Promote replica to primary (failover)
gcloud sql instances promote-replica myapp-db-replica

# Check replication status
gcloud sql instances describe myapp-db-replica \
  --format="value(replicaConfiguration.failoverTarget)"
```

### Terraform: Cloud SQL with Cross-Region Replica

```hcl
# Primary instance
resource "google_sql_database_instance" "primary" {
  name             = "myapp-db-primary"
  database_version = "POSTGRES_15"
  region           = "me-central1"

  settings {
    tier              = "db-custom-4-16384"
    availability_type = "REGIONAL"  # HA within region
    disk_type         = "PD_SSD"
    disk_size         = 100
    disk_autoresize   = true

    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 14
      }
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }

    maintenance_window {
      day          = 7  # Sunday
      hour         = 3
      update_track = "stable"
    }
  }

  deletion_protection = true
}

# Cross-region read replica
resource "google_sql_database_instance" "replica" {
  name                 = "myapp-db-replica"
  master_instance_name = google_sql_database_instance.primary.name
  database_version     = "POSTGRES_15"
  region               = "europe-west1"

  replica_configuration {
    failover_target = true
  }

  settings {
    tier              = "db-custom-4-16384"
    availability_type = "REGIONAL"
    disk_type         = "PD_SSD"
    disk_size         = 100
    disk_autoresize   = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }
  }

  deletion_protection = true
}

# User for application
resource "google_sql_user" "app" {
  name     = "appuser"
  instance = google_sql_database_instance.primary.name
  password = var.db_password
}

# Database
resource "google_sql_database" "app" {
  name     = "appdb"
  instance = google_sql_database_instance.primary.name
}
```

### Cloud Spanner (Multi-Region)

```
+---------------------------------------------------------------+
|                   Cloud Spanner Multi-Region                   |
|                                                                |
|   Configuration: nam-eur-asia1 (or custom multi-region)        |
|                                                                |
|   +-------------------+  +-------------------+                 |
|   |   Region 1        |  |   Region 2        |                 |
|   |   (us-central1)   |  |   (europe-west1)  |                 |
|   |   +-----+ +-----+ |  |   +-----+ +-----+ |                 |
|   |   |Node | |Node | |  |   |Node | |Node | |                 |
|   |   +-----+ +-----+ |  |   +-----+ +-----+ |                 |
|   +-------------------+  +-------------------+                 |
|            |                      |                            |
|            +----------+-----------+                            |
|                       |                                        |
|            Synchronous Replication                             |
|            (Strong Consistency)                                |
|                                                                |
|   - RPO: 0 (synchronous replication)                           |
|   - RTO: ~0 (automatic failover)                               |
|   - Global strong consistency                                  |
+---------------------------------------------------------------+
```

```hcl
resource "google_spanner_instance" "main" {
  name         = "myapp-spanner"
  config       = "nam-eur-asia1"  # Multi-region configuration
  display_name = "MyApp Spanner"

  num_nodes = 3  # Per region

  labels = {
    environment = "production"
  }
}

resource "google_spanner_database" "app" {
  instance = google_spanner_instance.main.name
  name     = "appdb"

  version_retention_period = "7d"
  deletion_protection      = true

  ddl = [
    "CREATE TABLE Users (UserId INT64 NOT NULL, Name STRING(100)) PRIMARY KEY(UserId)"
  ]
}
```

### Firestore / Datastore (Multi-Region)

```hcl
resource "google_firestore_database" "main" {
  project     = var.project_id
  name        = "(default)"
  location_id = "nam5"  # Multi-region: US + Europe

  type             = "FIRESTORE_NATIVE"
  concurrency_mode = "OPTIMISTIC"

  point_in_time_recovery_enablement = "POINT_IN_TIME_RECOVERY_ENABLED"
}
```

---

## 3. Storage Replication (Cloud Storage)

### Dual-Region and Multi-Region Buckets

```
+---------------------------------------------------------------+
|              Cloud Storage Redundancy Options                  |
|                                                                |
|  +------------------+  +------------------+  +---------------+ |
|  |     Regional     |  |   Dual-Region    |  | Multi-Region  | |
|  | Single region    |  | 2 specific       |  | Auto-managed  | |
|  | (cheapest)       |  | regions          |  | (most durable)| |
|  +------------------+  +------------------+  +---------------+ |
|                                                                |
|  Turbo Replication (Dual-Region):                              |
|  - 100% of objects replicated within 15 minutes                |
|  - RPO: ~15 minutes guaranteed                                 |
|                                                                |
|  Multi-Region: us, eu, asia                                    |
|  Dual-Region: nam4 (Iowa+SC), eur4 (Netherlands+Finland)       |
+---------------------------------------------------------------+
```

### gcloud: Dual-Region Bucket with Turbo Replication

```bash
# Create dual-region bucket with turbo replication
gcloud storage buckets create gs://myapp-data-primary \
  --location=eur4 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access \
  --enable-turbo-replication

# Enable versioning
gcloud storage buckets update gs://myapp-data-primary \
  --versioning

# Enable object lifecycle
cat > lifecycle.json << EOF
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365, "isLive": false}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
    }
  ]
}
EOF

gcloud storage buckets update gs://myapp-data-primary \
  --lifecycle-file=lifecycle.json
```

### Terraform: Dual-Region Storage

```hcl
resource "google_storage_bucket" "primary" {
  name          = "myapp-data-${var.project_id}"
  location      = "EUR4"  # Dual-region: Netherlands + Finland
  storage_class = "STANDARD"

  uniform_bucket_level_access = true

  versioning {
    enabled = true
  }

  # Turbo replication for dual-region
  custom_placement_config {
    data_locations = ["EUROPE-WEST1", "EUROPE-WEST4"]
  }

  lifecycle_rule {
    condition {
      age = 30
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }

  lifecycle_rule {
    condition {
      age        = 365
      with_state = "ARCHIVED"
    }
    action {
      type = "Delete"
    }
  }

  # Soft delete for recovery
  soft_delete_policy {
    retention_duration_seconds = 604800  # 7 days
  }
}

# Multi-region bucket for critical data
resource "google_storage_bucket" "critical" {
  name          = "myapp-critical-${var.project_id}"
  location      = "EU"  # Multi-region
  storage_class = "STANDARD"

  uniform_bucket_level_access = true

  versioning {
    enabled = true
  }
}
```

---

## 4. Compute Multi-Region (GKE)

### GKE Multi-Cluster Architecture

```
+---------------------------------------------------------------+
|                   GKE Multi-Cluster Setup                      |
|                                                                |
|   +---------------------------+  +---------------------------+ |
|   |     GKE Cluster 1         |  |     GKE Cluster 2         | |
|   |     (me-central1)         |  |     (europe-west1)        | |
|   |                           |  |                           | |
|   |  +-----+ +-----+ +-----+  |  |  +-----+ +-----+ +-----+  | |
|   |  |Node | |Node | |Node |  |  |  |Node | |Node | |Node |  | |
|   |  +-----+ +-----+ +-----+  |  |  +-----+ +-----+ +-----+  | |
|   |                           |  |                           | |
|   |  +---------------------+  |  |  +---------------------+  | |
|   |  | Gateway / Ingress   |  |  |  | Gateway / Ingress   |  | |
|   |  +---------------------+  |  |  +---------------------+  | |
|   +-------------+-------------+  +-------------+-------------+ |
|                 |                              |               |
|                 +------------------------------+               |
|                              |                                 |
|                   +----------v----------+                      |
|                   | Multi-Cluster Ingress|                     |
|                   | (Global LB)          |                     |
|                   +---------------------+                      |
|                                                                |
|   Shared: Artifact Registry (multi-region), Cloud DNS          |
+---------------------------------------------------------------+
```

### Terraform: Multi-Region GKE

```hcl
# Primary GKE cluster
resource "google_container_cluster" "primary" {
  name     = "gke-primary"
  location = "me-central1"

  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1

  # Enable multi-cluster features
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Enable Gateway API
  gateway_api_config {
    channel = "CHANNEL_STANDARD"
  }

  release_channel {
    channel = "REGULAR"
  }

  # Enable backup
  addons_config {
    gke_backup_agent_config {
      enabled = true
    }
  }
}

resource "google_container_node_pool" "primary" {
  name       = "primary-pool"
  location   = "me-central1"
  cluster    = google_container_cluster.primary.name
  node_count = 3

  node_config {
    machine_type = "e2-standard-4"
    disk_size_gb = 100

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }

  autoscaling {
    min_node_count = 3
    max_node_count = 10
  }
}

# Secondary GKE cluster (DR)
resource "google_container_cluster" "secondary" {
  name     = "gke-secondary"
  location = "europe-west1"

  remove_default_node_pool = true
  initial_node_count       = 1

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  gateway_api_config {
    channel = "CHANNEL_STANDARD"
  }

  release_channel {
    channel = "REGULAR"
  }

  addons_config {
    gke_backup_agent_config {
      enabled = true
    }
  }
}

resource "google_container_node_pool" "secondary" {
  name       = "secondary-pool"
  location   = "europe-west1"
  cluster    = google_container_cluster.secondary.name
  node_count = 3

  node_config {
    machine_type = "e2-standard-4"
    disk_size_gb = 100

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }

  autoscaling {
    min_node_count = 3
    max_node_count = 10
  }
}

# Multi-region Artifact Registry
resource "google_artifact_registry_repository" "main" {
  location      = "us"  # Multi-region
  repository_id = "myapp"
  format        = "DOCKER"

  docker_config {
    immutable_tags = true
  }
}
```

### GKE Backup for DR

```hcl
# Backup plan for primary cluster
resource "google_gke_backup_backup_plan" "primary" {
  name     = "primary-backup-plan"
  cluster  = google_container_cluster.primary.id
  location = "me-central1"

  backup_config {
    include_volume_data = true
    include_secrets     = true
    all_namespaces      = true
  }

  backup_schedule {
    cron_schedule = "0 2 * * *"  # Daily at 2 AM
  }

  retention_policy {
    backup_delete_lock_days = 7
    backup_retain_days      = 30
  }
}

# Restore to secondary cluster
resource "google_gke_backup_restore_plan" "dr" {
  name             = "dr-restore-plan"
  location         = "europe-west1"
  backup_plan      = google_gke_backup_backup_plan.primary.id
  cluster          = google_container_cluster.secondary.id

  restore_config {
    all_namespaces                     = true
    namespaced_resource_restore_mode   = "MERGE_SKIP_ON_CONFLICT"
    volume_data_restore_policy         = "RESTORE_VOLUME_DATA_FROM_BACKUP"
    cluster_resource_restore_scope {
      all_group_kinds = true
    }
  }
}
```

---

## 5. Backup Strategy

### Google Cloud Backup

```
+---------------------------------------------------------------+
|                    GCP Backup Strategy                         |
|                                                                |
|   +------------------+  +------------------+                   |
|   | Cloud SQL        |  | GKE Backup       |                   |
|   | Auto Backups     |  | for GKE          |                   |
|   | - Daily          |  | - Daily          |                   |
|   | - PITR 7 days    |  | - Namespaces     |                   |
|   | - Retain 14 days |  | - Volumes        |                   |
|   +------------------+  +------------------+                   |
|                                                                |
|   +------------------+  +------------------+                   |
|   | GCS Versioning   |  | VM Snapshots     |                   |
|   | - Object versions|  | - Daily schedule |                   |
|   | - Soft delete    |  | - Multi-region   |                   |
|   | - Lifecycle mgmt |  | - Retain 14 days |                   |
|   +------------------+  +------------------+                   |
|                                                                |
|   Cross-Region: Backup data stored in multiple regions         |
+---------------------------------------------------------------+
```

### Terraform: Compute Engine Snapshot Schedule

```hcl
# Snapshot schedule policy
resource "google_compute_resource_policy" "daily_backup" {
  name   = "daily-backup-policy"
  region = "me-central1"

  snapshot_schedule_policy {
    schedule {
      daily_schedule {
        days_in_cycle = 1
        start_time    = "02:00"
      }
    }

    retention_policy {
      max_retention_days    = 14
      on_source_disk_delete = "KEEP_AUTO_SNAPSHOTS"
    }

    snapshot_properties {
      labels = {
        backup = "daily"
      }
      storage_locations = ["eu", "us"]  # Multi-region
    }
  }
}

# Attach policy to disk
resource "google_compute_disk_resource_policy_attachment" "app_disk" {
  name = google_compute_resource_policy.daily_backup.name
  disk = google_compute_disk.app_data.name
  zone = "me-central1-a"
}
```

---

## 6. Automated Failover with Cloud Functions

### Failover Orchestrator

```python
# main.py - Cloud Function for DR orchestration
import google.auth
from google.cloud import compute_v1
from google.cloud import sql_admin_v1
from google.cloud import monitoring_v3
import functions_framework

PROJECT_ID = "your-project-id"
PRIMARY_REGION = "me-central1"
DR_REGION = "europe-west1"
SQL_INSTANCE = "myapp-db-primary"
SQL_REPLICA = "myapp-db-replica"

@functions_framework.http
def failover_orchestrator(request):
    """
    Triggered by Cloud Monitoring alert when primary region fails.
    Orchestrates failover to DR region.
    """
    try:
        # 1. Verify primary is truly down
        if not verify_primary_failure():
            return {"status": "Primary region recovered, no failover needed"}

        # 2. Promote Cloud SQL replica
        promote_sql_replica()

        # 3. Scale up DR compute
        scale_dr_region()

        # 4. Send notification
        send_notification("FAILOVER COMPLETED", "Successfully failed over to DR region")

        return {"status": "Failover completed successfully"}

    except Exception as e:
        send_notification("FAILOVER FAILED", str(e))
        raise

def verify_primary_failure():
    """Check if primary region is actually down using uptime checks"""
    client = monitoring_v3.UptimeCheckServiceClient()

    # Get uptime check results
    # Return True if majority of checks fail
    return True  # Simplified for example

def promote_sql_replica():
    """Promote Cloud SQL replica to primary"""
    credentials, project = google.auth.default()
    client = sql_admin_v1.SqlInstancesServiceClient(credentials=credentials)

    # Promote replica
    request = sql_admin_v1.SqlInstancesPromoteReplicaRequest(
        project=PROJECT_ID,
        instance=SQL_REPLICA,
    )

    operation = client.promote_replica(request=request)
    operation.result()  # Wait for completion

    print(f"SQL replica {SQL_REPLICA} promoted to primary")

def scale_dr_region():
    """Scale up instance group in DR region"""
    client = compute_v1.InstanceGroupManagersClient()

    request = compute_v1.SetTargetPoolsInstanceGroupManagerRequest(
        project=PROJECT_ID,
        zone=f"{DR_REGION}-a",
        instance_group_manager="app-mig-secondary",
    )

    # Resize to production capacity
    resize_request = compute_v1.ResizeInstanceGroupManagerRequest(
        project=PROJECT_ID,
        zone=f"{DR_REGION}-a",
        instance_group_manager="app-mig-secondary",
        size=10,
    )

    client.resize(request=resize_request)
    print("DR instance group scaled up")

def send_notification(subject, message):
    """Send notification via Pub/Sub"""
    from google.cloud import pubsub_v1

    publisher = pubsub_v1.PublisherClient()
    topic_path = publisher.topic_path(PROJECT_ID, "dr-alerts")

    data = f"{subject}: {message}".encode("utf-8")
    publisher.publish(topic_path, data)
```

### Terraform: Cloud Function Deployment

```hcl
# Cloud Function for failover
resource "google_cloudfunctions2_function" "failover" {
  name     = "failover-orchestrator"
  location = "europe-west1"  # Deploy in DR region

  build_config {
    runtime     = "python311"
    entry_point = "failover_orchestrator"

    source {
      storage_source {
        bucket = google_storage_bucket.functions.name
        object = google_storage_bucket_object.function_zip.name
      }
    }
  }

  service_config {
    max_instance_count = 1
    available_memory   = "256M"
    timeout_seconds    = 540

    environment_variables = {
      PROJECT_ID     = var.project_id
      PRIMARY_REGION = "me-central1"
      DR_REGION      = "europe-west1"
    }

    service_account_email = google_service_account.failover.email
  }
}

# Uptime check for primary region
resource "google_monitoring_uptime_check_config" "primary" {
  display_name = "Primary Region Health"
  timeout      = "10s"
  period       = "60s"

  http_check {
    path         = "/health"
    port         = 443
    use_ssl      = true
    validate_ssl = true
  }

  monitored_resource {
    type = "uptime_url"
    labels = {
      project_id = var.project_id
      host       = "app-primary.example.com"
    }
  }
}

# Alert policy
resource "google_monitoring_alert_policy" "primary_down" {
  display_name = "Primary Region Down"
  combiner     = "OR"

  conditions {
    display_name = "Uptime check failed"

    condition_threshold {
      filter          = "metric.type=\"monitoring.googleapis.com/uptime_check/check_passed\" AND resource.type=\"uptime_url\""
      duration        = "300s"
      comparison      = "COMPARISON_LT"
      threshold_value = 1

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_NEXT_OLDER"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.pagerduty.id]

  alert_strategy {
    auto_close = "604800s"
  }
}
```

---

## 7. Zero-Downtime Checklist

### Pre-Disaster Preparation

```
+-- Infrastructure Setup
|   +-- [ ] Multi-region VPCs with Cloud Interconnect/VPN
|   +-- [ ] Global Load Balancer with health checks
|   +-- [ ] Cloud SQL cross-region replicas
|   +-- [ ] Cloud Spanner multi-region (if applicable)
|   +-- [ ] GCS dual-region with turbo replication
|   +-- [ ] Artifact Registry multi-region
|   +-- [ ] GKE clusters in both regions
|   +-- [ ] GKE Backup configured
|
+-- Application Readiness
|   +-- [ ] Application handles database failover
|   +-- [ ] Connection strings use Cloud SQL Proxy
|   +-- [ ] Graceful degradation implemented
|   +-- [ ] Session state in Memorystore/Firestore
|   +-- [ ] Static assets on Cloud CDN
|
+-- Monitoring & Alerting
|   +-- [ ] Uptime checks for both regions
|   +-- [ ] Cloud Monitoring dashboards
|   +-- [ ] Alert policies configured
|   +-- [ ] PagerDuty/Opsgenie integration
|
+-- Documentation & Runbooks
    +-- [ ] Failover runbook documented
    +-- [ ] Failback procedure documented
    +-- [ ] Contact list updated
    +-- [ ] DR drill scheduled quarterly
```

### Failover Execution

```
AUTOMATED FAILOVER (Target: < 5 minutes)
=========================================

1. Detection (0-60 seconds)
   +-- Uptime checks detect failure
   +-- Cloud Monitoring alerts trigger
   +-- Pub/Sub notification sent

2. Traffic Failover (0-60 seconds)
   +-- Global LB automatically routes to healthy backend
   +-- Users seamlessly redirected to DR region

3. Database Failover (60-300 seconds)
   +-- Cloud SQL replica promoted to primary
   +-- Application reconnects via Cloud SQL Proxy
   +-- Spanner/Firestore: Automatic (multi-region)

4. Scaling (Parallel)
   +-- GKE HPA scales pods in DR region
   +-- MIG scales instances

5. Verification
   +-- Health endpoints green
   +-- Application functional
   +-- Monitoring dashboards updated
```

---

## 8. DR Testing

### Quarterly DR Drill Script

```bash
#!/bin/bash
# gcp-dr-test.sh

set -e
DATE=$(date +%Y-%m-%d)
PROJECT_ID="your-project-id"

echo "=== Starting GCP DR Drill ==="

# 1. Notify team
gcloud pubsub topics publish dr-alerts \
  --message="DR Drill starting at $(date)"

# 2. Simulate primary failure by reducing capacity
echo "Simulating primary region failure..."
gcloud compute instance-groups managed resize app-mig-primary \
  --size=0 \
  --region=me-central1

# 3. Monitor Global LB failover
echo "Monitoring failover..."
for i in {1..10}; do
  curl -s -o /dev/null -w "%{http_code}" https://app.example.com/health
  sleep 30
done

# 4. Promote SQL replica (if testing database failover)
echo "Promoting SQL replica..."
gcloud sql instances promote-replica myapp-db-replica

# 5. Verify DR region serving traffic
echo "Verifying DR region..."
curl -s https://app.example.com/health | jq .

# 6. Restore primary (fail back)
echo "Restoring primary region..."
gcloud compute instance-groups managed resize app-mig-primary \
  --size=3 \
  --region=me-central1

# 7. Generate report
echo "=== DR Drill Complete ==="
echo "Report saved to dr-drill-$DATE.json"

gcloud pubsub topics publish dr-alerts \
  --message="DR Drill completed successfully at $(date)"
```

---

## 9. Cost Optimization

| Strategy | Active-Active | Warm Standby | Pilot Light |
|----------|---------------|--------------|-------------|
| **Compute** | 100% duplicate | 25-50% capacity | Minimal |
| **Database** | Full replicas | Full replicas | Full replicas |
| **Storage** | Dual-region | Dual-region | Regional |
| **Monthly Cost** | 2x primary | 1.3-1.5x primary | 1.1-1.2x primary |

### Cost-Saving Tips

```
1. Use Committed Use Discounts in DR region
   +-- 1-3 year commitments for predictable DR costs
   +-- Applies to Compute, Cloud SQL, GKE

2. Right-size DR compute
   +-- E2 instances for standby workloads
   +-- Scale up only during failover

3. Use Nearline/Coldline for older backups
   +-- Automatic lifecycle policies
   +-- Archive tier for compliance retention

4. Optimize Cloud SQL
   +-- Smaller machine type for replica
   +-- Scale up during failover

5. Use Preemptible/Spot VMs for non-critical
   +-- Batch jobs in DR region
   +-- Dev/test environments
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
|  1. Global LB routes to healthy backend (~0 seconds)           |
|  2. Traffic flows to DR region                                 |
|  3. Monitor: https://status.cloud.google.com                   |
|                                                                |
|  MANUAL STEPS (if needed):                                     |
|  1. Promote Cloud SQL replica:                                 |
|     gcloud sql instances promote-replica myapp-db-replica      |
|                                                                |
|  2. Scale DR compute:                                          |
|     gcloud compute instance-groups managed resize \            |
|       app-mig-secondary --size=10 --region=europe-west1        |
|                                                                |
|  3. Verify application:                                        |
|     curl https://app.example.com/health                        |
|                                                                |
|  CONTACTS:                                                     |
|  - On-call: PagerDuty escalation                               |
|  - GCP Support: Premium support case                           |
|  - Status: https://status.cloud.google.com                     |
|                                                                |
+---------------------------------------------------------------+
```

## Gotchas

- Cloud SQL replica promotion is irreversible (creates new primary)
- Global LB health checks have configurable intervals (default 5s)
- GCS dual-region turbo replication guarantees 15-minute RPO
- Cloud Spanner multi-region adds latency for writes
- Some services don't support cross-region replication natively
- DR region must have sufficient quota
- Cloud SQL replicas have async replication (seconds of lag)
- GKE Backup restores may need manual service recreation

## Related Documentation

- [GCP Disaster Recovery Planning Guide](https://cloud.google.com/architecture/dr-scenarios-planning-guide)
- [Cloud SQL High Availability](https://cloud.google.com/sql/docs/mysql/high-availability)
- [Cloud Spanner Multi-Region](https://cloud.google.com/spanner/docs/instances#multi-region-configurations)
- [Global Load Balancing](https://cloud.google.com/load-balancing/docs/https)
- [GKE Backup](https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/concepts/backup-for-gke)
- [Cloud Storage Redundancy](https://cloud.google.com/storage/docs/locations)
