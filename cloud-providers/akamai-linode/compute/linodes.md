# Linode Compute Instances (Linodes)

> Virtual machines with dedicated resources running on Akamai's global infrastructure.

## Overview

Linodes are Linux-based virtual machines that form the core of Akamai Cloud Computing (formerly Linode). They offer predictable pricing, SSD storage, and a global network of data centers.

## Key Concepts

| Term | Definition |
|------|------------|
| Linode | Virtual machine instance |
| Plan | CPU, RAM, storage, transfer allocation |
| Image | OS or custom disk image |
| Region | Geographic data center location |
| Backup | Automated point-in-time snapshots |
| StackScript | Deployment automation script |

## Plan Types

| Type | Use Case | Features |
|------|----------|----------|
| **Shared CPU** | General workloads, dev/test | Cost-effective, burstable |
| **Dedicated CPU** | Production, consistent performance | Dedicated cores |
| **High Memory** | Databases, caching | RAM-optimized |
| **GPU** | ML, rendering, scientific computing | NVIDIA Quadro RTX |
| **Premium** | Enterprise workloads | AMD EPYC, NVMe |

### Shared CPU Plans

| Plan | vCPU | RAM | Storage | Transfer | Price/mo |
|------|------|-----|---------|----------|----------|
| Nanode 1GB | 1 | 1 GB | 25 GB | 1 TB | $5 |
| Linode 2GB | 1 | 2 GB | 50 GB | 2 TB | $12 |
| Linode 4GB | 2 | 4 GB | 80 GB | 4 TB | $24 |
| Linode 8GB | 4 | 8 GB | 160 GB | 5 TB | $48 |
| Linode 16GB | 6 | 16 GB | 320 GB | 8 TB | $96 |

### Dedicated CPU Plans

| Plan | vCPU | RAM | Storage | Transfer | Price/mo |
|------|------|-----|---------|----------|----------|
| Dedicated 4GB | 2 | 4 GB | 80 GB | 4 TB | $36 |
| Dedicated 8GB | 4 | 8 GB | 160 GB | 5 TB | $72 |
| Dedicated 16GB | 8 | 16 GB | 320 GB | 6 TB | $144 |
| Dedicated 32GB | 16 | 32 GB | 640 GB | 7 TB | $288 |

## Architecture

```
+---------------------------------------------------------------+
|                     Akamai Cloud Computing                    |
|                                                               |
|  +--------------------+    +--------------------+             |
|  |    Region: Newark  |    | Region: Frankfurt |             |
|  |                    |    |                    |             |
|  |  +------+ +------+ |    |  +------+ +------+ |             |
|  |  |Linode| |Linode| |    |  |Linode| |Linode| |             |
|  |  | web1 | | web2 | |    |  | web3 | | web4 | |             |
|  |  +------+ +------+ |    |  +------+ +------+ |             |
|  |         |          |    |         |          |             |
|  |  +------v------+   |    |  +------v------+   |             |
|  |  |NodeBalancer |   |    |  |NodeBalancer |   |             |
|  |  +-------------+   |    |  +-------------+   |             |
|  +--------------------+    +--------------------+             |
|                                                               |
|  Global Network: 40 Tbps+ capacity                            |
+---------------------------------------------------------------+
```

## Regions

| Region | Location | ID |
|--------|----------|-----|
| Newark | US East | us-east |
| Atlanta | US Southeast | us-southeast |
| Dallas | US Central | us-central |
| Fremont | US West | us-west |
| Toronto | Canada | ca-central |
| London | UK | eu-west |
| Frankfurt | Germany | eu-central |
| Singapore | Asia Pacific | ap-south |
| Tokyo | Japan | ap-northeast |
| Sydney | Australia | ap-southeast |
| Mumbai | India | ap-west |
| São Paulo | Brazil | sa-east |

## Create Linode

### CLI

```bash
# Install Linode CLI
pip install linode-cli

# Configure
linode-cli configure

# Create Linode
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --root_pass "SecurePassword123!" \
  --label my-linode \
  --tags production

# Create with SSH key
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/debian11 \
  --authorized_keys "$(cat ~/.ssh/id_rsa.pub)" \
  --label my-linode

# Create with StackScript
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --stackscript_id 12345 \
  --stackscript_data '{"db_password": "secret"}'
```

### API

```bash
curl -X POST https://api.linode.com/v4/linode/instances \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "g6-standard-2",
    "region": "us-east",
    "image": "linode/ubuntu22.04",
    "root_pass": "SecurePassword123!",
    "label": "my-linode",
    "tags": ["production"]
  }'
```

### Terraform

```hcl
terraform {
  required_providers {
    linode = {
      source = "linode/linode"
    }
  }
}

provider "linode" {
  token = var.linode_token
}

resource "linode_instance" "web" {
  label      = "my-linode"
  type       = "g6-standard-2"
  region     = "us-east"
  image      = "linode/ubuntu22.04"

  authorized_keys = [var.ssh_public_key]

  tags = ["production", "web"]
}
```

## Images

### Official Images

| Distribution | Versions |
|--------------|----------|
| Ubuntu | 20.04, 22.04, 24.04 |
| Debian | 10, 11, 12 |
| CentOS | Stream 8, Stream 9 |
| Rocky Linux | 8, 9 |
| AlmaLinux | 8, 9 |
| Fedora | 38, 39, 40 |
| openSUSE | Leap 15.5 |
| Arch Linux | Latest |
| Alpine | 3.18, 3.19 |
| Gentoo | Latest |
| Slackware | 15.0 |

### Custom Images

```bash
# Create image from disk
linode-cli images create \
  --disk_id 12345 \
  --label "my-custom-image" \
  --description "Ubuntu with app pre-installed"

# Upload custom image
linode-cli images upload \
  --region us-east \
  --label "my-image" \
  --description "Custom image" \
  /path/to/image.raw.gz
```

## StackScripts

Deployment automation scripts.

```bash
#!/bin/bash
# <UDF name="db_password" label="Database Password" />
# <UDF name="app_version" label="App Version" default="latest" />

apt-get update && apt-get upgrade -y
apt-get install -y nginx mysql-server

# Use UDF variables
echo "DB_PASSWORD=$DB_PASSWORD" >> /etc/environment
echo "APP_VERSION=$APP_VERSION" >> /etc/environment

systemctl enable nginx
systemctl start nginx
```

```bash
# Create StackScript
linode-cli stackscripts create \
  --label "My App Setup" \
  --images "linode/ubuntu22.04" \
  --script "$(cat setup.sh)" \
  --description "Installs and configures my app"

# List StackScripts
linode-cli stackscripts list --is_public false
```

## Backups

### Automatic Backups

```
Backup Schedule:
+-- Daily (last 2 days retained)
+-- Weekly (last 2 weeks retained)
+-- Bi-weekly (last 2 bi-weekly retained)
+-- Manual Snapshot (1 slot)
```

```bash
# Enable backups
linode-cli linodes backups-enable 12345678

# List backups
linode-cli linodes backups-list 12345678

# Restore from backup
linode-cli linodes backups-restore 12345678 \
  --backup_id 87654321 \
  --linode_id 12345678 \
  --overwrite true

# Create manual snapshot
linode-cli linodes snapshot 12345678 --label "pre-upgrade"
```

### Backup Pricing

| Plan Size | Backup Price/mo |
|-----------|-----------------|
| 1 GB | $2 |
| 2 GB | $2.50 |
| 4 GB | $5 |
| 8 GB | $10 |
| 16 GB | $20 |

## Resize & Migration

```bash
# Resize Linode (requires reboot)
linode-cli linodes resize 12345678 \
  --type g6-standard-4 \
  --allow_auto_disk_resize true

# Migrate to different region (clone)
linode-cli linodes clone 12345678 \
  --region eu-west \
  --type g6-standard-2 \
  --label "my-linode-eu"

# Migrate within datacenter
linode-cli linodes migrate 12345678
```

## Networking

```
+-----------------------------------------------+
|                   Linode                      |
|                                               |
|  Public IPv4: 192.0.2.1                       |
|  Public IPv6: 2600:3c00::1/128                |
|  Private IPv4: 192.168.1.1 (VLAN)             |
|                                               |
|  +-- eth0: Public network                     |
|  +-- eth1: VLAN (private)                     |
+-----------------------------------------------+
```

```bash
# Add private IP
linode-cli linodes ip-add 12345678 --type ipv4 --public false

# List IPs
linode-cli linodes ips-list 12345678

# Configure VLAN
linode-cli linodes config-update 12345678 54321 \
  --interfaces.purpose vlan \
  --interfaces.label my-vlan \
  --interfaces.ipam_address "10.0.0.1/24"
```

## CLI Quick Reference

```bash
# List Linodes
linode-cli linodes list

# View Linode details
linode-cli linodes view 12345678

# Start/Stop/Reboot
linode-cli linodes boot 12345678
linode-cli linodes shutdown 12345678
linode-cli linodes reboot 12345678

# Delete Linode
linode-cli linodes delete 12345678

# List available plans
linode-cli linodes types

# List available images
linode-cli images list

# List regions
linode-cli regions list

# SSH to Linode
ssh root@192.0.2.1
```

## Pricing

| Component | Cost |
|-----------|------|
| Compute | Per plan (see tables above) |
| Transfer out | $0.005/GB over quota |
| Transfer in | Free |
| IPv4 | Included (1 per Linode) |
| Additional IPv4 | $1/mo |
| IPv6 | Free (/128 per Linode, /64 routed) |
| Backups | ~20% of Linode cost |

## Best Practices

```
1. Security
   +-- Use SSH keys, disable password auth
   +-- Configure Cloud Firewall
   +-- Enable automatic security updates
   +-- Use private network for inter-Linode traffic

2. Performance
   +-- Choose correct plan type for workload
   +-- Use Block Storage for large datasets
   +-- Utilize Object Storage for static assets
   +-- Consider NodeBalancer for load distribution

3. Reliability
   +-- Enable automatic backups
   +-- Distribute across multiple regions
   +-- Use NodeBalancer health checks
   +-- Monitor with Longview agent
```

## Gotchas

- Transfer quota is pooled across all Linodes in account
- Resizing requires reboot (plan migrations)
- Private IP requires VLAN configuration
- Backups don't include Block Storage volumes
- GPU instances limited to specific regions
- StackScripts only run on initial deployment
- IPv6 is /128 per Linode, /64 available on request
- Disk resize only works when increasing size

## Limits

| Resource | Limit |
|----------|-------|
| Linodes per account | 100+ (soft limit) |
| Block Storage per region | 10 TB |
| Images | 25 per account |
| StackScripts | Unlimited |
| Tags per resource | 100 |
| SSH keys | 100 |
| API requests | 1,600/min |
