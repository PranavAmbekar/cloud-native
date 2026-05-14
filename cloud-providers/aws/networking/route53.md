# Amazon Route 53

> Highly available and scalable DNS web service with domain registration and health checking.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Hosted Zone | Container for DNS records for a domain |
| Record Set | DNS record (A, AAAA, CNAME, etc.) |
| Alias | Route 53 specific record pointing to AWS resources |
| Health Check | Monitor endpoint availability |
| Routing Policy | How Route 53 responds to queries |
| TTL | Time to Live (cache duration) |

---

## DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | 192.0.2.1 |
| AAAA | IPv6 address | 2001:0db8::1 |
| CNAME | Canonical name (alias to another domain) | www → example.com |
| MX | Mail exchange | mail.example.com |
| TXT | Text (verification, SPF) | "v=spf1 ..." |
| NS | Name server | ns-xxx.awsdns-xx.com |
| SOA | Start of Authority | Primary NS info |
| SRV | Service locator | _sip._tcp.example.com |
| CAA | Certificate Authority Authorization | Allowed CAs |
| PTR | Reverse DNS | IP to domain |

---

## Hosted Zones

### Public Hosted Zone
- Routes traffic on the internet
- Domain must be registered
- $0.50/month per hosted zone

### Private Hosted Zone
- Routes traffic within VPC(s)
- Not accessible from internet
- Associate with one or more VPCs

```
┌─────────────────────────────────────────┐
│              Private Zone               │
│           internal.company.com          │
│                    │                    │
│    ┌───────────────┼───────────────┐    │
│    │               │               │    │
│    ▼               ▼               ▼    │
│  VPC-A          VPC-B           VPC-C   │
│                                         │
│   db.internal.company.com → 10.0.1.5   │
│   api.internal.company.com → 10.0.2.10 │
└─────────────────────────────────────────┘
```

---

## Alias Records

Route 53-specific record type pointing to AWS resources.

### Alias vs CNAME

| Feature | Alias | CNAME |
|---------|-------|-------|
| Zone apex | Yes | No |
| AWS resources | Native support | No |
| Charges | Free | Standard DNS |
| TTL | Inherited | Configurable |
| Health checks | Yes | Limited |

### Alias Targets
- CloudFront distributions
- ELB (ALB, NLB, CLB)
- S3 website endpoints
- API Gateway
- VPC Interface Endpoints
- Global Accelerator
- Another Route 53 record

```
# Cannot use CNAME at zone apex
example.com → CNAME → xxx.cloudfront.net  ❌

# Alias works at zone apex
example.com → ALIAS → xxx.cloudfront.net  ✓
```

---

## Routing Policies

### Simple Routing
```
query: www.example.com
       │
       ▼
   ┌───────┐
   │   A   │ → 1.2.3.4, 5.6.7.8
   └───────┘
   (random selection by client)
```
- One record, multiple values
- No health checks
- Client chooses randomly

### Weighted Routing
```
query: www.example.com
       │
       ├── 70% → 1.2.3.4 (weight: 70)
       ├── 20% → 5.6.7.8 (weight: 20)
       └── 10% → 9.10.11.12 (weight: 10)
```
- Distribute traffic by percentage
- A/B testing, gradual migration
- Health checks supported

### Latency Routing
```
query: www.example.com
       │
       ├── User in Tokyo → ap-northeast-1 endpoint
       ├── User in London → eu-west-1 endpoint
       └── User in NYC → us-east-1 endpoint
```
- Route to lowest latency region
- Based on AWS latency measurements
- Health checks supported

### Failover Routing
```
query: www.example.com
       │
       ├── Primary (us-east-1) ← Health Check
       │        │
       │        ▼ (if unhealthy)
       └── Secondary (us-west-2)
```
- Active-passive setup
- Primary must have health check
- Secondary used when primary fails

### Geolocation Routing
```
query: www.example.com
       │
       ├── User in US → us-endpoint
       ├── User in Europe → eu-endpoint
       ├── User in Asia → asia-endpoint
       └── Default → default-endpoint
```
- Route based on user location
- Continent, country, or US state
- Must set default record
- Content restrictions, localization

### Geoproximity Routing
```
                  ┌─────────────────┐
                  │   Traffic Flow  │
                  └─────────────────┘
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    ▼                    ▼                    ▼
 Region A            Region B            Region C
 (bias: +50)         (bias: 0)          (bias: -25)
    │                    │                    │
    └──── Expanded ──────┴──── Shrunk ────────┘
           coverage              coverage
```
- Route based on geographic distance
- Bias to expand/shrink regions
- Requires Traffic Flow (paid feature)

### Multi-Value Answer
```
query: www.example.com
       │
       ▼
   Returns up to 8 healthy records
   [1.2.3.4, 5.6.7.8, 9.10.11.12]
   (client-side load balancing)
```
- Up to 8 healthy records returned
- Health checks on each record
- Not a substitute for ELB

### IP-Based Routing
```
query: www.example.com
       │
       ├── Client IP in 203.0.113.0/24 → endpoint-1
       └── Client IP in 198.51.100.0/24 → endpoint-2
```
- Route based on client IP (CIDR)
- ISP-specific routing
- Enterprise traffic routing

---

## Health Checks

### Types
| Type | Monitors |
|------|----------|
| Endpoint | HTTP, HTTPS, TCP |
| Calculated | Status of other health checks |
| CloudWatch Alarm | CloudWatch metric state |

### Endpoint Health Check
```
Route 53 Health Checkers (global)
           │
           ▼
    ┌─────────────┐
    │  Endpoint   │
    │ /health-check │
    └─────────────┘

Settings:
- Protocol: HTTP/HTTPS/TCP
- Port: 80/443/custom
- Path: /health
- Interval: 30s (10s = fast, extra cost)
- Threshold: 3 (failures before unhealthy)
```

### Calculated Health Check
```
┌─────────────────────────────────────┐
│    Calculated Health Check          │
│                                     │
│   HC-1 ✓  AND  HC-2 ✓  AND  HC-3 ✗  │
│                 │                   │
│                 ▼                   │
│   Status: Unhealthy (1 of 3 failed) │
│   OR                                │
│   Status: Healthy (2 of 3 passed)   │
└─────────────────────────────────────┘
```
- Combine multiple health checks
- AND, OR, or threshold logic

### Private Resource Health Checks
- Health checkers are public
- Cannot directly check private endpoints

Solution:
```
Private Resource → CloudWatch Metric → CloudWatch Alarm → Route 53 Health Check
```

---

## Domain Registration

- Register directly with Route 53
- Transfer from other registrars
- Auto-renew option
- Domain lock protection

```bash
# Check availability
aws route53domains check-domain-availability --domain-name example.com

# Register domain
aws route53domains register-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --admin-contact file://contact.json \
  --registrant-contact file://contact.json \
  --tech-contact file://contact.json
```

---

## Traffic Flow

Visual policy editor for complex routing.

```
┌─────────────────────────────────────────────────┐
│              Traffic Flow Policy                │
│                                                 │
│   Start → Geolocation → Weighted → Endpoint    │
│              │             │                    │
│              │             ├── 70% → us-east-1 │
│              │             └── 30% → us-west-2 │
│              │                                  │
│              └── EU → eu-west-1                │
└─────────────────────────────────────────────────┘
```

- $50/month per policy record
- Version control
- Reusable across hosted zones

---

## DNSSEC

DNS Security Extensions - protect against DNS spoofing.

```bash
# Enable DNSSEC signing
aws route53 enable-hosted-zone-dnssec \
  --hosted-zone-id Z1234567890ABC

# Create key-signing key (KSK)
aws route53 create-key-signing-key \
  --hosted-zone-id Z1234567890ABC \
  --key-management-service-arn arn:aws:kms:... \
  --name my-ksk \
  --status ACTIVE
```

---

## CLI Quick Reference

```bash
# Create hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)

# List hosted zones
aws route53 list-hosted-zones

# Create record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "1.2.3.4"}]
      }
    }]
  }'

# Create alias record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "d123.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'

# Create health check
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "IPAddress": "1.2.3.4",
    "Port": 80,
    "Type": "HTTP",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# List records
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Hosted zone | $0.50/month |
| Queries (first 1B) | $0.40/million |
| Queries (over 1B) | $0.20/million |
| Health checks (basic) | $0.50/month |
| Health checks (advanced) | $1.00/month |
| Domain registration | Varies by TLD |

---

## Limits

| Resource | Limit |
|----------|-------|
| Hosted zones per account | 500 |
| Records per hosted zone | 10,000 |
| Health checks per account | 200 |
| Domains registered | 20 (soft limit) |

---

## Exam Tips

1. **Alias vs CNAME** - Alias for zone apex, AWS resources, free
2. **Simple routing** - no health checks, random client selection
3. **Weighted** - traffic distribution, A/B testing
4. **Latency** - route to lowest latency region
5. **Failover** - active-passive, requires health check on primary
6. **Geolocation** - must have default record
7. **Geoproximity** - requires Traffic Flow, use bias
8. **Multi-value** - up to 8 healthy records, client-side LB
9. **Health checks are public** - use CloudWatch for private resources
10. **Private hosted zone** - associate with VPCs
11. **TTL** - lower = more queries = more cost
12. **DNSSEC** - protects against DNS spoofing
