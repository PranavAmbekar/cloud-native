# Amazon EC2 (Elastic Compute Cloud)

> Virtual servers in the cloud with complete control over computing resources.

## Overview

EC2 provides resizable compute capacity. You launch virtual servers (instances), choose CPU, memory, storage, and networking, and pay only for what you use.

## Key Concepts

| Term | Definition |
|------|------------|
| Instance | A virtual server |
| AMI (Amazon Machine Image) | Template containing OS + software to launch instances |
| Instance Type | Hardware configuration (CPU, memory, storage, network) |
| EBS (Elastic Block Store) | Persistent block storage volumes |
| Security Group | Virtual firewall for instances (stateful) |
| Key Pair | SSH credentials for Linux / RDP for Windows |
| Elastic IP | Static public IPv4 address |
| Placement Group | Logical grouping for performance or fault tolerance |

## Instance Types

| Family | Use Case | Examples |
|--------|----------|----------|
| **General Purpose (M, T)** | Balanced compute/memory | Web servers, dev environments |
| **Compute Optimized (C)** | High-performance processors | Batch processing, gaming, ML inference |
| **Memory Optimized (R, X)** | Large in-memory datasets | Databases, in-memory caching |
| **Storage Optimized (I, D)** | High sequential read/write | Data warehousing, distributed file systems |
| **Accelerated (P, G, Inf)** | GPU/custom hardware | ML training, graphics rendering |

### Instance Type Naming

```
m5.2xlarge
| |  |
| |  +- Size (nano, micro, small, medium, large, xlarge, 2xlarge...)
| +---- Generation (higher = newer)
+------ Family (m = general purpose)
```

## Pricing Models

| Model | Discount | Use Case |
|-------|----------|----------|
| **On-Demand** | 0% | Unpredictable workloads, short-term |
| **Reserved (1yr)** | ~40% | Steady-state, predictable usage |
| **Reserved (3yr)** | ~60% | Long-term committed workloads |
| **Savings Plans** | Up to 72% | Flexible commitment across instance families |
| **Spot Instances** | Up to 90% | Fault-tolerant, flexible workloads |
| **Dedicated Hosts** | Varies | Compliance, licensing requirements |

### Spot Instance Key Points

- Can be interrupted with 2-minute warning
- Use for: batch jobs, CI/CD, big data, containerized workloads
- Spot Fleet: maintain target capacity across instance types/AZs
- Spot Block: 1-6 hour uninterrupted (deprecated in some regions)

## Placement Groups

| Type | Behavior | Use Case |
|------|----------|----------|
| **Cluster** | Instances close together in single AZ | Low latency, high throughput (HPC) |
| **Spread** | Instances on distinct hardware | Critical instances, max 7 per AZ |
| **Partition** | Groups of instances on separate racks | Large distributed systems (Hadoop, Kafka) |

## Storage Options

| Type | Persistence | Performance | Use Case |
|------|-------------|-------------|----------|
| **EBS** | Persistent | Varies by type | Boot volumes, databases |
| **Instance Store** | Ephemeral | Very high IOPS | Caching, temporary data |
| **EFS** | Persistent | Shared | Shared file systems |

## Network Features

- **ENI (Elastic Network Interface)**: Virtual network card
- **ENA (Enhanced Networking)**: Up to 100 Gbps, lower latency
- **EFA (Elastic Fabric Adapter)**: HPC, ML workloads

## Auto Scaling

```
+-----------------------------------------+
|           Auto Scaling Group            |
|  +-----+  +-----+  +-----+  +-----+   |
|  | EC2 |  | EC2 |  | EC2 |  | EC2 |   |
|  +-----+  +-----+  +-----+  +-----+   |
|     Min: 2    Desired: 4    Max: 10    |
+-----------------------------------------+
         |
         v
   Scaling Policies
   - Target tracking (keep CPU at 50%)
   - Step scaling (add 2 if CPU > 80%)
   - Scheduled (scale up at 9am)
```

## Security

### Security Groups (Stateful)
- Default: deny all inbound, allow all outbound
- Rules: allow only (no deny rules)
- Reference other security groups

### Network ACLs (Stateless)
- Subnet level
- Rules: allow AND deny
- Numbered rules, evaluated in order

## User Data & Metadata

```bash
# User Data - runs at first boot
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd

# Instance Metadata (from within instance)
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/public-ipv4

# IMDSv2 (more secure, requires token)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/
```

## CLI Quick Reference

```bash
# Launch instance
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678

# List instances
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]'

# Start/Stop/Terminate
aws ec2 start-instances --instance-ids i-12345678
aws ec2 stop-instances --instance-ids i-12345678
aws ec2 terminate-instances --instance-ids i-12345678

# Create AMI
aws ec2 create-image --instance-id i-12345678 --name "My AMI"
```

## Exam Tips (Solutions Architect)

1. **Spot vs Reserved**: Spot for fault-tolerant, Reserved for steady-state
2. **Placement Groups**: Cluster = performance, Spread = availability, Partition = big data
3. **Hibernation**: Saves RAM to EBS, faster restart (EBS root volume must be encrypted)
4. **Instance Store**: Data lost on stop/terminate (not on reboot)
5. **Dedicated Hosts vs Dedicated Instances**: Hosts = visibility into sockets/cores (licensing)
6. **Capacity Reservations**: Reserve capacity in specific AZ without commitment
7. **IMDSv2**: More secure, requires token-based access

## Gotchas

- Instance store data is lost on stop (not just terminate)
- Security groups are stateful; NACLs are stateless
- T2/T3 have CPU credits; can burst but may throttle
- Elastic IPs incur charges when NOT associated
- AMIs are region-specific (must copy to use in other regions)

## Limits

| Resource | Default Limit |
|----------|---------------|
| Instances per region | 20 (On-Demand), varies by type |
| Elastic IPs per region | 5 |
| Security groups per VPC | 500 |
| Rules per security group | 60 inbound + 60 outbound |
