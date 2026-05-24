# Linode VLANs

> Private, isolated networking between Linodes in the same data center.

## Overview

VLANs (Virtual LANs) provide free, private Layer 2 networking between Linodes in the same region. Traffic on VLANs never touches the public internet and is not metered against transfer quotas.

## Key Concepts

| Term | Definition |
|------|------------|
| VLAN | Private Layer 2 network segment |
| Label | VLAN identifier (per region) |
| IPAM Address | IP assigned to Linode on VLAN |
| Interface | Network connection (eth0, eth1) |
| Purpose | Interface type (public, vlan, vpc) |

## Architecture

```
+---------------------------------------------------------------+
|                      Region: us-east                          |
|                                                               |
|  +-----------------------------------------------------------+|
|  |                    VLAN: backend                          ||
|  |                  (Private Network)                        ||
|  |                                                           ||
|  |  +--------+     +--------+     +--------+                ||
|  |  | web-1  |     | web-2  |     | web-3  |                ||
|  |  |10.0.0.1|-----|10.0.0.2|-----|10.0.0.3|                ||
|  |  +--------+     +--------+     +--------+                ||
|  |                      |                                    ||
|  |              +-------v-------+                            ||
|  |              |    db-1       |                            ||
|  |              |   10.0.0.10   |                            ||
|  |              +---------------+                            ||
|  +-----------------------------------------------------------+|
|                                                               |
|  Public Internet access via eth0 (separate interface)         |
+---------------------------------------------------------------+
```

## Pricing

| Component | Cost |
|-----------|------|
| VLAN | **Free** |
| Private traffic | **Free** (not metered) |
| IP addresses | Self-assigned (IPAM) |

## Create VLAN

### During Linode Creation

```bash
# Create Linode with VLAN
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --root_pass "SecurePassword!" \
  --label web-1 \
  --interfaces '[
    {"purpose": "public"},
    {"purpose": "vlan", "label": "backend", "ipam_address": "10.0.0.1/24"}
  ]'
```

### Add VLAN to Existing Linode

```bash
# Get current config ID
linode-cli linodes configs-list 12345

# Update config with VLAN interface
linode-cli linodes config-update 12345 54321 \
  --interfaces '[
    {"purpose": "public"},
    {"purpose": "vlan", "label": "backend", "ipam_address": "10.0.0.2/24"}
  ]'

# Reboot to apply changes
linode-cli linodes reboot 12345
```

### Terraform

```hcl
resource "linode_instance" "web" {
  label      = "web-1"
  type       = "g6-standard-2"
  region     = "us-east"
  image      = "linode/ubuntu22.04"

  interface {
    purpose = "public"
  }

  interface {
    purpose      = "vlan"
    label        = "backend"
    ipam_address = "10.0.0.1/24"
  }
}

resource "linode_instance" "db" {
  label      = "db-1"
  type       = "g6-standard-4"
  region     = "us-east"
  image      = "linode/ubuntu22.04"

  interface {
    purpose = "public"
  }

  interface {
    purpose      = "vlan"
    label        = "backend"
    ipam_address = "10.0.0.10/24"
  }
}
```

## Configure Networking

### Automatic (Network Helper)

With Network Helper enabled (default), Linode automatically configures the VLAN interface.

### Manual Configuration

If Network Helper is disabled:

```bash
# Check interface name
ip link show

# Configure interface manually
cat > /etc/netplan/02-vlan.yaml << EOF
network:
  version: 2
  ethernets:
    eth1:
      addresses:
        - 10.0.0.1/24
      routes:
        - to: 10.0.0.0/24
          via: 0.0.0.0
          scope: link
EOF

netplan apply
```

### Verify Connectivity

```bash
# Check interface
ip addr show eth1

# Ping another Linode on same VLAN
ping 10.0.0.2

# Test throughput
iperf3 -s  # Server
iperf3 -c 10.0.0.2  # Client
```

## VLAN-Only Linodes

Create Linodes without public internet access:

```bash
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --root_pass "SecurePassword!" \
  --label internal-db \
  --interfaces '[
    {"purpose": "vlan", "label": "backend", "ipam_address": "10.0.0.100/24"}
  ]'
```

Access via another Linode on the VLAN:

```bash
# From public-facing Linode
ssh root@10.0.0.100
```

## Multiple VLANs

```bash
# Create Linode on multiple VLANs
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --root_pass "SecurePassword!" \
  --label router \
  --interfaces '[
    {"purpose": "public"},
    {"purpose": "vlan", "label": "frontend", "ipam_address": "10.0.1.1/24"},
    {"purpose": "vlan", "label": "backend", "ipam_address": "10.0.2.1/24"}
  ]'
```

## Use Cases

### Database Isolation

```
Internet --> Web Server (public + vlan) --> Database (vlan only)
                                               |
                                               +-- 10.0.0.0/24 only
```

### Multi-Tier Application

```
Internet
    |
    v
+---+---+   +-------+   +-------+
| Web-1 |   | Web-2 |   | Web-3 |
+---+---+   +---+---+   +---+---+
    |           |           |
    +-----+-----+-----+-----+
          |     VLAN: app     |
          |   10.0.1.0/24     |
    +-----+-----+-----+-----+
    |           |           |
+---+---+   +---+---+   +---+---+
| App-1 |   | App-2 |   | App-3 |
+---+---+   +---+---+   +---+---+
    |           |           |
    +-----+-----+-----+-----+
          |    VLAN: db      |
          |   10.0.2.0/24    |
    +-----+-----+-----+-----+
          |           |
      +---+---+   +---+---+
      | DB-1  |   | DB-2  |
      +-------+   +-------+
```

### Kubernetes Private Network

```bash
# Create nodes with VLAN for pod network
for i in 1 2 3; do
  linode-cli linodes create \
    --type g6-standard-4 \
    --region us-east \
    --image linode/ubuntu22.04 \
    --root_pass "SecurePassword!" \
    --label k8s-node-$i \
    --interfaces '[
      {"purpose": "public"},
      {"purpose": "vlan", "label": "k8s-internal", "ipam_address": "10.10.0.'$i'/24"}
    ]'
done
```

## CLI Quick Reference

```bash
# Create Linode with VLAN
linode-cli linodes create \
  --type g6-standard-2 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --interfaces '[{"purpose": "public"}, {"purpose": "vlan", "label": "my-vlan", "ipam_address": "10.0.0.1/24"}]'

# List VLANs in region
linode-cli vlans list --region us-east

# View Linode interfaces
linode-cli linodes configs-list 12345
linode-cli linodes config-view 12345 54321

# Update interfaces (requires reboot)
linode-cli linodes config-update 12345 54321 \
  --interfaces '[...]'

# Reboot to apply changes
linode-cli linodes reboot 12345
```

## Best Practices

```
1. IP Address Management
   +-- Use consistent CIDR ranges
   +-- Document IP assignments
   +-- Reserve ranges for different purposes
   +-- Consider using 10.x.x.x for VLANs

2. Security
   +-- Use VLANs for sensitive traffic
   +-- Database servers: VLAN only (no public)
   +-- Use Cloud Firewall for public interfaces
   +-- Enable host-based firewall (iptables)

3. Performance
   +-- VLAN traffic is low latency
   +-- Full Linode bandwidth available
   +-- No transfer metering on VLAN

4. Naming
   +-- Use descriptive VLAN labels
   +-- Include purpose: db, app, mgmt
   +-- Keep consistent across environments
```

## Gotchas

- VLANs are region-specific (cannot span regions)
- VLAN label creates/joins the VLAN (no explicit create)
- Interface order matters (eth0, eth1, eth2)
- Changing interfaces requires reboot
- IP addresses are self-managed (no DHCP)
- Maximum 3 interfaces per Linode
- VLAN-only Linodes need NAT gateway for internet
- Network Helper may override manual configs
- Same VLAN label in different regions = different VLANs

## Limits

| Resource | Limit |
|----------|-------|
| VLANs per region | No limit |
| Linodes per VLAN | No limit |
| Interfaces per Linode | 3 |
| VLAN label length | 64 characters |
| IP range | /24 recommended (up to /8) |
