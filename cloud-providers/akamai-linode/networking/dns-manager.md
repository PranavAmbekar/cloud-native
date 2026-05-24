# Linode DNS Manager

> Free authoritative DNS hosting with global distribution.

## Overview

Linode DNS Manager provides free, authoritative DNS hosting with global anycast distribution. Manage domains and DNS records through the API, CLI, or Cloud Manager.

## Key Concepts

| Term | Definition |
|------|------------|
| Domain | DNS zone for your domain |
| Record | DNS entry (A, AAAA, MX, etc.) |
| SOA | Start of Authority record |
| TTL | Time to Live (caching duration) |
| Primary | Master DNS zone |
| Secondary | Slave zone (zone transfer) |

## Supported Record Types

| Type | Description |
|------|-------------|
| **A** | IPv4 address |
| **AAAA** | IPv6 address |
| **CNAME** | Canonical name (alias) |
| **MX** | Mail exchange |
| **TXT** | Text record (SPF, DKIM, etc.) |
| **NS** | Name server |
| **SRV** | Service location |
| **CAA** | Certificate Authority Authorization |
| **PTR** | Reverse DNS |

## Architecture

```
+---------------------------------------------------------------+
|                    Linode DNS Infrastructure                  |
|                                                               |
|  Global Anycast Network                                       |
|  +----+ +----+ +----+ +----+ +----+ +----+                   |
|  | US | | EU | | UK | | SG | | JP | | AU |  Name Servers      |
|  +----+ +----+ +----+ +----+ +----+ +----+                   |
|                         |                                     |
|            ns1-5.linode.com (anycast)                        |
|                         |                                     |
|  +------------------------------------------------------+    |
|  |                Your Domain Zones                      |    |
|  |  +----------------+ +----------------+                |    |
|  |  | example.com    | | example.org    |                |    |
|  |  | A, MX, TXT...  | | A, MX, TXT...  |                |    |
|  |  +----------------+ +----------------+                |    |
|  +------------------------------------------------------+    |
+---------------------------------------------------------------+
```

## Pricing

| Component | Cost |
|-----------|------|
| DNS hosting | **Free** |
| Domains | Unlimited |
| Records | Unlimited |
| Queries | Unlimited |

## Name Servers

Point your domain to these name servers:

```
ns1.linode.com
ns2.linode.com
ns3.linode.com
ns4.linode.com
ns5.linode.com
```

## Create Domain

### CLI

```bash
# Create domain zone
linode-cli domains create \
  --domain example.com \
  --type master \
  --soa_email admin@example.com

# Create with records
linode-cli domains create \
  --domain example.com \
  --type master \
  --soa_email admin@example.com \
  --tags production

# List domains
linode-cli domains list

# View domain
linode-cli domains view 12345
```

### API

```bash
curl -X POST https://api.linode.com/v4/domains \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "domain": "example.com",
    "type": "master",
    "soa_email": "admin@example.com"
  }'
```

### Terraform

```hcl
resource "linode_domain" "main" {
  domain    = "example.com"
  type      = "master"
  soa_email = "admin@example.com"
}
```

## Manage Records

### A Record

```bash
# Create A record
linode-cli domains records-create 12345 \
  --type A \
  --name www \
  --target 192.0.2.1 \
  --ttl_sec 300

# Root domain
linode-cli domains records-create 12345 \
  --type A \
  --name "" \
  --target 192.0.2.1

# Wildcard
linode-cli domains records-create 12345 \
  --type A \
  --name "*" \
  --target 192.0.2.1
```

### AAAA Record (IPv6)

```bash
linode-cli domains records-create 12345 \
  --type AAAA \
  --name www \
  --target 2001:db8::1 \
  --ttl_sec 300
```

### CNAME Record

```bash
linode-cli domains records-create 12345 \
  --type CNAME \
  --name blog \
  --target www.example.com \
  --ttl_sec 300
```

### MX Record

```bash
# Google Workspace
linode-cli domains records-create 12345 \
  --type MX \
  --name "" \
  --target aspmx.l.google.com \
  --priority 1

linode-cli domains records-create 12345 \
  --type MX \
  --name "" \
  --target alt1.aspmx.l.google.com \
  --priority 5
```

### TXT Record

```bash
# SPF record
linode-cli domains records-create 12345 \
  --type TXT \
  --name "" \
  --target "v=spf1 include:_spf.google.com ~all"

# DKIM record
linode-cli domains records-create 12345 \
  --type TXT \
  --name "google._domainkey" \
  --target "v=DKIM1; k=rsa; p=MIIBIjAN..."

# Domain verification
linode-cli domains records-create 12345 \
  --type TXT \
  --name "" \
  --target "google-site-verification=abc123"
```

### SRV Record

```bash
# SIP service
linode-cli domains records-create 12345 \
  --type SRV \
  --name "_sip._tcp" \
  --target sip.example.com \
  --port 5060 \
  --priority 10 \
  --weight 100
```

### CAA Record

```bash
# Allow Let's Encrypt
linode-cli domains records-create 12345 \
  --type CAA \
  --name "" \
  --target "letsencrypt.org" \
  --tag issue
```

## Terraform Records

```hcl
resource "linode_domain_record" "www" {
  domain_id   = linode_domain.main.id
  name        = "www"
  record_type = "A"
  target      = linode_instance.web.ip_address
  ttl_sec     = 300
}

resource "linode_domain_record" "mx" {
  domain_id   = linode_domain.main.id
  name        = ""
  record_type = "MX"
  target      = "mail.example.com"
  priority    = 10
}

resource "linode_domain_record" "spf" {
  domain_id   = linode_domain.main.id
  name        = ""
  record_type = "TXT"
  target      = "v=spf1 a mx ~all"
}
```

## Import Zone

```bash
# Import zone file
linode-cli domains import \
  --domain example.com \
  --remote_nameserver ns1.oldprovider.com
```

## Zone Transfer (Secondary)

```bash
# Create secondary zone
linode-cli domains create \
  --domain example.com \
  --type slave \
  --master_ips '["192.0.2.10"]'

# Linode will AXFR from master
```

## Update Records

```bash
# List records
linode-cli domains records-list 12345

# Update record
linode-cli domains records-update 12345 67890 \
  --target 192.0.2.2 \
  --ttl_sec 600

# Delete record
linode-cli domains records-delete 12345 67890
```

## Common Configurations

### Basic Website

```bash
# Domain zone
linode-cli domains create --domain example.com --type master --soa_email admin@example.com

# Root A record
linode-cli domains records-create $DOMAIN_ID --type A --name "" --target $LINODE_IP

# WWW CNAME
linode-cli domains records-create $DOMAIN_ID --type CNAME --name www --target example.com
```

### Email (Google Workspace)

```bash
# MX records
linode-cli domains records-create $DOMAIN_ID --type MX --name "" --target aspmx.l.google.com --priority 1
linode-cli domains records-create $DOMAIN_ID --type MX --name "" --target alt1.aspmx.l.google.com --priority 5
linode-cli domains records-create $DOMAIN_ID --type MX --name "" --target alt2.aspmx.l.google.com --priority 5

# SPF
linode-cli domains records-create $DOMAIN_ID --type TXT --name "" --target "v=spf1 include:_spf.google.com ~all"

# DMARC
linode-cli domains records-create $DOMAIN_ID --type TXT --name "_dmarc" --target "v=DMARC1; p=quarantine; rua=mailto:admin@example.com"
```

### Load Balanced Servers

```bash
# Multiple A records (round-robin)
linode-cli domains records-create $DOMAIN_ID --type A --name "" --target 192.0.2.1
linode-cli domains records-create $DOMAIN_ID --type A --name "" --target 192.0.2.2
linode-cli domains records-create $DOMAIN_ID --type A --name "" --target 192.0.2.3
```

## CLI Quick Reference

```bash
# Domains
linode-cli domains list
linode-cli domains create --domain example.com --type master --soa_email admin@example.com
linode-cli domains view 12345
linode-cli domains update 12345 --soa_email new@example.com
linode-cli domains delete 12345

# Records
linode-cli domains records-list 12345
linode-cli domains records-create 12345 --type A --name www --target 192.0.2.1
linode-cli domains records-view 12345 67890
linode-cli domains records-update 12345 67890 --target 192.0.2.2
linode-cli domains records-delete 12345 67890

# Import
linode-cli domains import --domain example.com --remote_nameserver ns1.old.com
```

## Best Practices

```
1. TTL Strategy
   +-- Production: 300-3600 seconds
   +-- Before changes: Lower to 300
   +-- After stable: Increase to 3600+
   +-- Avoid very low TTL (< 60)

2. Email Configuration
   +-- Always set up SPF
   +-- Configure DKIM
   +-- Add DMARC policy
   +-- Test with mail-tester.com

3. Security
   +-- Use CAA records
   +-- Implement DNSSEC (external)
   +-- Monitor for unauthorized changes

4. Redundancy
   +-- All 5 name servers are anycast
   +-- Consider secondary DNS provider
   +-- Keep zone file backup
```

## Gotchas

- Zone propagation can take up to 15 minutes
- TTL changes don't affect cached records immediately
- CNAME at root requires ALIAS workaround (not supported)
- Wildcard doesn't match root domain
- SOA refresh/retry controlled by Linode
- No DNSSEC support (manage externally)
- Import requires TCP/53 access to source NS
- Deleting domain deletes all records

## Limits

| Resource | Limit |
|----------|-------|
| Domains per account | 500 |
| Records per domain | 12,000 |
| TXT record length | 512 bytes (chain for longer) |
| TTL minimum | 30 seconds |
| TTL maximum | 604800 seconds (7 days) |
| API rate | 1,600 requests/minute |
