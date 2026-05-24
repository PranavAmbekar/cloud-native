# Azure Virtual Machines

> Scalable on-demand computing resources with full control over the operating system and software.

## Overview

Azure Virtual Machines (VMs) provide Infrastructure-as-a-Service (IaaS) computing. You create VMs with your choice of OS, size, and configuration. Pay per minute for what you use.

## Key Concepts

| Term | Definition |
|------|------------|
| Virtual Machine | Compute instance running in Azure |
| Image | Template containing OS + software (marketplace or custom) |
| VM Size | Hardware configuration (CPU, memory, storage, GPU) |
| Managed Disk | Persistent block storage for VMs |
| Network Security Group (NSG) | Virtual firewall (stateful) |
| Availability Set | Logical grouping across fault/update domains |
| Availability Zone | Physically separate datacenters within a region |
| Virtual Machine Scale Set (VMSS) | Auto-scaling group of identical VMs |

## VM Size Families

| Family | Use Case | Examples |
|--------|----------|----------|
| **General Purpose (B, D, Dv5)** | Balanced CPU/memory | Web servers, dev/test |
| **Compute Optimized (F, Fx)** | High CPU-to-memory ratio | Batch processing, gaming |
| **Memory Optimized (E, M)** | High memory-to-CPU ratio | Databases, in-memory caching |
| **Storage Optimized (L)** | High disk throughput/IOPS | Big data, SQL/NoSQL |
| **GPU (NC, ND, NV)** | GPU-accelerated | ML training, rendering, VDI |
| **High Performance (HB, HC)** | Fastest CPU, RDMA | HPC, simulations |

### VM Size Naming

```
Standard_D4s_v5
|        | | |
|        | | +- Version (higher = newer)
|        | +--- "s" = Premium SSD capable
|        +----- Size (2, 4, 8, 16, etc. vCPUs)
+-------------- Family (D = general purpose)
```

## Pricing Models

| Model | Discount | Use Case |
|-------|----------|----------|
| **Pay-as-you-go** | 0% | Unpredictable workloads |
| **Reserved (1yr)** | ~40% | Steady-state workloads |
| **Reserved (3yr)** | ~60% | Long-term committed workloads |
| **Savings Plan** | Up to 65% | Flexible commitment |
| **Spot VMs** | Up to 90% | Fault-tolerant, interruptible workloads |
| **Azure Hybrid Benefit** | Up to 85% | Existing Windows Server/SQL licenses |
| **Dedicated Host** | Varies | Compliance, licensing |

### Spot VMs Key Points

- Can be evicted with 30-second notice
- Use for: batch jobs, dev/test, CI/CD, containerized workloads
- Set max price or use market rate
- Eviction policies: Stop-Deallocate or Delete

## Availability Options

| Option | SLA | Scope |
|--------|-----|-------|
| **Single VM (Premium SSD)** | 99.9% | Single instance |
| **Availability Set** | 99.95% | Fault domains within datacenter |
| **Availability Zones** | 99.99% | Multiple datacenters in region |
| **Virtual Machine Scale Sets** | Varies | Auto-scaling across zones |

### Availability Set

```
+---------------------------------------------------------+
|                    Availability Set                     |
|  +-----------------+    +-----------------+             |
|  | Fault Domain 0  |    | Fault Domain 1  |    FD 2    |
|  |   +-----+       |    |   +-----+       |  +-----+   |
|  |   | VM1 |       |    |   | VM2 |       |  | VM3 |   |
|  |   +-----+       |    |   +-----+       |  +-----+   |
|  | (Rack/Power)    |    | (Rack/Power)    |            |
|  +-----------------+    +-----------------+            |
|                                                         |
|  Update Domains: 0, 1, 2, 3, 4 (max 20)                |
|  (Only one UD updated at a time)                        |
+---------------------------------------------------------+
```

## Disk Types

| Type | IOPS | Throughput | Use Case |
|------|------|------------|----------|
| **Ultra Disk** | Up to 160,000 | 4,000 MB/s | Mission-critical, SAP HANA |
| **Premium SSD v2** | Up to 80,000 | 1,200 MB/s | Production workloads |
| **Premium SSD** | Up to 20,000 | 900 MB/s | Production workloads |
| **Standard SSD** | Up to 6,000 | 750 MB/s | Web servers, dev/test |
| **Standard HDD** | Up to 2,000 | 500 MB/s | Backup, infrequent access |

### Disk Types

- **OS Disk**: Contains the operating system
- **Data Disk**: Additional storage (number depends on VM size)
- **Temporary Disk**: Ephemeral, data lost on stop/deallocate
- **Ephemeral OS Disk**: OS disk on local VM storage (faster, no cost)

## Networking

- **Virtual Network (VNet)**: Isolated network for VMs
- **Network Interface (NIC)**: Virtual network adapter
- **Public IP**: Optional public-facing IP address
- **NSG**: Firewall rules (allow/deny, inbound/outbound)
- **Application Security Group (ASG)**: Group VMs for NSG rules
- **Accelerated Networking**: Hardware offload for lower latency

## Virtual Machine Scale Sets (VMSS)

```
+-----------------------------------------------------+
|            Virtual Machine Scale Set                 |
|   +-----+  +-----+  +-----+  +-----+  +-----+     |
|   | VM0 |  | VM1 |  | VM2 |  | VM3 |  | VM4 |     |
|   +-----+  +-----+  +-----+  +-----+  +-----+     |
|      Min: 2      Current: 5       Max: 100         |
+-----------------------------------------------------+
        |
        v
  Scaling Rules
  - CPU > 70% -> Scale out
  - CPU < 30% -> Scale in
  - Schedule: 9am scale to 10
```

### Scaling Modes

- **Uniform**: Identical VMs from same image
- **Flexible**: Mix VM sizes/images in same scale set

## Extensions

Common VM extensions:

| Extension | Purpose |
|-----------|---------|
| Custom Script Extension | Run scripts at deployment |
| Azure Monitor Agent | Monitoring and logs |
| Desired State Configuration | PowerShell DSC |
| Azure Disk Encryption | BitLocker/DM-Crypt |
| Dependency Agent | Application dependency mapping |

## Cloud-Init / Custom Data

```yaml
#cloud-config
package_upgrade: true
packages:
  - nginx
  - docker.io

runcmd:
  - systemctl start nginx
  - systemctl enable nginx
```

## Instance Metadata Service (IMDS)

```bash
# Get instance metadata (from within VM)
curl -H "Metadata:true" "http://169.254.169.254/metadata/instance?api-version=2021-02-01"

# Get access token for managed identity
curl -H "Metadata:true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
```

## CLI Quick Reference

```bash
# Create VM
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --admin-username azureuser \
  --generate-ssh-keys

# List VMs
az vm list --output table

# Start/Stop/Deallocate
az vm start --resource-group myRG --name myVM
az vm stop --resource-group myRG --name myVM
az vm deallocate --resource-group myRG --name myVM

# Resize VM
az vm resize --resource-group myRG --name myVM --size Standard_D4s_v5

# Delete VM
az vm delete --resource-group myRG --name myVM --yes

# Create scale set
az vmss create \
  --resource-group myRG \
  --name myVMSS \
  --image Ubuntu2204 \
  --instance-count 2 \
  --vm-sku Standard_D2s_v5
```

## Exam Tips (AZ-104, AZ-305)

1. **Stop vs Deallocate**: Stop keeps hardware allocated (still billed), Deallocate releases hardware
2. **Availability Sets vs Zones**: Sets = same datacenter, Zones = different datacenters
3. **Fault Domain**: Max 3 per availability set
4. **Update Domain**: Max 20, one updated at a time during maintenance
5. **Spot VMs**: Check eviction history, set max price
6. **Ephemeral OS Disk**: Faster boot, no storage cost, lost on stop
7. **Proximity Placement Group**: Keep VMs physically close for low latency
8. **Azure Hybrid Benefit**: Bring your own Windows Server/SQL licenses
9. **Reserved Instances**: Can exchange for different size in same family
10. **VMSS Flexible**: Allows mixing VM sizes (newer, preferred)

## Gotchas

- Temporary disk data is lost on stop/deallocate (not just delete)
- Premium SSD requires "s" in VM size name (e.g., Standard_D2**s**_v5)
- NSGs are stateful; allow rules work both directions for established connections
- Public IP is dynamic by default (changes on deallocate); use Static for persistent
- Spot VMs cannot be resized
- Availability Sets cannot be changed after VM creation
- VM names cannot be changed after creation

## Limits

| Resource | Default Limit |
|----------|---------------|
| VMs per subscription | 25,000 per region |
| VM cores per subscription | 20-350 (varies by size, can increase) |
| Availability sets per subscription | 2,500 per region |
| VMs per availability set | 200 |
| Data disks per VM | Varies by size (up to 64) |
| NICs per VM | Varies by size |
