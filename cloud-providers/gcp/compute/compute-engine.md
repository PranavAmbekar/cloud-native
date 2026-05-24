# Google Compute Engine (GCE)

> Scalable, high-performance virtual machines running in Google's data centers.

## Overview

Google Compute Engine delivers virtual machines running in Google's innovative data centers and worldwide fiber network. Compute Engine offers scale, performance, and value that lets you launch large compute clusters on Google's infrastructure.

## Key Concepts

| Term | Definition |
|------|------------|
| Instance | A virtual machine hosted on Google infrastructure |
| Machine Type | Predefined or custom CPU/memory configuration |
| Image | Boot disk template (OS + software) |
| Persistent Disk | Durable block storage for instances |
| Instance Template | Reusable VM configuration |
| Instance Group | Collection of VMs (managed or unmanaged) |
| Preemptible/Spot VM | Short-lived, low-cost instances |
| Sole-tenant Node | Dedicated physical server |

## Architecture

```
+---------------------------------------------------------------+
|                         GCP Project                           |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                         Region                          |  |
|  |  +---------------------------------------------------+  |  |
|  |  |                       Zone A                      |  |  |
|  |  |  +----------+  +----------+  +----------+         |  |  |
|  |  |  | Instance |  | Instance |  | Instance |         |  |  |
|  |  |  |   VM-1   |  |   VM-2   |  |   VM-3   |         |  |  |
|  |  |  +----------+  +----------+  +----------+         |  |  |
|  |  +---------------------------------------------------+  |  |
|  |  +---------------------------------------------------+  |  |
|  |  |                       Zone B                      |  |  |
|  |  |  +---------------------------------------------+  |  |  |
|  |  |  |       Managed Instance Group (MIG)          |  |  |  |
|  |  |  |  +----+  +----+  +----+  +----+             |  |  |  |
|  |  |  |  | VM |  | VM |  | VM |  | VM |             |  |  |  |
|  |  |  |  +----+  +----+  +----+  +----+             |  |  |  |
|  |  |  +---------------------------------------------+  |  |  |
|  |  +---------------------------------------------------+  |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

## Machine Types

### Machine Type Families

| Family | Use Case | Examples |
|--------|----------|----------|
| **General Purpose (E2, N2, N2D, N1)** | Balanced workloads | Web servers, dev/test |
| **Compute Optimized (C2, C2D, H3)** | Compute-intensive | Gaming, HPC, batch |
| **Memory Optimized (M1, M2, M3)** | Large in-memory databases | SAP HANA, in-memory analytics |
| **Accelerator Optimized (A2, A3, G2)** | ML training, graphics | AI/ML, rendering |
| **Storage Optimized (Z3)** | High disk throughput | Databases, analytics |

### Machine Type Naming

```
n2-standard-8
|   |        |
|   |        +-- vCPUs (8 vCPUs)
|   +----------- Type (standard, highmem, highcpu)
+--------------- Series (n2, e2, c2, etc.)

Examples:
- e2-micro: 0.25-2 vCPUs, 1 GB (free tier eligible)
- n2-standard-4: 4 vCPUs, 16 GB
- n2-highmem-8: 8 vCPUs, 64 GB
- c2-standard-60: 60 vCPUs, 240 GB
```

### Custom Machine Types

```bash
# Create custom machine type
gcloud compute instances create my-vm \
  --custom-cpu=6 \
  --custom-memory=24GB \
  --zone=us-central1-a
```

## Pricing Models

| Model | Discount | Use Case |
|-------|----------|----------|
| **On-demand** | 0% | Unpredictable, short-term |
| **Sustained Use** | Up to 30% (auto) | Running 25%+ of month |
| **Committed Use (1yr)** | Up to 37% | Predictable workloads |
| **Committed Use (3yr)** | Up to 55% | Long-term workloads |
| **Spot VMs** | Up to 91% | Fault-tolerant, batch jobs |
| **Preemptible VMs** | Up to 80% | Legacy, max 24 hours |

### Spot VMs

```bash
# Create spot VM
gcloud compute instances create spot-vm \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --zone=us-central1-a
```

- Can be reclaimed anytime (30-second warning)
- No maximum runtime (unlike preemptible 24hr limit)
- Same price as preemptible
- Use for: batch, CI/CD, fault-tolerant workloads

## Persistent Disks

| Type | IOPS | Throughput | Use Case |
|------|------|------------|----------|
| **pd-standard** | 0.75/GB | 0.12 MB/s/GB | Cold data, backups |
| **pd-balanced** | 6/GB | 0.28 MB/s/GB | General purpose |
| **pd-ssd** | 30/GB | 0.48 MB/s/GB | Production workloads |
| **pd-extreme** | Up to 120K | Up to 2.4 GB/s | High-performance |
| **Hyperdisk Extreme** | Up to 350K | Up to 5 GB/s | Highest IOPS |
| **Local SSD** | 900K+ | 9.4 GB/s | Ephemeral, ultra-high IOPS |

### Disk Features

```
Persistent Disk Features:
+-- Snapshots: Point-in-time backups
+-- Images: Create from disk for reuse
+-- Resize: Increase size without downtime
+-- Regional: Replicate across 2 zones
+-- Encryption: Default (Google-managed) or CMEK

Local SSD:
+-- 375 GB per device (up to 24 devices)
+-- Data lost on stop/delete
+-- Cannot be detached/attached
```

## Instance Groups

### Managed Instance Group (MIG)

```
+-------------------------------------------------------------+
|                  Managed Instance Group                      |
|                                                              |
|  Instance Template                                           |
|  +-- Machine type: n2-standard-4                             |
|  +-- Boot disk: debian-11                                    |
|  +-- Startup script: install_app.sh                          |
|  +-- Network tags: web-server                                |
|                                                              |
|  +--------+  +--------+  +--------+  +--------+              |
|  | VM-001 |  | VM-002 |  | VM-003 |  | VM-004 |              |
|  +--------+  +--------+  +--------+  +--------+              |
|                                                              |
|  Autoscaling: min=2, max=10, target CPU=60%                  |
|  Health Check: HTTP /health every 10s                        |
|  Auto-healing: Replace unhealthy instances                   |
+-------------------------------------------------------------+
```

### MIG Types

| Type | Scope | Use Case |
|------|-------|----------|
| **Zonal MIG** | Single zone | Simple deployments |
| **Regional MIG** | Multiple zones in region | High availability |

### Autoscaling Policies

| Policy | Description |
|--------|-------------|
| CPU utilization | Scale based on average CPU |
| Load balancing capacity | Scale based on LB metrics |
| Cloud Monitoring metric | Custom metrics |
| Schedule | Time-based scaling |

## Images

### Image Types

| Type | Description |
|------|-------------|
| **Public Images** | Google-provided (Debian, Ubuntu, Windows, etc.) |
| **Custom Images** | Created from disks or imported |
| **Image Families** | Pointer to latest image version |
| **Machine Images** | Full VM backup (disk + metadata + config) |

```bash
# List public images
gcloud compute images list --filter="family:debian"

# Create custom image from disk
gcloud compute images create my-image \
  --source-disk=my-disk \
  --source-disk-zone=us-central1-a

# Create image family
gcloud compute images create my-image-v2 \
  --source-disk=my-disk \
  --family=my-app-images
```

## Startup & Shutdown Scripts

```bash
# Startup script (runs at boot)
gcloud compute instances create my-vm \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx'

# Or from file
gcloud compute instances create my-vm \
  --metadata-from-file=startup-script=startup.sh

# Shutdown script
gcloud compute instances create my-vm \
  --metadata=shutdown-script='#!/bin/bash
    echo "Shutting down" >> /var/log/shutdown.log'
```

## Metadata

```bash
# Instance metadata (from within VM)
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/

# Common metadata endpoints
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/name
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/zone
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip

# Service account token
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
```

## Live Migration

```
+-----------------+         +-----------------+
|   Source Host   |  -----> |   Target Host   |
|   (maintenance) |  live   |   (running)     |
|                 | migrate |                 |
+-----------------+         +-----------------+

Maintenance Policy Options:
+-- MIGRATE: Live migration (default)
+-- TERMINATE: Stop during maintenance
```

## Shielded VMs

Security features for verified boot:

| Feature | Description |
|---------|-------------|
| Secure Boot | Verify boot loader signature |
| vTPM | Virtual Trusted Platform Module |
| Integrity Monitoring | Verify boot integrity |

```bash
gcloud compute instances create my-vm \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring
```

## CLI Quick Reference

```bash
# Create instance
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-4 \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-ssd

# List instances
gcloud compute instances list

# Start/Stop/Delete
gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances delete my-vm --zone=us-central1-a

# SSH into instance
gcloud compute ssh my-vm --zone=us-central1-a

# Create instance template
gcloud compute instance-templates create my-template \
  --machine-type=n2-standard-4 \
  --image-family=debian-11 \
  --image-project=debian-cloud

# Create managed instance group
gcloud compute instance-groups managed create my-mig \
  --template=my-template \
  --size=3 \
  --zone=us-central1-a

# Set autoscaling
gcloud compute instance-groups managed set-autoscaling my-mig \
  --zone=us-central1-a \
  --min-num-replicas=2 \
  --max-num-replicas=10 \
  --target-cpu-utilization=0.6

# Create snapshot
gcloud compute disks snapshot my-disk \
  --zone=us-central1-a \
  --snapshot-names=my-snapshot

# Resize instance
gcloud compute instances set-machine-type my-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-8
```

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **E2**: Most cost-effective, good for small/medium workloads
2. **N2**: Better performance than E2, sustained use discounts
3. **Spot vs Preemptible**: Spot has no 24-hour limit, same pricing
4. **Sustained Use**: Automatic, no commitment required
5. **Live Migration**: Default behavior during maintenance
6. **Regional MIG**: Distributes across zones in region
7. **Sole-tenant**: Required for BYOL, compliance
8. **Local SSD**: Data lost on stop/delete (not just terminate)
9. **pd-balanced**: Good balance of cost and performance
10. **Shielded VM**: Security features, slight overhead

## Gotchas

- Instance names must be unique within a project
- Stopped instances still incur disk charges
- Local SSD data is lost on stop (not just delete)
- Cannot change machine type while running
- Regional PD requires instances in same region
- Spot VMs can be preempted anytime
- Committed use discounts are per-region
- Custom machine types must follow vCPU-to-memory ratios
- Live migration may briefly pause the VM
- Serial console access requires enabling in project

## Limits

| Resource | Default Limit |
|----------|---------------|
| Instances per zone | Varies by quota |
| CPUs per project | 24 (varies by region) |
| Persistent disk per instance | 128 disks |
| Local SSDs per instance | 24 devices |
| Snapshots per project | 5,000 |
| Images per project | 2,000 |
| Instance groups per zone | 2,000 |
| Instances per MIG | 2,000 |
