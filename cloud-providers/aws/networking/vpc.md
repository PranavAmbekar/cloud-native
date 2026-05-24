# Amazon VPC (Virtual Private Cloud)

> Isolated virtual network where you launch AWS resources with full control over networking.

## Overview

VPC lets you define a logically isolated network in AWS. You control IP ranges, subnets, route tables, gateways, and security settings.

## Key Concepts

| Term | Definition |
|------|------------|
| VPC | Virtual network dedicated to your AWS account |
| CIDR Block | IP address range (e.g., 10.0.0.0/16) |
| Subnet | Segment of VPC's IP range in a single AZ |
| Route Table | Rules for where network traffic is directed |
| Internet Gateway | Enables internet access for VPC |
| NAT Gateway | Allows private instances to access internet |
| Security Group | Stateful firewall at instance level |
| Network ACL | Stateless firewall at subnet level |

## VPC Architecture

```
+------------------------------------------------------------------+
| VPC: 10.0.0.0/16                                                 |
|                                                                  |
|  +---------------------------------------------------------+    |
|  | Public Subnet: 10.0.1.0/24 (AZ-a)                        |    |
|  |  +-----------+                                           |    |
|  |  |    EC2    | <-- Security Group                        |    |
|  |  |  Public   |     (stateful)                            |    |
|  |  |    IP     |                                           |    |
|  |  +-----------+                                           |    |
|  +---------------------------------------------------------+    |
|       |                                                          |
|       | Route: 0.0.0.0/0 -> Internet Gateway                     |
|       v                                                          |
|  +-----------+                                                   |
|  |    IGW    | <----------- Internet                             |
|  +-----------+                                                   |
|                                                                  |
|  +---------------------------------------------------------+    |
|  | Private Subnet: 10.0.2.0/24 (AZ-a)                       |    |
|  |  +-----------+                                           |    |
|  |  |    EC2    | (no public IP)                            |    |
|  |  |  Private  |                                           |    |
|  |  +-----------+                                           |    |
|  +---------------------------------------------------------+    |
|       |                                                          |
|       | Route: 0.0.0.0/0 -> NAT Gateway                          |
|       v                                                          |
|  +-----------+                                                   |
|  |    NAT    | (in public subnet)                                |
|  |  Gateway  |                                                   |
|  +-----------+                                                   |
|                                                                  |
+------------------------------------------------------------------+
```

## CIDR Notation

| CIDR | # IPs | Range Example |
|------|-------|---------------|
| /16 | 65,536 | 10.0.0.0 - 10.0.255.255 |
| /20 | 4,096 | 10.0.0.0 - 10.0.15.255 |
| /24 | 256 | 10.0.0.0 - 10.0.0.255 |
| /28 | 16 | 10.0.0.0 - 10.0.0.15 |

**VPC CIDR**: Min /28 (16 IPs), Max /16 (65,536 IPs)

**Reserved IPs per subnet** (5 addresses):
- .0 - Network address
- .1 - VPC router
- .2 - DNS server
- .3 - Reserved for future
- .255 - Broadcast (not used but reserved)

## Internet Gateway (IGW)

- One IGW per VPC
- Horizontally scaled, redundant, highly available
- Provides NAT for instances with public IPs
- Must update route table: `0.0.0.0/0 -> igw-xxxxx`

## NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| Managed | AWS managed | You manage |
| Availability | HA within AZ | Single instance |
| Bandwidth | Up to 100 Gbps | Depends on instance type |
| Cost | Per hour + data | Instance cost |
| Security Groups | No | Yes |
| Bastion Host | No | Can be used as |

**NAT Gateway HA**: Deploy in each AZ, update route tables per AZ

## Route Tables

```
Destination     Target          Notes
-----------     ------          -----
10.0.0.0/16     local           VPC internal (automatic)
0.0.0.0/0       igw-xxxxx       Internet (public subnet)
0.0.0.0/0       nat-xxxxx       Internet via NAT (private subnet)
10.1.0.0/16     pcx-xxxxx       Peered VPC
```

- Each subnet must be associated with ONE route table
- Main route table is default for unassociated subnets
- Most specific route wins

## Security Groups vs NACLs

| Feature | Security Group | Network ACL |
|---------|----------------|-------------|
| Level | Instance (ENI) | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow AND Deny |
| Evaluation | All rules | Rules in order (lowest number first) |
| Default | Deny all in, allow all out | Allow all |
| Association | Multiple SGs per instance | One NACL per subnet |

### Security Group Example
```
Inbound:
  Type        Port    Source
  SSH         22      10.0.0.0/16
  HTTP        80      0.0.0.0/0
  Custom TCP  3000    sg-xxxxxxxx  (another SG)

Outbound:
  All traffic  All     0.0.0.0/0
```

### NACL Example
```
Inbound:
  Rule#   Type    Port      Source        Allow/Deny
  100     HTTP    80        0.0.0.0/0     ALLOW
  110     HTTPS   443       0.0.0.0/0     ALLOW
  120     SSH     22        10.0.0.0/16   ALLOW
  *       All     All       0.0.0.0/0     DENY

Outbound:
  Rule#   Type    Port        Dest          Allow/Deny
  100     Custom  1024-65535  0.0.0.0/0     ALLOW  (ephemeral ports)
  *       All     All         0.0.0.0/0     DENY
```

## VPC Peering

- Connect two VPCs (same or different accounts/regions)
- Not transitive: A<->B and B<->C does NOT mean A<->C
- No overlapping CIDR blocks
- Must update route tables in BOTH VPCs

```
VPC-A (10.0.0.0/16) <--PCX--> VPC-B (172.16.0.0/16)

VPC-A Route Table:         VPC-B Route Table:
172.16.0.0/16 -> pcx-xxx    10.0.0.0/16 -> pcx-xxx
```

## Transit Gateway

- Hub for connecting VPCs, VPNs, and Direct Connect
- Solves VPC peering mesh problem
- Regional but can be peered cross-region
- Supports thousands of VPCs

```
       +-------+
       | VPC-A |
       +---+---+
           |
    +------+------+
    |   Transit   |
    |   Gateway   |
    +------+------+
      /    |    \
+-----+ +-----+ +-----+
| VPC | | VPC | | VPN |
|  B  | |  C  | |     |
+-----+ +-----+ +-----+
```

## VPC Endpoints

### Gateway Endpoints (Free)
- S3 and DynamoDB only
- Route table entry points to endpoint
- Regional service

### Interface Endpoints (PrivateLink)
- Most AWS services
- ENI in your subnet with private IP
- Costs: hourly + data transfer
- Can access across regions (with additional setup)

```
Private Subnet -> ENI (Interface Endpoint) -> AWS Service
                       (private IP)
```

## VPN & Direct Connect

| Feature | Site-to-Site VPN | Direct Connect |
|---------|------------------|----------------|
| Connection | Over internet (encrypted) | Dedicated private line |
| Setup time | Minutes | Weeks to months |
| Bandwidth | Up to 1.25 Gbps | 1 Gbps to 100 Gbps |
| Latency | Variable | Consistent |
| Cost | Lower | Higher |
| Redundancy | Easy (multiple tunnels) | Requires additional circuit |

### Site-to-Site VPN Components
- Virtual Private Gateway (VGW) - AWS side
- Customer Gateway - Your side
- Two tunnels for redundancy

### Direct Connect + VPN
- Use VPN over Direct Connect for encryption
- Direct Connect alone is NOT encrypted

## VPC Flow Logs

Capture IP traffic information:
- VPC level, Subnet level, or ENI level
- Published to CloudWatch Logs, S3, or Kinesis Firehose
- Does NOT capture: DNS, DHCP, metadata (169.254.169.254), Windows license activation

```
2 123456789012 eni-xxx 10.0.1.5 10.0.2.10 443 49152 6 25 20000 1620000000 1620000060 ACCEPT OK

Fields: version account-id eni src-addr dst-addr src-port dst-port protocol packets bytes start end action log-status
```

## IPv6

- All IPv6 addresses are public (no NAT needed)
- Egress-only Internet Gateway for outbound-only IPv6
- Dual-stack: instances can have both IPv4 and IPv6

## CLI Quick Reference

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create subnet
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a

# Create Internet Gateway and attach
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx

# Create route to IGW
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx
```

## Exam Tips

1. **Public subnet**: Route table has route to IGW
2. **Private subnet**: No direct route to IGW, uses NAT for outbound
3. **Bastion Host**: Public subnet instance to SSH into private instances
4. **Security Groups are stateful**: Return traffic automatically allowed
5. **NACLs are stateless**: Must explicitly allow return traffic (ephemeral ports)
6. **VPC Peering is NOT transitive**: Need direct peering or Transit Gateway
7. **Gateway Endpoints are FREE**: Use for S3 and DynamoDB
8. **CIDR cannot overlap**: For peering, endpoints, or any connection
9. **NAT Gateway**: HA within AZ, deploy per AZ for full HA
10. **Direct Connect is NOT encrypted**: Add VPN on top if needed

## Gotchas

- Cannot change VPC CIDR after creation (can add secondary CIDRs)
- Deleting VPC requires removing all dependencies first
- Default VPC has internet access by default (public subnets)
- Security group changes take effect immediately
- NAT Gateway costs: $0.045/hr + $0.045/GB processed

## Limits

| Resource | Default Limit |
|----------|---------------|
| VPCs per region | 5 |
| Subnets per VPC | 200 |
| Route tables per VPC | 200 |
| Routes per route table | 50 |
| Security groups per VPC | 500 |
| Rules per security group | 60 in + 60 out |
| Network ACLs per VPC | 200 |
| Rules per NACL | 20 |
| Elastic IPs per region | 5 |
| VPC peering connections | 50 |
