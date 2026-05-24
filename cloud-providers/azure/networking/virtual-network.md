# Azure Virtual Network (VNet)

> Isolated, private network in Azure for secure communication between Azure resources.

## Overview

Azure Virtual Network (VNet) is the fundamental building block for private networks in Azure. It enables Azure resources to securely communicate with each other, the internet, and on-premises networks.

## Key Concepts

| Term | Definition |
|------|------------|
| VNet | Isolated network boundary in Azure |
| Subnet | Segment of VNet with its own address range |
| Network Interface (NIC) | Connects VM to VNet |
| Network Security Group (NSG) | Firewall rules for traffic filtering |
| Route Table | Custom routing for subnet traffic |
| Service Endpoint | Direct connection to Azure services |
| Private Endpoint | Private IP for Azure services in your VNet |
| Peering | Connect VNets together |

## VNet Architecture

```
+-------------------------------------------------------------+
|                   VNet: 10.0.0.0/16                         |
|                                                             |
|  +--------------------+    +--------------------+           |
|  | Subnet: Web        |    | Subnet: App        |           |
|  | 10.0.1.0/24        |    | 10.0.2.0/24        |           |
|  |  +----+  +----+    |    |  +----+  +----+    |           |
|  |  |VM1 |  |VM2 |    |    |  |VM3 |  |VM4 |    |           |
|  |  +----+  +----+    |    |  +----+  +----+    |           |
|  |  NSG: Web-NSG      |    |  NSG: App-NSG      |           |
|  +--------------------+    +--------------------+           |
|                                                             |
|  +--------------------+    +--------------------+           |
|  | Subnet: DB         |    | Subnet: Gateway    |           |
|  | 10.0.3.0/24        |    | 10.0.255.0/24      |           |
|  |  +----+            |    |  +-------------+   |           |
|  |  | SQL|            |    |  | VPN Gateway |   |           |
|  |  +----+            |    |  +-------------+   |           |
|  |  Private Endpoint  |    |                    |           |
|  +--------------------+    +--------------------+           |
+-------------------------------------------------------------+
```

## Address Space

### Reserved Addresses per Subnet

| Address | Purpose |
|---------|---------|
| .0 | Network address |
| .1 | Default gateway |
| .2, .3 | Azure DNS |
| .255 | Broadcast |

```
Subnet: 10.0.1.0/24
- Available IPs: 10.0.1.4 - 10.0.1.254 (251 addresses)
- Azure reserves 5 addresses per subnet
```

### Common CIDR Blocks

| CIDR | Total IPs | Usable IPs |
|------|-----------|------------|
| /24 | 256 | 251 |
| /25 | 128 | 123 |
| /26 | 64 | 59 |
| /27 | 32 | 27 |
| /28 | 16 | 11 |

## Subnets

### Special Subnets

| Subnet | Purpose | Requirements |
|--------|---------|--------------|
| GatewaySubnet | VPN/ExpressRoute gateways | Name must be "GatewaySubnet" |
| AzureFirewallSubnet | Azure Firewall | /26 minimum |
| AzureBastionSubnet | Azure Bastion | /26 minimum |
| RouteServerSubnet | Azure Route Server | /27 minimum |

### Subnet Delegation

Reserve subnet for specific Azure service:

```bash
# Delegate subnet to Azure Container Instances
az network vnet subnet update \
  --name mySubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --delegations Microsoft.ContainerInstance/containerGroups
```

## Network Security Groups (NSG)

### Rule Components

| Property | Description |
|----------|-------------|
| Priority | 100-4096 (lower = higher priority) |
| Source/Destination | IP, CIDR, Service Tag, or ASG |
| Port | Single, range, or * (all) |
| Protocol | TCP, UDP, ICMP, or * (all) |
| Action | Allow or Deny |
| Direction | Inbound or Outbound |

### Default Rules

| Priority | Rule | Direction |
|----------|------|-----------|
| 65000 | AllowVNetInBound | Inbound |
| 65001 | AllowAzureLoadBalancerInBound | Inbound |
| 65500 | DenyAllInBound | Inbound |
| 65000 | AllowVNetOutBound | Outbound |
| 65001 | AllowInternetOutBound | Outbound |
| 65500 | DenyAllOutBound | Outbound |

### Service Tags

Common service tags for NSG rules:

| Tag | Description |
|-----|-------------|
| VirtualNetwork | VNet + peered VNets |
| Internet | Public internet |
| AzureCloud | All Azure IPs |
| Storage | Azure Storage |
| Sql | Azure SQL Database |
| AzureActiveDirectory | Azure AD |
| AzureLoadBalancer | Azure LB health probes |

### Application Security Groups (ASG)

Group VMs logically for NSG rules:

```
+------------------------------------------------------+
|                      NSG Rule                         |
|  Source: ASG-WebServers                              |
|  Destination: ASG-AppServers                         |
|  Port: 8080                                          |
|  Action: Allow                                       |
+------------------------------------------------------+
              |                    |
     +--------+--------+    +-----+-----+
     | ASG-WebServers  |    |ASG-AppServers|
     |  VM1, VM2, VM3  |    |  VM4, VM5   |
     +-----------------+    +-------------+
```

## VNet Peering

### Types

| Type | Description |
|------|-------------|
| **VNet Peering** | Same region |
| **Global VNet Peering** | Cross-region |

### Peering Properties

```
VNet A                              VNet B
+------------------+    Peering    +------------------+
|   10.0.0.0/16    |<------------>|   10.1.0.0/16    |
|                  |               |                  |
|  Traffic flows   |               |  Traffic flows   |
|  directly over   |               |  over Microsoft  |
|  Azure backbone  |               |  backbone        |
+------------------+               +------------------+
```

### Configuration Options

| Setting | Description |
|---------|-------------|
| Allow forwarded traffic | Accept traffic forwarded by NVA |
| Allow gateway transit | Share VPN/ExpressRoute gateway |
| Use remote gateway | Use peered VNet's gateway |

## Connectivity Options

### VPN Gateway

| Type | Use Case | Bandwidth |
|------|----------|-----------|
| **Site-to-Site (S2S)** | On-premises to Azure | Up to 10 Gbps |
| **Point-to-Site (P2S)** | Individual devices | Up to 1 Gbps |
| **VNet-to-VNet** | Azure VNets via VPN | Up to 10 Gbps |

### ExpressRoute

Private connection to Azure (not over internet).

```
On-premises                 ExpressRoute                 Azure
+----------+    +-------------------------+    +----------+
| Datacenter|----    Private Connection   |----   VNet   |
|          |    |    50 Mbps - 100 Gbps   |    |          |
+----------+    +-------------------------+    +----------+
```

### Azure Bastion

Secure RDP/SSH without public IPs.

```
+-----------------------------------------------------+
|                      VNet                            |
|                                                      |
|  +--------------------------------------------+    |
|  |         AzureBastionSubnet                  |    |
|  |   +----------------------------------+     |    |
|  |   |         Azure Bastion            |     |    |
|  |   |   (Public IP, NSG managed)       |     |    |
|  |   +----------------------------------+     |    |
|  +--------------------------------------------+    |
|                        |                             |
|              RDP/SSH over TLS                       |
|                        |                             |
|  +---------------------v------------------------+  |
|  |              VM Subnet                         |  |
|  |   +------+  +------+  +------+              |  |
|  |   | VM1  |  | VM2  |  | VM3  |  (No public IP)|
|  |   +------+  +------+  +------+              |  |
|  +----------------------------------------------+  |
+-----------------------------------------------------+
```

## Service Endpoints vs Private Endpoints

| Feature | Service Endpoint | Private Endpoint |
|---------|------------------|------------------|
| Traffic path | Over Azure backbone | Private IP in VNet |
| Source IP | VNet subnet IP | Private IP |
| DNS resolution | Public endpoint | Private endpoint |
| Cross-region | No | Yes |
| On-premises access | No | Yes (via VPN/ER) |
| Cost | Free | Hourly + data |

### Service Endpoint

```bash
# Enable service endpoint for Storage
az network vnet subnet update \
  --name mySubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --service-endpoints Microsoft.Storage
```

### Private Endpoint

```bash
# Create private endpoint for Storage
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id <storage-account-id> \
  --group-id blob \
  --connection-name myConnection
```

## NAT Gateway

Outbound internet connectivity with static IP.

```
+-----------------------------------------------------+
|                      VNet                            |
|  +--------------------------------------------+    |
|  |              Subnet                          |    |
|  |   +------+  +------+  +------+             |    |
|  |   | VM1  |  | VM2  |  | VM3  |             |    |
|  |   +------+  +------+  +------+             |    |
|  +-------------------+--------------------------+  |
|                    |                                 |
|              +-----v-----+                          |
|              |NAT Gateway|                          |
|              | Public IP |                          |
|              +-----+-----+                          |
+--------------------+--------------------------------+
                     |
                     v
                 Internet
```

## Route Tables (UDR)

Custom routing for subnet traffic.

### Route Types

| Type | Description |
|------|-------------|
| **System routes** | Automatic (VNet, internet, peering) |
| **User-defined routes** | Custom routes you create |
| **BGP routes** | Learned via VPN/ExpressRoute |

### Common UDR Scenarios

```bash
# Route all traffic through firewall/NVA
az network route-table route create \
  --name ToFirewall \
  --route-table-name myRouteTable \
  --resource-group myRG \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4
```

## DNS

### Options

| Option | Description |
|--------|-------------|
| **Azure-provided DNS** | Default, hostname resolution in VNet |
| **Custom DNS** | Your own DNS servers |
| **Azure Private DNS** | Private zones for VNet resources |
| **Azure DNS** | Public domain hosting |

### Private DNS Zones

```
+-----------------------------------------------------+
|           Private DNS Zone: contoso.local           |
|                                                      |
|  Records:                                           |
|  +-- vm1.contoso.local -> 10.0.1.4                  |
|  +-- vm2.contoso.local -> 10.0.1.5                  |
|  +-- sql.contoso.local -> 10.0.2.4                  |
|                                                      |
|  Linked VNets:                                      |
|  +-- VNet-Prod (auto-registration enabled)         |
|  +-- VNet-Dev                                       |
+-----------------------------------------------------+
```

## CLI Quick Reference

```bash
# Create VNet
az network vnet create \
  --name myVNet \
  --resource-group myRG \
  --address-prefix 10.0.0.0/16 \
  --subnet-name default \
  --subnet-prefix 10.0.1.0/24

# Add subnet
az network vnet subnet create \
  --name AppSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --address-prefixes 10.0.2.0/24

# Create NSG
az network nsg create \
  --name myNSG \
  --resource-group myRG

# Add NSG rule
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name AllowHTTP \
  --priority 100 \
  --access Allow \
  --direction Inbound \
  --protocol Tcp \
  --destination-port-ranges 80 443

# Associate NSG to subnet
az network vnet subnet update \
  --name AppSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --network-security-group myNSG

# Create VNet peering
az network vnet peering create \
  --name VNet1-to-VNet2 \
  --resource-group myRG \
  --vnet-name VNet1 \
  --remote-vnet /subscriptions/.../VNet2 \
  --allow-vnet-access

# Create route table
az network route-table create \
  --name myRouteTable \
  --resource-group myRG

# Associate route table
az network vnet subnet update \
  --name AppSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --route-table myRouteTable
```

## Exam Tips (AZ-104, AZ-305)

1. **Address space**: Cannot overlap with peered VNets or on-premises
2. **NSG**: Stateful - return traffic automatically allowed
3. **NSG association**: Can attach to subnet AND NIC (both evaluated)
4. **Service tags**: Use instead of hardcoding Azure IPs
5. **Peering**: Non-transitive (A<->B, B<->C doesn't mean A<->C)
6. **Gateway transit**: Share VPN gateway with peered VNets
7. **Private Endpoint**: Creates NIC in your subnet with private IP
8. **Service Endpoint**: Traffic stays on Azure backbone but uses public endpoint
9. **NAT Gateway**: Provides static outbound IP, replaces default SNAT
10. **Bastion**: Requires AzureBastionSubnet, /26 minimum

## Gotchas

- VNet address spaces cannot be changed while resources exist (can add ranges)
- Peering is not transitive (need hub-spoke or mesh)
- NSG default rules cannot be deleted (only overridden)
- GatewaySubnet must not have an NSG
- Service endpoints are configured per subnet, not per resource
- Private DNS zone links require manual creation
- Changing subnet address range requires subnet to be empty
- NAT Gateway overrides other outbound rules
- ASGs must be in same region as resources

## Limits

| Resource | Limit |
|----------|-------|
| VNets per subscription per region | 1000 |
| Subnets per VNet | 3000 |
| VNet peerings per VNet | 500 |
| Private endpoints per VNet | 1000 |
| NSGs per subscription | 5000 |
| Rules per NSG | 1000 |
| Route tables per subscription | 200 |
| Routes per route table | 400 |
| Address prefixes per VNet | 1000 |
