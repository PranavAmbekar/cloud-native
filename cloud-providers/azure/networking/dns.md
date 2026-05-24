# Azure DNS

> Host your DNS domains in Azure for high availability and performance.

## Overview

Azure DNS provides DNS domain hosting using Microsoft Azure infrastructure. It supports both public DNS zones (internet-facing) and private DNS zones (VNet-only resolution).

## Key Concepts

| Term | Definition |
|------|------------|
| DNS Zone | Container for DNS records of a domain |
| Record Set | Collection of records with same name and type |
| Alias Record | Points to Azure resource instead of IP |
| Private DNS Zone | DNS zone for VNet name resolution |
| Virtual Network Link | Connects private zone to VNet |

## DNS Zone Types

| Type | Use Case | Resolution |
|------|----------|------------|
| **Public DNS Zone** | Internet-facing domains | Global, public |
| **Private DNS Zone** | Internal VNet resources | VNet only |

## Public DNS Zones

### Supported Record Types

| Type | Description | Example |
|------|-------------|---------|
| A | IPv4 address | www -> 1.2.3.4 |
| AAAA | IPv6 address | www -> 2001:db8::1 |
| CNAME | Canonical name (alias) | www -> myapp.azurewebsites.net |
| MX | Mail exchange | @ -> mail.example.com |
| TXT | Text record | @ -> "v=spf1 include:_spf.google.com" |
| NS | Name server | @ -> ns1-01.azure-dns.com |
| SOA | Start of authority | Auto-managed |
| SRV | Service location | _sip._tcp -> sipserver:5060 |
| CAA | Certificate authority | @ -> "0 issue letsencrypt.org" |
| PTR | Reverse lookup | 4.3.2.1 -> server.example.com |

### Record Set

```
Zone: contoso.com
|-- @ (apex)
|   |-- A: 1.2.3.4, 1.2.3.5 (multiple values)
|   |-- MX: mail.contoso.com (priority 10)
|   +-- TXT: "v=spf1 include:_spf.microsoft.com"
|-- www
|   +-- CNAME: contoso.azurewebsites.net
|-- api
|   +-- A: 5.6.7.8
+-- mail
    +-- A: 9.10.11.12
```

### Alias Records

Point to Azure resources instead of static IPs.

| Resource Type | Use Case |
|---------------|----------|
| Public IP | Dynamic IP updates |
| Traffic Manager | DNS load balancing |
| Front Door | Global load balancing |
| CDN endpoint | Content delivery |
| Another record set | Same zone reference |

```
Standard A Record:
  www.contoso.com -> 1.2.3.4 (static)

Alias Record:
  www.contoso.com -> [Azure Public IP resource]
  (Auto-updates when IP changes)
```

### Apex Domain (Zone Apex)

CNAME not allowed at apex. Use Alias records instead.

```
X Cannot do:
  contoso.com CNAME -> myapp.azurewebsites.net

OK Can do:
  contoso.com ALIAS -> myapp.azurewebsites.net
  (Uses Azure Alias record feature)
```

## Private DNS Zones

### Architecture

```
+-------------------------------------------------------------+
|           Private DNS Zone: contoso.internal                |
|                                                             |
|  Records:                                                   |
|  |-- vm1 -> 10.0.1.4                                        |
|  |-- vm2 -> 10.0.1.5                                        |
|  |-- sql -> 10.0.2.10                                       |
|  +-- api -> 10.0.3.20                                       |
|                                                             |
+---------------------------+---------------------------------+
                            |
               Virtual Network Links
                            |
          +-----------------+-----------------+
          |                 |                 |
    +-----v-----+     +-----v-----+     +-----v-----+
    |  VNet-Prod |     |  VNet-Dev |     | VNet-Test |
    |  (Auto-reg)|     |           |     |           |
    +-----------+     +-----------+     +-----------+
```

### Auto-Registration

Automatically create DNS records for VMs.

```
VNet with Auto-Registration enabled:
|-- VM created (vm1, IP: 10.0.1.4)
|   +-- DNS record auto-created: vm1.contoso.internal -> 10.0.1.4
|-- VM deleted
|   +-- DNS record auto-deleted
+-- VM IP changed
    +-- DNS record auto-updated
```

### Private DNS for Azure Services

Common private DNS zones for Azure Private Endpoints:

| Service | Private DNS Zone |
|---------|------------------|
| Blob Storage | privatelink.blob.core.windows.net |
| Azure SQL | privatelink.database.windows.net |
| Cosmos DB | privatelink.documents.azure.com |
| Key Vault | privatelink.vaultcore.azure.net |
| App Service | privatelink.azurewebsites.net |
| ACR | privatelink.azurecr.io |

```
Private Endpoint DNS Resolution:

mystorageaccount.blob.core.windows.net
    |
    +-- CNAME: mystorageaccount.privatelink.blob.core.windows.net
                    |
                    +-- A: 10.0.1.5 (Private Endpoint IP)
```

## DNS Resolution Flow

### Public DNS

```
User (Browser)
    |
    v
Recursive Resolver (ISP/8.8.8.8)
    |
    |-- Query: www.contoso.com?
    |
    v
Root NS -> .com NS -> Azure DNS
    |
    +-- Response: 1.2.3.4
```

### Private DNS (VNet)

```
VM in VNet
    |
    v
Azure DNS (168.63.129.16)
    |
    |-- Query: vm1.contoso.internal?
    |
    v
Private DNS Zone (linked to VNet)
    |
    +-- Response: 10.0.1.4
```

## DNS Forwarding

### Custom DNS Server

```
+-----------------------------------------------------+
|                        VNet                         |
|                                                     |
|  +-----------------------------------------------+  |
|  |           Custom DNS Server                   |  |
|  |                10.0.0.4                       |  |
|  |                                               |  |
|  |  Conditional Forwarders:                      |  |
|  |  |-- *.internal -> Private DNS Zone          |  |
|  |  +-- * -> Azure DNS (168.63.129.16)          |  |
|  +-----------------------------------------------+  |
|                                                     |
|  VNet DNS Settings: 10.0.0.4                        |
+-----------------------------------------------------+
```

### Azure DNS Private Resolver

Managed DNS forwarding service.

```
+-------------------------------------------------------------+
|                    Azure DNS Private Resolver               |
|                                                             |
|  +---------------------+    +---------------------+         |
|  |   Inbound Endpoint  |    |  Outbound Endpoint  |         |
|  |   (VNet queries)    |    | (Forward to on-prem)|         |
|  |     10.0.1.4        |    |     10.0.2.4        |         |
|  +---------------------+    +---------------------+         |
|                                                             |
|  Forwarding Rules:                                          |
|  |-- onprem.local -> 192.168.1.1 (on-prem DNS)              |
|  +-- azure.com -> Azure DNS                                 |
+-------------------------------------------------------------+
```

## Delegation

### Delegate Subdomain

```
Parent Zone: contoso.com (external registrar or Azure)
    |
    +-- NS record: dev.contoso.com -> Azure DNS name servers

Child Zone: dev.contoso.com (Azure DNS)
    |-- www -> 1.2.3.4
    +-- api -> 5.6.7.8
```

### Delegate to Azure DNS

Update at your domain registrar:

```
Domain: contoso.com
Name Servers:
  ns1-01.azure-dns.com
  ns2-01.azure-dns.net
  ns3-01.azure-dns.org
  ns4-01.azure-dns.info
```

## CLI Quick Reference

```bash
# Create public DNS zone
az network dns zone create \
  --name contoso.com \
  --resource-group myRG

# Create A record
az network dns record-set a add-record \
  --zone-name contoso.com \
  --resource-group myRG \
  --record-set-name www \
  --ipv4-address 1.2.3.4

# Create CNAME record
az network dns record-set cname set-record \
  --zone-name contoso.com \
  --resource-group myRG \
  --record-set-name api \
  --cname myapi.azurewebsites.net

# Create alias record
az network dns record-set a update \
  --zone-name contoso.com \
  --resource-group myRG \
  --name www \
  --target-resource /subscriptions/.../publicIPAddresses/myPublicIP

# Create private DNS zone
az network private-dns zone create \
  --name contoso.internal \
  --resource-group myRG

# Link private zone to VNet
az network private-dns link vnet create \
  --name myLink \
  --zone-name contoso.internal \
  --resource-group myRG \
  --virtual-network myVNet \
  --registration-enabled true

# Create private DNS record
az network private-dns record-set a add-record \
  --zone-name contoso.internal \
  --resource-group myRG \
  --record-set-name vm1 \
  --ipv4-address 10.0.1.4

# List record sets
az network dns record-set list \
  --zone-name contoso.com \
  --resource-group myRG \
  --output table
```

## Exam Tips (AZ-104, AZ-305)

1. **Alias records**: Use for apex domain to Azure resources
2. **CNAME at apex**: Not allowed, use Alias record instead
3. **Auto-registration**: Only one VNet per private zone can have it
4. **Private DNS zones**: Can link to multiple VNets
5. **168.63.129.16**: Azure's recursive DNS resolver
6. **TTL**: Minimum 1 second, default 3600 seconds
7. **Private Endpoint DNS**: Requires specific zone names
8. **NS delegation**: Point to Azure DNS name servers at registrar
9. **Record sets**: Multiple values for same name/type
10. **Private Resolver**: Managed DNS forwarding service

## Gotchas

- CNAME cannot coexist with other record types for same name
- CNAME not allowed at zone apex (use Alias)
- Auto-registration limited to one VNet per zone
- Private DNS zones don't resolve from on-premises by default
- Azure DNS doesn't support DNSSEC
- TTL changes take time to propagate (up to previous TTL)
- Private DNS zone names must be valid DNS names
- Alias records only work with specific Azure resources
- Private Endpoint DNS requires zone linking to VNet
- MX and NS records cannot be Alias records

## Limits

| Resource | Limit |
|----------|-------|
| DNS zones per subscription | 250 |
| Record sets per zone | 10,000 |
| Records per record set | 20 |
| Private DNS zones per subscription | 1000 |
| Virtual network links per private zone | 1000 |
| Auto-registration VNets per zone | 1 |
| Private zones per VNet link | 1000 |
| DNS queries per second | 500/zone |
| Alias record target updates | Near instant |
