# Azure Disaster Recovery Guide

> Zero-downtime architecture and failover strategies for business continuity when entire regions fail.

## Real-World Context

In 2026, during the US-Israel-Iran conflict, Iranian forces bombed cloud data centers in the Middle East region. While this specific incident targeted AWS, it demonstrated that **any cloud provider's regional infrastructure can be destroyed** by military action, natural disasters, or infrastructure attacks.

Companies relying on single-region deployments faced:

- Complete application outages lasting days or weeks
- Permanent data loss for single-region deployments
- Business shutdowns and customer exodus
- Existential threats without backup strategies

**This guide provides a battle-tested approach using Azure services to ensure your applications survive regional outages with zero downtime.**

## Key Concepts

| Term | Definition |
|------|------------|
| **RTO** | Recovery Time Objective - Maximum acceptable downtime |
| **RPO** | Recovery Point Objective - Maximum acceptable data loss |
| **Failover** | Switching traffic from primary to secondary region |
| **Failback** | Returning to primary region after recovery |
| **Active-Active** | Both regions serve traffic simultaneously |
| **Active-Passive** | Secondary region on standby until failover |
| **Azure Site Recovery** | Native DR orchestration service |
| **Geo-Redundant Storage** | Automatic cross-region replication |

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
|  |              Azure Front Door / Traffic Manager             |  |
|  |       Health Probes + Priority/Weighted/Geographic          |  |
|  +---------------------+--------------------+------------------+  |
|                        |                    |                     |
|          +-------------v------+    +--------v--------------+      |
|          |   Primary Region    |    |  Secondary Region     |      |
|          |    (UAE North)      |    |   (West Europe)       |      |
+----------+---------------------+----+------------------------+-----+

+---------------------------+          +---------------------------+
|      PRIMARY REGION       |          |     SECONDARY REGION      |
|        UAE North          |          |       West Europe         |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |    Front Door CDN   |  |          |  |    Front Door CDN   |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |  Application Gateway |  |          |  |  Application Gateway |  |
|  |   (Health Probes)    |  |          |  |   (Health Probes)    |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  |    AKS / App Service |  |          |  |    AKS / App Service |  |
|  |    Auto Scaling      |  |          |  |    Auto Scaling      |  |
|  +----------+----------+  |          |  +----------+----------+  |
|             |             |          |             |             |
|  +----------v----------+  |          |  +----------v----------+  |
|  | Azure SQL / Cosmos  |<-------------->| Azure SQL / Cosmos  |  |
|  |   (Geo-Replication)  |  | Active   |   (Geo-Secondary)    |  |
|  +---------------------+  | Geo-Repl |  +---------------------+  |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  |   Cosmos DB         |<-------------->|   Cosmos DB         |  |
|  |   (Multi-Region)    |  |  Sync    |  |   (Multi-Region)    |  |
|  +---------------------+  |          |  +---------------------+  |
|                           |          |                           |
|  +---------------------+  |          |  +---------------------+  |
|  | Storage (GRS/GZRS)  |-------------->| Storage (Secondary) |  |
|  |                     |  |   Async  |  |                     |  |
|  +---------------------+  |          |  +---------------------+  |
+---------------------------+          +---------------------------+
```

## Component-by-Component DR Setup

---

## 1. Traffic Management (Front Door / Traffic Manager)

### Azure Front Door (Recommended)

```
+---------------------------------------------------------------+
|                     Azure Front Door                           |
|                                                                |
|   Global Load Balancing + CDN + WAF + Health Probes            |
|                                                                |
|   +-------------------+    +-------------------+               |
|   | Origin Group 1    |    | Origin Group 2    |               |
|   | (Primary Region)  |    | (DR Region)       |               |
|   |                   |    |                   |               |
|   | Priority: 1       |    | Priority: 2       |               |
|   | Weight: 1000      |    | Weight: 1         |               |
|   +-------------------+    +-------------------+               |
|                                                                |
|   Health Probe: /health every 30s                              |
|   Failover: Automatic when primary unhealthy                   |
+---------------------------------------------------------------+
```

### CLI: Front Door Setup

```bash
# Create Front Door profile
az afd profile create \
  --profile-name myapp-frontdoor \
  --resource-group rg-global \
  --sku Premium_AzureFrontDoor

# Create endpoint
az afd endpoint create \
  --endpoint-name myapp \
  --profile-name myapp-frontdoor \
  --resource-group rg-global

# Create origin group with health probe
az afd origin-group create \
  --origin-group-name primary-origins \
  --profile-name myapp-frontdoor \
  --resource-group rg-global \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-path "/health" \
  --probe-interval-in-seconds 30 \
  --sample-size 4 \
  --successful-samples-required 3

# Add primary origin
az afd origin create \
  --origin-name primary-app \
  --origin-group-name primary-origins \
  --profile-name myapp-frontdoor \
  --resource-group rg-global \
  --host-name myapp-uaenorth.azurewebsites.net \
  --origin-host-header myapp-uaenorth.azurewebsites.net \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled \
  --http-port 80 \
  --https-port 443

# Add secondary origin (DR)
az afd origin create \
  --origin-name secondary-app \
  --origin-group-name primary-origins \
  --profile-name myapp-frontdoor \
  --resource-group rg-global \
  --host-name myapp-westeurope.azurewebsites.net \
  --origin-host-header myapp-westeurope.azurewebsites.net \
  --priority 2 \
  --weight 1 \
  --enabled-state Enabled \
  --http-port 80 \
  --https-port 443

# Create route
az afd route create \
  --route-name default-route \
  --endpoint-name myapp \
  --profile-name myapp-frontdoor \
  --resource-group rg-global \
  --origin-group primary-origins \
  --supported-protocols Https \
  --https-redirect Enabled \
  --patterns-to-match "/*"
```

### Terraform: Front Door with Failover

```hcl
resource "azurerm_cdn_frontdoor_profile" "main" {
  name                = "myapp-frontdoor"
  resource_group_name = azurerm_resource_group.global.name
  sku_name            = "Premium_AzureFrontDoor"
}

resource "azurerm_cdn_frontdoor_endpoint" "main" {
  name                     = "myapp"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id
}

resource "azurerm_cdn_frontdoor_origin_group" "main" {
  name                     = "primary-origins"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id
  session_affinity_enabled = false

  health_probe {
    interval_in_seconds = 30
    path                = "/health"
    protocol            = "Https"
    request_type        = "GET"
  }

  load_balancing {
    sample_size                        = 4
    successful_samples_required        = 3
    additional_latency_in_milliseconds = 50
  }
}

resource "azurerm_cdn_frontdoor_origin" "primary" {
  name                          = "primary-app"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.main.id
  enabled                       = true

  host_name          = azurerm_linux_web_app.primary.default_hostname
  origin_host_header = azurerm_linux_web_app.primary.default_hostname
  http_port          = 80
  https_port         = 443
  priority           = 1
  weight             = 1000

  certificate_name_check_enabled = true
}

resource "azurerm_cdn_frontdoor_origin" "secondary" {
  name                          = "secondary-app"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.main.id
  enabled                       = true

  host_name          = azurerm_linux_web_app.secondary.default_hostname
  origin_host_header = azurerm_linux_web_app.secondary.default_hostname
  http_port          = 80
  https_port         = 443
  priority           = 2
  weight             = 1

  certificate_name_check_enabled = true
}

resource "azurerm_cdn_frontdoor_route" "main" {
  name                          = "default-route"
  cdn_frontdoor_endpoint_id     = azurerm_cdn_frontdoor_endpoint.main.id
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.main.id
  cdn_frontdoor_origin_ids      = [
    azurerm_cdn_frontdoor_origin.primary.id,
    azurerm_cdn_frontdoor_origin.secondary.id
  ]

  supported_protocols    = ["Https"]
  patterns_to_match      = ["/*"]
  https_redirect_enabled = true
}
```

---

## 2. Database Replication

### Azure SQL Geo-Replication

```
+---------------------------------------------------------------+
|                 Azure SQL Geo-Replication                      |
|                                                                |
|   Primary Region (UAE North)     Secondary Region (West EU)   |
|   +-------------------------+    +-------------------------+  |
|   |   Primary Database      |    |   Geo-Secondary         |  |
|   |   +-------+ +-------+   |    |   +-------+ +-------+   |  |
|   |   |Read/  | |Compute|   |--->|   | Read  | |Compute|   |  |
|   |   |Write  | |       |   |Async|   | Only  | |       |   |  |
|   |   +-------+ +-------+   |Repl|   +-------+ +-------+   |  |
|   |                         |    |                         |  |
|   |   RPO: ~5 seconds       |    |   Promotes to Primary   |  |
|   |   RTO: ~30 seconds      |    |   on failover           |  |
|   +-------------------------+    +-------------------------+  |
|                                                                |
|   Auto-Failover Groups: Automatic failover with grace period   |
+---------------------------------------------------------------+
```

### CLI: Azure SQL Failover Group

```bash
# Create primary SQL server
az sql server create \
  --name myapp-sql-primary \
  --resource-group rg-primary \
  --location uaenorth \
  --admin-user sqladmin \
  --admin-password "SecurePassword123!"

# Create secondary SQL server
az sql server create \
  --name myapp-sql-secondary \
  --resource-group rg-secondary \
  --location westeurope \
  --admin-user sqladmin \
  --admin-password "SecurePassword123!"

# Create primary database
az sql db create \
  --name myappdb \
  --server myapp-sql-primary \
  --resource-group rg-primary \
  --service-objective S3 \
  --backup-storage-redundancy Geo

# Create failover group
az sql failover-group create \
  --name myapp-fog \
  --server myapp-sql-primary \
  --resource-group rg-primary \
  --partner-server myapp-sql-secondary \
  --partner-resource-group rg-secondary \
  --failover-policy Automatic \
  --grace-period 1 \
  --add-db myappdb

# Check failover group status
az sql failover-group show \
  --name myapp-fog \
  --server myapp-sql-primary \
  --resource-group rg-primary

# Manual failover (for testing)
az sql failover-group set-primary \
  --name myapp-fog \
  --server myapp-sql-secondary \
  --resource-group rg-secondary
```

### Terraform: Azure SQL Failover Group

```hcl
# Primary SQL Server
resource "azurerm_mssql_server" "primary" {
  name                         = "myapp-sql-primary"
  resource_group_name          = azurerm_resource_group.primary.name
  location                     = "uaenorth"
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = var.sql_admin_password

  identity {
    type = "SystemAssigned"
  }
}

# Secondary SQL Server
resource "azurerm_mssql_server" "secondary" {
  name                         = "myapp-sql-secondary"
  resource_group_name          = azurerm_resource_group.secondary.name
  location                     = "westeurope"
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = var.sql_admin_password

  identity {
    type = "SystemAssigned"
  }
}

# Primary Database
resource "azurerm_mssql_database" "primary" {
  name                        = "myappdb"
  server_id                   = azurerm_mssql_server.primary.id
  sku_name                    = "S3"
  geo_backup_enabled          = true
  storage_account_type        = "Geo"
  zone_redundant              = true

  short_term_retention_policy {
    retention_days           = 7
    backup_interval_in_hours = 12
  }

  long_term_retention_policy {
    weekly_retention  = "P4W"
    monthly_retention = "P12M"
    yearly_retention  = "P5Y"
    week_of_year      = 1
  }
}

# Failover Group
resource "azurerm_mssql_failover_group" "main" {
  name      = "myapp-fog"
  server_id = azurerm_mssql_server.primary.id

  databases = [azurerm_mssql_database.primary.id]

  partner_server {
    id = azurerm_mssql_server.secondary.id
  }

  read_write_endpoint_failover_policy {
    mode          = "Automatic"
    grace_minutes = 60
  }

  readonly_endpoint_failover_policy_enabled = true
}

output "sql_failover_group_endpoint" {
  value = "${azurerm_mssql_failover_group.main.name}.database.windows.net"
}
```

### Connection String (Use Failover Group Endpoint)

```python
# Python - Use failover group endpoint for automatic failover
import pyodbc

# Always use the failover group endpoint, NOT individual server endpoints
connection_string = (
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=myapp-fog.database.windows.net;"  # Failover group endpoint
    "Database=myappdb;"
    "UID=sqladmin;"
    "PWD=SecurePassword123!;"
    "Encrypt=yes;"
    "TrustServerCertificate=no;"
)

conn = pyodbc.connect(connection_string)
```

### Cosmos DB Multi-Region

```
+---------------------------------------------------------------+
|                    Cosmos DB Multi-Region                      |
|                                                                |
|   Write Region (UAE North)       Read Regions                  |
|   +---------------------+        +---------------------+       |
|   |   UAE North         |<------>|   West Europe       |       |
|   |   (Write + Read)    |  Sync  |   (Read Replica)    |       |
|   +---------------------+        +---------------------+       |
|            ^                              ^                    |
|            |                              |                    |
|            v                              v                    |
|   +---------------------+        +---------------------+       |
|   |   East US           |        |   Southeast Asia    |       |
|   |   (Read Replica)    |        |   (Read Replica)    |       |
|   +---------------------+        +---------------------+       |
|                                                                |
|   - Automatic failover to any read region                      |
|   - RPO: 0 (strong consistency) or seconds (eventual)          |
|   - RTO: ~1 minute                                             |
+---------------------------------------------------------------+
```

```hcl
resource "azurerm_cosmosdb_account" "main" {
  name                = "myapp-cosmos"
  location            = azurerm_resource_group.primary.location
  resource_group_name = azurerm_resource_group.primary.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  automatic_failover_enabled = true

  consistency_policy {
    consistency_level       = "Session"
    max_interval_in_seconds = 5
    max_staleness_prefix    = 100
  }

  # Primary write region
  geo_location {
    location          = "uaenorth"
    failover_priority = 0
    zone_redundant    = true
  }

  # Secondary region (auto-failover target)
  geo_location {
    location          = "westeurope"
    failover_priority = 1
    zone_redundant    = true
  }

  # Additional read regions
  geo_location {
    location          = "eastus"
    failover_priority = 2
    zone_redundant    = true
  }

  backup {
    type                = "Continuous"
    tier                = "Continuous30Days"
  }
}
```

---

## 3. Storage Replication

### Storage Account Geo-Redundancy

```
+---------------------------------------------------------------+
|              Azure Storage Redundancy Options                  |
|                                                                |
|  +------------------+  +------------------+  +---------------+ |
|  |       LRS        |  |       ZRS        |  |     GRS       | |
|  | Local Redundant  |  | Zone Redundant   |  | Geo Redundant | |
|  | 3 copies in 1 DC |  | 3 copies in 3 AZ |  | 6 copies      | |
|  +------------------+  +------------------+  | (3 local +    | |
|                                              | 3 in DR)      | |
|                                              +---------------+ |
|                                                                |
|  +------------------+  +------------------+                    |
|  |      GZRS        |  |     RA-GZRS      |                    |
|  | Geo-Zone         |  | Read-Access      |                    |
|  | Redundant        |  | Geo-Zone         |                    |
|  | (Best for DR)    |  | (Best for DR     |                    |
|  |                  |  |  with read)      |                    |
|  +------------------+  +------------------+                    |
+---------------------------------------------------------------+
```

### CLI: Geo-Redundant Storage

```bash
# Create RA-GZRS storage account
az storage account create \
  --name myappstorprimary \
  --resource-group rg-primary \
  --location uaenorth \
  --sku Standard_RAGZRS \
  --kind StorageV2 \
  --access-tier Hot \
  --min-tls-version TLS1_2

# Check replication status
az storage account show \
  --name myappstorprimary \
  --resource-group rg-primary \
  --query "{sku:sku.name, secondaryLocation:secondaryLocation, statusOfSecondary:statusOfSecondary}"

# Initiate failover (when primary unavailable)
az storage account failover \
  --name myappstorprimary \
  --resource-group rg-primary \
  --no-wait
```

### Terraform: Geo-Redundant Storage

```hcl
resource "azurerm_storage_account" "main" {
  name                     = "myappstorprimary"
  resource_group_name      = azurerm_resource_group.primary.name
  location                 = "uaenorth"
  account_tier             = "Standard"
  account_replication_type = "RAGZRS"  # Read-Access Geo-Zone Redundant
  account_kind             = "StorageV2"
  min_tls_version          = "TLS1_2"

  blob_properties {
    versioning_enabled = true

    delete_retention_policy {
      days = 30
    }

    container_delete_retention_policy {
      days = 30
    }
  }
}

# Output secondary endpoint for reads
output "storage_secondary_blob_endpoint" {
  value = azurerm_storage_account.main.secondary_blob_endpoint
}
```

---

## 4. Azure Site Recovery (ASR)

### ASR Architecture

```
+---------------------------------------------------------------+
|                   Azure Site Recovery                          |
|                                                                |
|   Source Region (UAE North)        Target Region (West EU)    |
|   +-------------------------+      +-------------------------+ |
|   |  +-------+ +-------+    |      |  +-------+ +-------+    | |
|   |  | VM 1  | | VM 2  |    |      |  | VM 1  | | VM 2  |    | |
|   |  +---+---+ +---+---+    |      |  |(Replica)|(Replica)|  | |
|   |      |         |        |      |  +-------+ +-------+    | |
|   |      v         v        |      |       (Dormant)         | |
|   |  +---------------+      |      +-------------------------+ |
|   |  |  ASR Agent    |      |               ^                  |
|   |  +-------+-------+      |               |                  |
|   |          |              |               |                  |
|   +----------|------- ------+               |                  |
|              |                              |                  |
|              +------------------------------+                  |
|                  Continuous Replication                        |
|                  (RPO: 30 seconds typical)                     |
+---------------------------------------------------------------+
```

### CLI: Azure Site Recovery Setup

```bash
# Create Recovery Services vault
az backup vault create \
  --name myapp-recovery-vault \
  --resource-group rg-dr \
  --location westeurope

# Enable replication for VM
az site-recovery replication-protection-container-mapping create \
  --fabric-name "Azure" \
  --name "UAE-to-WestEU" \
  --protection-container "uaenorth" \
  --recovery-fabric-name "Azure" \
  --recovery-protection-container "westeurope" \
  --resource-group rg-dr \
  --vault-name myapp-recovery-vault \
  --policy-id "/subscriptions/.../replicationPolicies/24-hour-retention"

# Test failover
az site-recovery replication-protected-item test-failover \
  --fabric-name "Azure" \
  --name "vm-web-1" \
  --protection-container "uaenorth" \
  --resource-group rg-dr \
  --vault-name myapp-recovery-vault \
  --failover-direction PrimaryToRecovery \
  --network-id "/subscriptions/.../virtualNetworks/vnet-dr"

# Actual failover
az site-recovery replication-protected-item failover \
  --fabric-name "Azure" \
  --name "vm-web-1" \
  --protection-container "uaenorth" \
  --resource-group rg-dr \
  --vault-name myapp-recovery-vault \
  --failover-direction PrimaryToRecovery
```

### Terraform: Azure Site Recovery

```hcl
# Recovery Services Vault
resource "azurerm_recovery_services_vault" "main" {
  name                = "myapp-recovery-vault"
  location            = azurerm_resource_group.dr.location
  resource_group_name = azurerm_resource_group.dr.name
  sku                 = "Standard"

  soft_delete_enabled = true

  identity {
    type = "SystemAssigned"
  }
}

# Replication Policy
resource "azurerm_site_recovery_replication_policy" "main" {
  name                                                 = "24-hour-retention"
  resource_group_name                                  = azurerm_resource_group.dr.name
  recovery_vault_name                                  = azurerm_recovery_services_vault.main.name
  recovery_point_retention_in_minutes                  = 1440  # 24 hours
  application_consistent_snapshot_frequency_in_minutes = 60    # 1 hour
}

# Primary fabric (source region)
resource "azurerm_site_recovery_fabric" "primary" {
  name                = "primary-fabric"
  resource_group_name = azurerm_resource_group.dr.name
  recovery_vault_name = azurerm_recovery_services_vault.main.name
  location            = "uaenorth"
}

# Secondary fabric (DR region)
resource "azurerm_site_recovery_fabric" "secondary" {
  name                = "secondary-fabric"
  resource_group_name = azurerm_resource_group.dr.name
  recovery_vault_name = azurerm_recovery_services_vault.main.name
  location            = "westeurope"

  depends_on = [azurerm_site_recovery_fabric.primary]
}

# Protection containers
resource "azurerm_site_recovery_protection_container" "primary" {
  name                 = "primary-container"
  resource_group_name  = azurerm_resource_group.dr.name
  recovery_vault_name  = azurerm_recovery_services_vault.main.name
  recovery_fabric_name = azurerm_site_recovery_fabric.primary.name
}

resource "azurerm_site_recovery_protection_container" "secondary" {
  name                 = "secondary-container"
  resource_group_name  = azurerm_resource_group.dr.name
  recovery_vault_name  = azurerm_recovery_services_vault.main.name
  recovery_fabric_name = azurerm_site_recovery_fabric.secondary.name
}

# Container mapping
resource "azurerm_site_recovery_protection_container_mapping" "main" {
  name                                      = "primary-to-secondary"
  resource_group_name                       = azurerm_resource_group.dr.name
  recovery_vault_name                       = azurerm_recovery_services_vault.main.name
  recovery_fabric_name                      = azurerm_site_recovery_fabric.primary.name
  recovery_source_protection_container_name = azurerm_site_recovery_protection_container.primary.name
  recovery_target_protection_container_id   = azurerm_site_recovery_protection_container.secondary.id
  recovery_replication_policy_id            = azurerm_site_recovery_replication_policy.main.id
}
```

---

## 5. Azure Backup

### Backup Strategy

```
+---------------------------------------------------------------+
|                     Azure Backup Strategy                      |
|                                                                |
|   Daily Backups        Weekly Backups       Monthly Backups    |
|   +-------------+      +-------------+      +-------------+    |
|   | Retain 7    |      | Retain 4    |      | Retain 12   |    |
|   | days        |      | weeks       |      | months      |    |
|   +-------------+      +-------------+      +-------------+    |
|                                                                |
|   Yearly Backups       Cross-Region Copy                       |
|   +-------------+      +----------------------------------+    |
|   | Retain 10   |      | All backups replicated to DR     |    |
|   | years       |      | Recovery Services Vault          |    |
|   +-------------+      +----------------------------------+    |
+---------------------------------------------------------------+
```

### Terraform: Azure Backup Policy

```hcl
resource "azurerm_recovery_services_vault" "backup" {
  name                = "myapp-backup-vault"
  location            = azurerm_resource_group.primary.location
  resource_group_name = azurerm_resource_group.primary.name
  sku                 = "Standard"
  storage_mode_type   = "GeoRedundant"

  cross_region_restore_enabled = true
}

resource "azurerm_backup_policy_vm" "main" {
  name                = "production-backup-policy"
  resource_group_name = azurerm_resource_group.primary.name
  recovery_vault_name = azurerm_recovery_services_vault.backup.name

  timezone = "UTC"

  backup {
    frequency = "Daily"
    time      = "02:00"
  }

  retention_daily {
    count = 7
  }

  retention_weekly {
    count    = 4
    weekdays = ["Sunday"]
  }

  retention_monthly {
    count    = 12
    weekdays = ["Sunday"]
    weeks    = ["First"]
  }

  retention_yearly {
    count    = 10
    weekdays = ["Sunday"]
    weeks    = ["First"]
    months   = ["January"]
  }
}

# Protect VMs
resource "azurerm_backup_protected_vm" "web" {
  resource_group_name = azurerm_resource_group.primary.name
  recovery_vault_name = azurerm_recovery_services_vault.backup.name
  source_vm_id        = azurerm_linux_virtual_machine.web.id
  backup_policy_id    = azurerm_backup_policy_vm.main.id
}
```

---

## 6. Compute Multi-Region (AKS)

### AKS Multi-Region Architecture

```
+---------------------------------------------------------------+
|                   AKS Multi-Region Setup                       |
|                                                                |
|   +---------------------------+  +---------------------------+ |
|   |     AKS Cluster 1         |  |     AKS Cluster 2         | |
|   |     (UAE North)           |  |     (West Europe)         | |
|   |                           |  |                           | |
|   |  +-----+ +-----+ +-----+  |  |  +-----+ +-----+ +-----+  | |
|   |  |Node | |Node | |Node |  |  |  |Node | |Node | |Node |  | |
|   |  +-----+ +-----+ +-----+  |  |  +-----+ +-----+ +-----+  | |
|   |                           |  |                           | |
|   |  +---------------------+  |  |  +---------------------+  | |
|   |  |   Ingress (NGINX)   |  |  |  |   Ingress (NGINX)   |  | |
|   |  +---------------------+  |  |  +---------------------+  | |
|   +-------------+-------------+  +-------------+-------------+ |
|                 |                              |               |
|                 +------------------------------+               |
|                              |                                 |
|                   +----------v----------+                      |
|                   |    Azure Front Door  |                      |
|                   +---------------------+                      |
|                                                                |
|   Shared: ACR (geo-replicated), Azure DNS, Key Vault           |
+---------------------------------------------------------------+
```

### Terraform: Multi-Region AKS

```hcl
# Primary AKS Cluster
resource "azurerm_kubernetes_cluster" "primary" {
  name                = "aks-primary"
  location            = "uaenorth"
  resource_group_name = azurerm_resource_group.primary.name
  dns_prefix          = "aksprimary"

  default_node_pool {
    name                = "default"
    node_count          = 3
    vm_size             = "Standard_D4s_v3"
    availability_zones  = ["1", "2", "3"]
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 10
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }
}

# Secondary AKS Cluster (DR)
resource "azurerm_kubernetes_cluster" "secondary" {
  name                = "aks-secondary"
  location            = "westeurope"
  resource_group_name = azurerm_resource_group.secondary.name
  dns_prefix          = "akssecondary"

  default_node_pool {
    name                = "default"
    node_count          = 3
    vm_size             = "Standard_D4s_v3"
    availability_zones  = ["1", "2", "3"]
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 10
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }
}

# Geo-replicated ACR
resource "azurerm_container_registry" "main" {
  name                = "myappacr"
  resource_group_name = azurerm_resource_group.global.name
  location            = "uaenorth"
  sku                 = "Premium"
  admin_enabled       = false

  georeplications {
    location                = "westeurope"
    zone_redundancy_enabled = true
  }

  georeplications {
    location                = "eastus"
    zone_redundancy_enabled = true
  }
}
```

---

## 7. Automated Failover with Azure Functions

### Failover Orchestrator

```python
# function_app/failover_orchestrator/__init__.py
import azure.functions as func
import logging
from azure.identity import DefaultAzureCredential
from azure.mgmt.sql import SqlManagementClient
from azure.mgmt.trafficmanager import TrafficManagerManagementClient

def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Failover orchestrator triggered')

    credential = DefaultAzureCredential()
    subscription_id = os.environ['AZURE_SUBSCRIPTION_ID']

    try:
        # 1. Failover SQL
        failover_sql(credential, subscription_id)

        # 2. Update Traffic Manager
        update_traffic_manager(credential, subscription_id)

        # 3. Scale up DR AKS
        scale_dr_cluster(credential, subscription_id)

        return func.HttpResponse("Failover completed successfully", status_code=200)

    except Exception as e:
        logging.error(f"Failover failed: {str(e)}")
        return func.HttpResponse(f"Failover failed: {str(e)}", status_code=500)

def failover_sql(credential, subscription_id):
    sql_client = SqlManagementClient(credential, subscription_id)

    # Failover the failover group
    sql_client.failover_groups.begin_failover(
        resource_group_name="rg-secondary",
        server_name="myapp-sql-secondary",
        failover_group_name="myapp-fog"
    ).result()

    logging.info("SQL failover completed")

def update_traffic_manager(credential, subscription_id):
    tm_client = TrafficManagerManagementClient(credential, subscription_id)

    # Disable primary endpoint
    tm_client.endpoints.update(
        resource_group_name="rg-global",
        profile_name="myapp-tm",
        endpoint_type="AzureEndpoints",
        endpoint_name="primary-endpoint",
        parameters={"properties": {"endpointStatus": "Disabled"}}
    )

    logging.info("Traffic Manager updated")

def scale_dr_cluster(credential, subscription_id):
    from azure.mgmt.containerservice import ContainerServiceClient

    aks_client = ContainerServiceClient(credential, subscription_id)

    # Scale up DR cluster
    cluster = aks_client.managed_clusters.get("rg-secondary", "aks-secondary")
    cluster.agent_pool_profiles[0].count = 10

    aks_client.managed_clusters.begin_create_or_update(
        "rg-secondary",
        "aks-secondary",
        cluster
    ).result()

    logging.info("DR AKS cluster scaled up")
```

---

## 8. Zero-Downtime Checklist

### Pre-Disaster Preparation

```
+-- Infrastructure Setup
|   +-- [ ] Multi-region resource groups created
|   +-- [ ] Azure Front Door with health probes configured
|   +-- [ ] SQL Failover Groups enabled
|   +-- [ ] Cosmos DB multi-region writes enabled
|   +-- [ ] Storage RA-GZRS configured
|   +-- [ ] ACR geo-replication enabled
|   +-- [ ] Azure Site Recovery configured for VMs
|   +-- [ ] Azure Backup with cross-region restore
|
+-- Application Readiness
|   +-- [ ] Connection strings use failover group endpoints
|   +-- [ ] Application handles transient failures
|   +-- [ ] Session state externalized (Redis/Cosmos)
|   +-- [ ] Static assets on Azure CDN
|
+-- Monitoring & Alerting
|   +-- [ ] Azure Monitor alerts for both regions
|   +-- [ ] Log Analytics workspace (centralized)
|   +-- [ ] Application Insights (multi-region)
|   +-- [ ] Action groups for notifications
|
+-- Documentation & Runbooks
    +-- [ ] Failover runbook in Azure Automation
    +-- [ ] Failback procedure documented
    +-- [ ] Contact list in Azure DevOps Wiki
    +-- [ ] DR drill scheduled quarterly
```

### Failover Execution

```
AUTOMATED FAILOVER (Target: < 5 minutes)
=========================================

1. Detection (0-60 seconds)
   +-- Front Door health probes detect failure
   +-- Azure Monitor alerts trigger
   +-- Action group sends notifications

2. Traffic Failover (60-120 seconds)
   +-- Front Door routes to secondary origin
   +-- Users automatically redirected

3. Database Failover (60-180 seconds)
   +-- SQL Failover Group auto-promotes secondary
   +-- Cosmos DB: Already active in both regions
   +-- Application reconnects via failover endpoint

4. Compute Scaling (Parallel)
   +-- AKS HPA scales pods in DR region
   +-- Azure Functions: Already globally deployed

5. Verification
   +-- Health endpoints green
   +-- Application functional
   +-- Monitoring dashboards updated
```

---

## 9. DR Testing

### Quarterly DR Drill Script

```bash
#!/bin/bash
# azure-dr-test.sh

set -e
DATE=$(date +%Y-%m-%d)

echo "=== Starting Azure DR Drill ==="

# 1. Notify team
az monitor action-group notify \
  --name dr-team \
  --resource-group rg-global \
  --action-name webhook \
  --webhook-properties "{\"message\": \"DR Drill starting at $(date)\"}"

# 2. Test SQL failover
echo "Testing SQL failover..."
az sql failover-group set-primary \
  --name myapp-fog \
  --server myapp-sql-secondary \
  --resource-group rg-secondary

# 3. Verify secondary is now primary
az sql failover-group show \
  --name myapp-fog \
  --server myapp-sql-secondary \
  --resource-group rg-secondary \
  --query "replicationRole"

# 4. Test application in DR region
echo "Testing application..."
curl -f https://myapp.azurefd.net/health || echo "Health check failed"

# 5. Fail back to primary
echo "Failing back to primary..."
az sql failover-group set-primary \
  --name myapp-fog \
  --server myapp-sql-primary \
  --resource-group rg-primary

# 6. Generate report
echo "=== DR Drill Complete ==="
echo "Report: dr-drill-$DATE.json"
```

---

## 10. Cost Optimization

| Strategy | Active-Active | Warm Standby | Pilot Light |
|----------|---------------|--------------|-------------|
| **Compute** | 100% duplicate | 25-50% capacity | Minimal |
| **Database** | Full replicas | Full replicas | Full replicas |
| **Storage** | RA-GZRS | RA-GZRS | GRS |
| **Monthly Cost** | 2x primary | 1.3-1.5x primary | 1.1-1.2x primary |

### Cost-Saving Tips

```
1. Use Reserved Instances in DR region
   +-- 1-3 year commitments for predictable DR costs
   +-- Azure Hybrid Benefit for Windows/SQL

2. Right-size DR compute
   +-- B-series VMs for standby workloads
   +-- Scale up only during failover

3. Use Standard tier storage for backups
   +-- Cool tier for older backups
   +-- Archive tier for compliance retention

4. Optimize SQL tiers
   +-- Smaller DTU/vCore in DR
   +-- Scale up during failover
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
|  1. Front Door routes to secondary (~60 seconds)               |
|  2. SQL Failover Group promotes secondary (~60 seconds)        |
|  3. Monitor: https://status.azure.com                          |
|                                                                |
|  MANUAL STEPS (if needed):                                     |
|  1. Failover SQL:                                              |
|     az sql failover-group set-primary \                        |
|       --name myapp-fog \                                       |
|       --server myapp-sql-secondary \                           |
|       --resource-group rg-secondary                            |
|                                                                |
|  2. Failover Storage:                                          |
|     az storage account failover \                              |
|       --name myappstorprimary \                                |
|       --resource-group rg-primary                              |
|                                                                |
|  3. Verify application:                                        |
|     curl https://myapp.azurefd.net/health                      |
|                                                                |
|  CONTACTS:                                                     |
|  - On-call: PagerDuty escalation                               |
|  - Azure Support: Premier support case                         |
|  - Status: https://status.azure.com                            |
|                                                                |
+---------------------------------------------------------------+
```

## Gotchas

- SQL Failover Group failover can take 1-2 minutes
- Storage account failover is one-way (requires manual re-protection)
- Front Door origin health probes have 30-second default interval
- ASR failover requires manual commit after testing
- Cosmos DB multi-region writes add latency for strong consistency
- Some Azure services don't support cross-region replication
- DR region must have sufficient quota/capacity
- Some configurations don't replicate (NSG rules, custom policies)

## Related Documentation

- [Azure Site Recovery Documentation](https://docs.microsoft.com/azure/site-recovery/)
- [Azure SQL Failover Groups](https://docs.microsoft.com/azure/azure-sql/database/auto-failover-group-overview)
- [Azure Front Door](https://docs.microsoft.com/azure/frontdoor/)
- [Azure Storage Redundancy](https://docs.microsoft.com/azure/storage/common/storage-redundancy)
- [Azure Backup](https://docs.microsoft.com/azure/backup/)
