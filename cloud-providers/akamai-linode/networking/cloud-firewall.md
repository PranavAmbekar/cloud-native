# Linode Cloud Firewall

> Network-level firewall for controlling inbound and outbound traffic to Linodes.

## Overview

Cloud Firewall provides a free, stateful firewall that filters traffic before it reaches your Linodes. Create reusable firewall rules and apply them to multiple Linodes for consistent security.

## Key Concepts

| Term | Definition |
|------|------------|
| Firewall | Named ruleset for traffic control |
| Inbound Rules | Traffic coming to your Linode |
| Outbound Rules | Traffic leaving your Linode |
| Device | Linode attached to firewall |
| Label | Identifier for rules and firewalls |

## Architecture

```
                          Internet
                              |
                              v
+---------------------------------------------------------------+
|                      Cloud Firewall                           |
|                                                               |
|  +-------------------------+  +---------------------------+   |
|  |     Inbound Rules       |  |     Outbound Rules        |   |
|  |                         |  |                           |   |
|  | + Allow TCP 22 (SSH)    |  | + Allow all outbound      |   |
|  | + Allow TCP 80 (HTTP)   |  | + (or restrict as needed) |   |
|  | + Allow TCP 443 (HTTPS) |  |                           |   |
|  | - Drop all other        |  |                           |   |
|  +-------------------------+  +---------------------------+   |
|                                                               |
+---------------------------+-----------------------------------+
                            |
            +---------------+---------------+
            |               |               |
        +-------+       +-------+       +-------+
        |Linode1|       |Linode2|       |Linode3|
        +-------+       +-------+       +-------+
```

## Pricing

| Component | Cost |
|-----------|------|
| Cloud Firewall | **Free** |
| Rules | Free |
| Firewalls per account | Unlimited |

## Create Firewall

### CLI

```bash
# Create firewall with rules
linode-cli firewalls create \
  --label web-firewall \
  --rules.inbound_policy DROP \
  --rules.outbound_policy ACCEPT \
  --rules.inbound '[
    {"label": "allow-ssh", "action": "ACCEPT", "protocol": "TCP", "ports": "22", "addresses": {"ipv4": ["0.0.0.0/0"]}},
    {"label": "allow-http", "action": "ACCEPT", "protocol": "TCP", "ports": "80", "addresses": {"ipv4": ["0.0.0.0/0"], "ipv6": ["::/0"]}},
    {"label": "allow-https", "action": "ACCEPT", "protocol": "TCP", "ports": "443", "addresses": {"ipv4": ["0.0.0.0/0"], "ipv6": ["::/0"]}}
  ]'

# Apply to Linode
linode-cli firewalls device-create 12345 \
  --id 98765 \
  --type linode
```

### API

```bash
curl -X POST https://api.linode.com/v4/networking/firewalls \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "web-firewall",
    "rules": {
      "inbound_policy": "DROP",
      "outbound_policy": "ACCEPT",
      "inbound": [
        {
          "label": "allow-ssh",
          "action": "ACCEPT",
          "protocol": "TCP",
          "ports": "22",
          "addresses": {"ipv4": ["0.0.0.0/0"]}
        }
      ]
    },
    "devices": {
      "linodes": [12345]
    }
  }'
```

### Terraform

```hcl
resource "linode_firewall" "web" {
  label = "web-firewall"

  inbound_policy  = "DROP"
  outbound_policy = "ACCEPT"

  inbound {
    label    = "allow-ssh"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "22"
    ipv4     = ["192.0.2.0/24"]  # Restrict to your IP
  }

  inbound {
    label    = "allow-http"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "80"
    ipv4     = ["0.0.0.0/0"]
    ipv6     = ["::/0"]
  }

  inbound {
    label    = "allow-https"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "443"
    ipv4     = ["0.0.0.0/0"]
    ipv6     = ["::/0"]
  }

  linodes = [linode_instance.web.id]
}
```

## Rule Structure

### Inbound Rule

```json
{
  "label": "allow-web",
  "action": "ACCEPT",
  "protocol": "TCP",
  "ports": "80, 443",
  "addresses": {
    "ipv4": ["0.0.0.0/0"],
    "ipv6": ["::/0"]
  }
}
```

### Rule Fields

| Field | Values | Description |
|-------|--------|-------------|
| label | String | Rule identifier |
| action | ACCEPT, DROP | Allow or deny traffic |
| protocol | TCP, UDP, ICMP, IPENCAP | Network protocol |
| ports | "22", "80-443", "22, 80, 443" | Port or range |
| addresses.ipv4 | CIDR list | Source/dest IPv4 |
| addresses.ipv6 | CIDR list | Source/dest IPv6 |

## Common Rule Sets

### Web Server

```bash
linode-cli firewalls create \
  --label web-firewall \
  --rules.inbound_policy DROP \
  --rules.outbound_policy ACCEPT \
  --rules.inbound '[
    {"label": "ssh", "action": "ACCEPT", "protocol": "TCP", "ports": "22", "addresses": {"ipv4": ["YOUR_IP/32"]}},
    {"label": "http", "action": "ACCEPT", "protocol": "TCP", "ports": "80", "addresses": {"ipv4": ["0.0.0.0/0"], "ipv6": ["::/0"]}},
    {"label": "https", "action": "ACCEPT", "protocol": "TCP", "ports": "443", "addresses": {"ipv4": ["0.0.0.0/0"], "ipv6": ["::/0"]}}
  ]'
```

### Database Server

```bash
linode-cli firewalls create \
  --label db-firewall \
  --rules.inbound_policy DROP \
  --rules.outbound_policy ACCEPT \
  --rules.inbound '[
    {"label": "ssh", "action": "ACCEPT", "protocol": "TCP", "ports": "22", "addresses": {"ipv4": ["YOUR_IP/32"]}},
    {"label": "mysql", "action": "ACCEPT", "protocol": "TCP", "ports": "3306", "addresses": {"ipv4": ["192.168.0.0/16"]}},
    {"label": "postgres", "action": "ACCEPT", "protocol": "TCP", "ports": "5432", "addresses": {"ipv4": ["192.168.0.0/16"]}}
  ]'
```

### Kubernetes Node

```bash
linode-cli firewalls create \
  --label k8s-node-firewall \
  --rules.inbound_policy DROP \
  --rules.outbound_policy ACCEPT \
  --rules.inbound '[
    {"label": "ssh", "action": "ACCEPT", "protocol": "TCP", "ports": "22", "addresses": {"ipv4": ["YOUR_IP/32"]}},
    {"label": "kubelet", "action": "ACCEPT", "protocol": "TCP", "ports": "10250", "addresses": {"ipv4": ["192.168.0.0/16"]}},
    {"label": "nodeport", "action": "ACCEPT", "protocol": "TCP", "ports": "30000-32767", "addresses": {"ipv4": ["0.0.0.0/0"]}}
  ]'
```

## Update Rules

```bash
# Update firewall rules
linode-cli firewalls rules-update 12345 \
  --inbound '[
    {"label": "ssh", "action": "ACCEPT", "protocol": "TCP", "ports": "22", "addresses": {"ipv4": ["NEW_IP/32"]}},
    {"label": "http", "action": "ACCEPT", "protocol": "TCP", "ports": "80", "addresses": {"ipv4": ["0.0.0.0/0"]}}
  ]' \
  --inbound_policy DROP \
  --outbound_policy ACCEPT

# View current rules
linode-cli firewalls rules-list 12345
```

## Attach/Detach Devices

```bash
# Attach Linode to firewall
linode-cli firewalls device-create 12345 \
  --id 98765 \
  --type linode

# List attached devices
linode-cli firewalls devices-list 12345

# Detach device
linode-cli firewalls device-delete 12345 54321
```

## Enable/Disable Firewall

```bash
# Disable firewall (allows all traffic)
linode-cli firewalls update 12345 --status disabled

# Enable firewall
linode-cli firewalls update 12345 --status enabled
```

## Policies

| Policy | Applied When |
|--------|--------------|
| inbound_policy | No rules match inbound traffic |
| outbound_policy | No rules match outbound traffic |

| Value | Behavior |
|-------|----------|
| ACCEPT | Allow traffic |
| DROP | Silently drop traffic |

## CLI Quick Reference

```bash
# Firewalls
linode-cli firewalls list
linode-cli firewalls create --label my-fw --rules.inbound_policy DROP
linode-cli firewalls view 12345
linode-cli firewalls update 12345 --label new-name
linode-cli firewalls delete 12345

# Rules
linode-cli firewalls rules-list 12345
linode-cli firewalls rules-update 12345 --inbound '[...]'

# Devices
linode-cli firewalls devices-list 12345
linode-cli firewalls device-create 12345 --id 98765 --type linode
linode-cli firewalls device-delete 12345 54321
```

## Best Practices

```
1. Default Deny
   +-- Set inbound_policy to DROP
   +-- Only allow necessary ports
   +-- Restrict SSH to specific IPs

2. Principle of Least Privilege
   +-- Use specific CIDR ranges
   +-- Avoid 0.0.0.0/0 when possible
   +-- Document each rule's purpose

3. Organized Rules
   +-- Use descriptive labels
   +-- Group related rules
   +-- Create separate firewalls by function

4. Testing
   +-- Test rules before applying to production
   +-- Verify SSH access before applying
   +-- Have console access as backup
```

## Firewall vs iptables

```
Cloud Firewall:
+-- Applied at network level (before reaching Linode)
+-- Managed via API/CLI
+-- Shared across multiple Linodes
+-- No CPU overhead on Linode

iptables/nftables:
+-- Applied at OS level
+-- More granular control
+-- Per-application rules
+-- Uses Linode CPU
```

**Recommendation**: Use Cloud Firewall for perimeter security, iptables for application-specific rules.

## Gotchas

- Firewall must be enabled after creation
- Rules apply to both IPv4 and IPv6 (specify both in addresses)
- Maximum 25 rules per firewall
- Changes apply immediately (no staging)
- Detaching firewall allows all traffic
- Cannot block DHCP/metadata (192.168.255.0/24)
- ICMP rules don't use ports field
- Rules evaluated in order, first match wins

## Limits

| Resource | Limit |
|----------|-------|
| Firewalls per account | 25 |
| Rules per firewall | 25 inbound + 25 outbound |
| Devices per firewall | 25 |
| Labels | 32 characters |
| Address entries per rule | 255 |
