# Elastic Load Balancing (ELB)

> Distribute incoming traffic across multiple targets for high availability.

---

## Load Balancer Types

| Type | Layer | Use Case |
|------|-------|----------|
| ALB (Application) | 7 (HTTP/HTTPS) | Web apps, microservices, containers |
| NLB (Network) | 4 (TCP/UDP) | Extreme performance, static IP |
| GLB (Gateway) | 3 (IP) | Third-party virtual appliances |
| CLB (Classic) | 4/7 | Legacy (avoid for new apps) |

---

## Application Load Balancer (ALB)

### Features
- HTTP/HTTPS (Layer 7)
- Path-based routing
- Host-based routing
- Query string/header routing
- WebSocket support
- HTTP/2 support
- gRPC support
- Sticky sessions
- User authentication (Cognito, OIDC)

### Architecture

```
+------------------------------------------------------------+
|                           ALB                              |
|                                                            |
|  +-----------------------------------------------------+   |
|  |                   Listener (443)                    |   |
|  |                                                     |   |
|  |   Rule: /api/*     ------>  Target Group: API       |   |
|  |   Rule: /images/*  ------>  Target Group: Static    |   |
|  |   Rule: Host=m.*   ------>  Target Group: Mobile    |   |
|  |   Default          ------>  Target Group: Web       |   |
|  |                                                     |   |
|  +-----------------------------------------------------+   |
|                                                            |
+------------------------------------------------------------+
           |              |               |
           v              v               v
      +--------+    +--------+     +--------+
      |  EC2   |    |  EC2   |     |  ECS   |
      |Targets |    |Targets |     |Targets |
      +--------+    +--------+     +--------+
```

### Routing Rules

```
Priority | Condition                  | Action
---------|----------------------------|------------------
1        | Path = /api/*              | Forward to API TG
2        | Host = api.example.com     | Forward to API TG
3        | Header X-Custom = mobile   | Forward to Mobile TG
4        | Query string ?platform=ios | Forward to iOS TG
5        | Default                    | Forward to Web TG
```

### Target Types
| Type | Description |
|------|-------------|
| instance | EC2 instance ID |
| ip | IP address (private IPs, on-prem) |
| lambda | Lambda function |

---

## Network Load Balancer (NLB)

### Features
- TCP/UDP/TLS (Layer 4)
- Extreme performance (millions of requests/sec)
- Ultra-low latency
- Static IP per AZ
- Elastic IP support
- Preserve source IP
- Long-lived TCP connections

### Architecture

```
+------------------------------------------------+
|                      NLB                       |
|                                                |
|   +-----------------------------------------+  |
|   |         Listener (TCP:443)              |  |
|   |                 |                       |  |
|   |                 v                       |  |
|   |         Target Group                    |  |
|   |     (TCP health checks)                 |  |
|   +-----------------------------------------+  |
|                                                |
|   Static IP: 1.2.3.4 (AZ-a)                    |
|   Static IP: 5.6.7.8 (AZ-b)                    |
|                                                |
+------------------------------------------------+
```

### Target Types
| Type | Description |
|------|-------------|
| instance | EC2 instance ID |
| ip | IP address |
| alb | Application Load Balancer |

### NLB + ALB
```
Client → NLB (static IP) → ALB (L7 features) → Targets
```

Use case: Need static IP with advanced routing.

---

## Gateway Load Balancer (GLB)

### Features
- Layer 3 (IP packets)
- Transparent to applications
- For security appliances (firewalls, IDS/IPS)
- Uses GENEVE protocol (port 6081)

### Architecture

```
+-------------------------------------------------------------+
|                                                             |
|   Traffic  -->  GLB  -->  Appliances  -->  GLB  -->  App    |
|                           (inspect)                         |
|                                                             |
+-------------------------------------------------------------+
```

---

## Target Groups

### Health Checks

```
Target Group
    |
    +-- Health Check Settings
    |   +-- Protocol: HTTP
    |   +-- Path: /health
    |   +-- Port: traffic-port
    |   +-- Healthy threshold: 2
    |   +-- Unhealthy threshold: 3
    |   +-- Timeout: 5 seconds
    |   +-- Interval: 30 seconds
    |   +-- Success codes: 200-299
    |
    +-- Targets
        +-- EC2: i-xxx (healthy)
        +-- EC2: i-yyy (healthy)
        +-- EC2: i-zzz (unhealthy - removed from rotation)
```

### Target States
| State | Description |
|-------|-------------|
| initial | Registering |
| healthy | Passing health checks |
| unhealthy | Failing health checks |
| unused | Not in AZ or deregistering |
| draining | Deregistering, finishing requests |

### Deregistration Delay
- Wait time before removing target
- Default: 300 seconds
- Allows in-flight requests to complete

---

## Sticky Sessions

### Application Cookie
```
Set-Cookie: AWSALB=xxx; Expires=...; Path=/
```
- ALB generates cookie
- Routes same user to same target

### Duration-based
- Specify TTL
- Automatic rotation

### Application Cookie (Custom)
- App generates cookie
- ALB uses it for routing

---

## Cross-Zone Load Balancing

```
Without Cross-Zone:              With Cross-Zone:
+----------+  +----------+      +----------+  +----------+
|   AZ-a   |  |   AZ-b   |      |   AZ-a   |  |   AZ-b   |
|          |  |          |      |          |  |          |
| 50% each |  | 50% each |      | 25% each |  | 25% each |
|   +-+    |  |   +-+    |      |   +-+    |  |   +-+    |
|   |1|    |  |   |3|    |      |   |1|    |  |   |3|    |
|   +-+    |  |   |4|    |      |   +-+    |  |   |4|    |
|          |  |   |5|    |      |          |  |   |5|    |
|          |  |   |6|    |      |          |  |   |6|    |
+----------+  +----------+      +----------+  +----------+
 1 gets 50%    3,4,5,6 get       Each gets 25%
               12.5% each
```

| LB Type | Default | Cost |
|---------|---------|------|
| ALB | Enabled | Free |
| NLB | Disabled | Charges apply |
| CLB | Disabled | Free |

---

## SSL/TLS Termination

### ALB
```
Client --HTTPS--> ALB --HTTP--> Targets
                   |
            SSL terminated here
```

- Manage certs with ACM
- Multiple certs (SNI)
- Security policies (TLS versions, ciphers)

### NLB
- TLS termination OR pass-through
- TCP pass-through preserves client certificate

### SNI (Server Name Indication)
- Multiple certificates on one listener
- Route based on hostname

```
client1.com --> cert1 --> Target Group 1
client2.com --> cert2 --> Target Group 2
```

---

## Connection Draining

```
1. Target marked for deregistration
2. Stop sending NEW requests
3. Wait for IN-FLIGHT requests (draining period)
4. After timeout, force close
```

Configure: 0-3600 seconds (0 = disabled)

---

## Access Logs

```
bucket/prefix/AWSLogs/account-id/elasticloadbalancing/region/yyyy/mm/dd/

Fields:
- timestamp
- elb
- client:port
- target:port
- request_processing_time
- target_processing_time
- response_processing_time
- elb_status_code
- target_status_code
- request
- user_agent
- ssl_cipher
- ssl_protocol
```

---

## CLI Quick Reference

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name my-alb \
  --type application \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx

# Create NLB
aws elbv2 create-load-balancer \
  --name my-nlb \
  --type network \
  --subnets subnet-xxx subnet-yyy

# Create target group
aws elbv2 create-target-group \
  --name my-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxx \
  --health-check-path /health

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-xxx Id=i-yyy

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Create rule
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:... \
  --priority 10
```

---

## Pricing

| Component | ALB | NLB |
|-----------|-----|-----|
| Hourly | $0.0225/hr | $0.0225/hr |
| LCU | $0.008/LCU-hr | $0.006/NLCU-hr |

**LCU (Load Balancer Capacity Unit)**: Based on new connections, active connections, processed bytes, rule evaluations.

---

## ALB vs NLB

| Feature | ALB | NLB |
|---------|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| Latency | ~400ms | ~100μs |
| Static IP | No (use Global Accelerator) | Yes |
| Preserve source IP | No (X-Forwarded-For) | Yes |
| SSL termination | Yes | Yes/Pass-through |
| WebSocket | Yes | Yes |
| Path routing | Yes | No |
| Host routing | Yes | No |
| Lambda targets | Yes | No |
| Security Groups | Yes | No (NACLs only) |

---

## Exam Tips

1. **ALB = Layer 7** - HTTP routing, path/host-based
2. **NLB = Layer 4** - extreme performance, static IP
3. **GLB = Layer 3** - security appliances, GENEVE
4. **Cross-zone** - ALB enabled by default, NLB disabled
5. **Sticky sessions** - maintain user-target affinity
6. **SNI** - multiple SSL certs on one listener
7. **Connection draining** - graceful deregistration
8. **X-Forwarded-For** - client IP header (ALB)
9. **NLB preserves source IP** - directly to target
10. **Lambda target** - ALB only
11. **Health checks** - unhealthy targets removed from rotation
12. **Access logs** - to S3, must enable
