# Google Cloud VPC (Virtual Private Cloud)

> Global, scalable, and flexible networking for your cloud resources.

## Overview

Google Cloud VPC provides networking for your cloud-based resources and services. VPC networks are global, spanning all regions, while subnets are regional.

## Key Concepts

| Term | Definition |
|------|------------|
| VPC Network | Global virtual network |
| Subnet | Regional IP address range |
| Route | Path for network traffic |
| Firewall Rule | Allow/deny traffic rules |
| Private Google Access | Access Google APIs without public IP |
| VPC Peering | Connect VPC networks |
| Shared VPC | Share network across projects |

## Architecture

```
+---------------------------------------------------------------+
|                    VPC Network (Global)                       |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                     us-central1                         |  |
|  |  +---------------------------------------------------+  |  |
|  |  |  Subnet: 10.0.1.0/24                              |  |  |
|  |  |  +--------+  +--------+  +--------+               |  |  |
|  |  |  |  VM-1  |  |  VM-2  |  |  VM-3  |               |  |  |
|  |  |  +--------+  +--------+  +--------+               |  |  |
|  |  +---------------------------------------------------+  |  |
|  +---------------------------------------------------------+  |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                    europe-west1                         |  |
|  |  +---------------------------------------------------+  |  |
|  |  |  Subnet: 10.0.2.0/24                              |  |  |
|  |  |  +--------+  +--------+                           |  |  |
|  |  |  |  VM-4  |  |  VM-5  |                           |  |  |
|  |  |  +--------+  +--------+                           |  |  |
|  |  +---------------------------------------------------+  |  |
|  +---------------------------------------------------------+  |
|                                                               |
|  VMs can communicate across regions using internal IPs       |
+---------------------------------------------------------------+
```

## VPC Network Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Auto mode** | Automatic subnets in each region | Quick setup, testing |
| **Custom mode** | You define subnets | Production, control |

```bash
# Create custom mode VPC
gcloud compute networks create my-vpc \
  --subnet-mode=custom

# Create auto mode VPC
gcloud compute networks create my-vpc \
  --subnet-mode=auto
```

## Subnets

### Create Subnet

```bash
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access

# With secondary ranges (for GKE)
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --secondary-range=pods=10.1.0.0/16,services=10.2.0.0/20
```

### Reserved IP Addresses

```
Subnet: 10.0.1.0/24
+-- 10.0.1.0   - Network address
+-- 10.0.1.1   - Default gateway
+-- 10.0.1.2   - Reserved
+-- 10.0.1.3   - Reserved
+-- ...        - Usable (10.0.1.4 - 10.0.1.254)
+-- 10.0.1.255 - Broadcast
```

## Firewall Rules

### Rule Components

| Component | Description |
|-----------|-------------|
| Direction | Ingress or egress |
| Priority | 0-65535 (lower = higher priority) |
| Action | Allow or deny |
| Target | Which VMs (all, tags, service account) |
| Source/Destination | IP ranges, tags, service accounts |
| Protocol/Ports | TCP, UDP, ICMP, etc. |

### Default Rules

| Rule | Direction | Priority | Action |
|------|-----------|----------|--------|
| default-allow-internal | Ingress | 65534 | Allow |
| default-allow-ssh | Ingress | 65534 | Allow (port 22) |
| default-allow-rdp | Ingress | 65534 | Allow (port 3389) |
| default-allow-icmp | Ingress | 65534 | Allow ICMP |
| implied-deny-ingress | Ingress | 65535 | Deny |
| implied-allow-egress | Egress | 65535 | Allow |

### Create Firewall Rules

```bash
# Allow SSH from specific IP
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --direction=ingress \
  --action=allow \
  --rules=tcp:22 \
  --source-ranges=1.2.3.4/32 \
  --target-tags=ssh-enabled \
  --priority=1000

# Allow internal traffic
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --direction=ingress \
  --action=allow \
  --rules=all \
  --source-ranges=10.0.0.0/8

# Allow HTTP/HTTPS with service account
gcloud compute firewall-rules create allow-web \
  --network=my-vpc \
  --direction=ingress \
  --action=allow \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-service-accounts=web-server@my-project.iam.gserviceaccount.com

# Deny all egress
gcloud compute firewall-rules create deny-all-egress \
  --network=my-vpc \
  --direction=egress \
  --action=deny \
  --rules=all \
  --destination-ranges=0.0.0.0/0 \
  --priority=65534
```

## Routes

### Route Types

| Type | Description |
|------|-------------|
| System routes | Auto-created for subnets |
| Custom static | User-defined routes |
| Dynamic routes | Learned via Cloud Router (BGP) |

### Create Routes

```bash
# Route to NVA (Network Virtual Appliance)
gcloud compute routes create route-to-nva \
  --network=my-vpc \
  --destination-range=0.0.0.0/0 \
  --next-hop-instance=nva-instance \
  --next-hop-instance-zone=us-central1-a \
  --priority=100

# Route to VPN tunnel
gcloud compute routes create route-to-onprem \
  --network=my-vpc \
  --destination-range=192.168.0.0/16 \
  --next-hop-vpn-tunnel=my-tunnel \
  --next-hop-vpn-tunnel-region=us-central1
```

## Private Google Access

Access Google APIs without external IP.

```bash
# Enable on subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access
```

### Private Service Connect

```bash
# Create Private Service Connect endpoint
gcloud compute forwarding-rules create psc-endpoint \
  --network=my-vpc \
  --region=us-central1 \
  --address=psc-address \
  --target-google-apis-bundle=all-apis
```

## VPC Peering

```
VPC-A (Project A)                  VPC-B (Project B)
+--------------------+             +--------------------+
|  10.0.0.0/16       |<----------->|  10.1.0.0/16       |
|                    |  Peering    |                    |
|  +--------+        |             |       +--------+   |
|  |  VM-1  |--------------------------------|  VM-2  | |
|  +--------+        |             |       +--------+   |
+--------------------+             +--------------------+
```

```bash
# Create peering from VPC-A to VPC-B
gcloud compute networks peerings create peering-a-to-b \
  --network=vpc-a \
  --peer-network=vpc-b \
  --peer-project=project-b

# Create peering from VPC-B to VPC-A
gcloud compute networks peerings create peering-b-to-a \
  --network=vpc-b \
  --peer-network=vpc-a \
  --peer-project=project-a
```

### Peering Properties

- Non-transitive (A<->B, B<->C doesn't mean A<->C)
- No overlapping IP ranges
- Both sides must create peering
- Internal IPs used for communication

## Shared VPC

```
Host Project (owns network)
+-------------------------------------------------------------+
|  VPC Network                                                 |
|  +-- Subnet A: 10.0.1.0/24                                   |
|  +-- Subnet B: 10.0.2.0/24                                   |
|  +-- Firewall Rules                                          |
+-------------------------------------------------------------+
        |                     |                     |
        v                     v                     v
+---------------+   +---------------+   +---------------+
| Service Proj A |   | Service Proj B |   | Service Proj C |
| (Uses Subnet A)|   | (Uses Subnet B)|   | (Uses both)    |
+---------------+   +---------------+   +---------------+
```

```bash
# Enable Shared VPC on host project
gcloud compute shared-vpc enable host-project

# Associate service project
gcloud compute shared-vpc associated-projects add service-project \
  --host-project=host-project
```

## Cloud NAT

Outbound internet access for VMs without external IPs.

```bash
# Create Cloud Router
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

# Create Cloud NAT
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

## VPN and Interconnect

### Cloud VPN

| Type | Description | Bandwidth |
|------|-------------|-----------|
| Classic VPN | Single tunnel | 3 Gbps |
| HA VPN | Multiple tunnels, 99.99% SLA | 3 Gbps × tunnels |

```bash
# Create HA VPN gateway
gcloud compute vpn-gateways create my-vpn-gateway \
  --network=my-vpc \
  --region=us-central1

# Create external VPN gateway (peer)
gcloud compute external-vpn-gateways create peer-gateway \
  --interfaces 0=1.2.3.4

# Create VPN tunnels
gcloud compute vpn-tunnels create tunnel-0 \
  --vpn-gateway=my-vpn-gateway \
  --peer-external-gateway=peer-gateway \
  --region=us-central1 \
  --ike-version=2 \
  --shared-secret=secret \
  --router=my-router \
  --interface=0 \
  --peer-external-gateway-interface=0
```

### Cloud Interconnect

| Type | Bandwidth | Latency |
|------|-----------|---------|
| Dedicated | 10-200 Gbps | Lowest |
| Partner | 50 Mbps - 50 Gbps | Low |

## CLI Quick Reference

```bash
# Create VPC
gcloud compute networks create my-vpc --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24

# List networks
gcloud compute networks list

# List subnets
gcloud compute networks subnets list --network=my-vpc

# Create firewall rule
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --allow=tcp:22 \
  --source-ranges=0.0.0.0/0

# List firewall rules
gcloud compute firewall-rules list --filter="network:my-vpc"

# Delete network
gcloud compute networks delete my-vpc
```

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **VPC is global**: Subnets are regional
2. **Auto vs Custom mode**: Custom recommended for production
3. **Firewall rules**: Stateful, apply to instances
4. **Network tags**: Target firewall rules to VMs
5. **VPC Peering**: Non-transitive, no overlapping IPs
6. **Shared VPC**: Centralize network management
7. **Private Google Access**: Access APIs without external IP
8. **Cloud NAT**: Outbound internet without external IP
9. **Routes**: Determine traffic path
10. **Implied rules**: Deny ingress, allow egress by default

## Gotchas

- VPC networks are global; subnets are regional
- Cannot change subnet mode after creation
- Firewall rules apply to all instances in network (use tags)
- VPC peering is non-transitive
- Cannot peer with overlapping IP ranges
- Shared VPC requires org-level permissions
- Auto-mode creates subnets you might not need
- Deleting VPC requires deleting all resources first
- Routes affect all instances unless tagged
- Cloud NAT is regional

## Limits

| Resource | Limit |
|----------|-------|
| Networks per project | 15 (can increase) |
| Subnets per network | 10,000 |
| Firewall rules per project | 500 |
| Routes per network | 250 |
| VPC peerings per network | 25 |
| Secondary ranges per subnet | 30 |
| Network tags per instance | 64 |
| IP aliases per instance | 100 |
| Static routes per network | 200 |
