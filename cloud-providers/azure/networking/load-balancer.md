# Azure Load Balancer

> High-performance, ultra-low-latency Layer 4 (TCP/UDP) load balancer for Azure.

## Overview

Azure Load Balancer distributes inbound traffic across backend resources. It operates at Layer 4 (transport layer) and supports both inbound and outbound scenarios with low latency and high throughput.

## Key Concepts

| Term | Definition |
|------|------------|
| Frontend IP | Public or private IP receiving traffic |
| Backend Pool | Group of VMs or instances to receive traffic |
| Health Probe | Checks backend instance health |
| Load Balancing Rule | Maps frontend IP:port to backend pool |
| Inbound NAT Rule | Port forwarding to specific backend instance |
| Outbound Rule | Configures outbound connectivity |

## SKU Comparison

| Feature | Basic | Standard |
|---------|-------|----------|
| Backend pool size | 300 instances | 1000 instances |
| Backend pool type | Availability Set or Scale Set | Any VM in VNet |
| Health probes | TCP, HTTP | TCP, HTTP, HTTPS |
| Availability Zones | No | Yes |
| SLA | No SLA | 99.99% |
| Secure by default | No | Yes (closed inbound) |
| Diagnostics | Basic | Rich metrics, logs |
| Global/Cross-region | No | Yes |
| HA Ports | No | Yes |
| Pricing | Free | Charged |

**Important**: Basic SKU is being retired. Use Standard for new deployments.

## Architecture

### Public Load Balancer

```
                    Internet
                        │
                        ▼
              ┌─────────────────┐
              │ Public Frontend │
              │   IP: 1.2.3.4   │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Load Balancer  │
              │   (Standard)    │
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
   │   VM1   │    │   VM2   │    │   VM3   │
   │10.0.0.4 │    │10.0.0.5 │    │10.0.0.6 │
   └─────────┘    └─────────┘    └─────────┘
              Backend Pool
```

### Internal Load Balancer

```
    Web Tier                    App Tier
   ┌────────┐               ┌────────────────┐
   │  VM1   │───┐           │ Internal LB    │
   │  VM2   │───┼──────────▶│ 10.0.2.100     │
   │  VM3   │───┘           └───────┬────────┘
   └────────┘                       │
                     ┌──────────────┼──────────────┐
                     │              │              │
                ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
                │  AppVM1 │    │  AppVM2 │    │  AppVM3 │
                └─────────┘    └─────────┘    └─────────┘
```

## Health Probes

| Protocol | Description |
|----------|-------------|
| **TCP** | TCP connection attempt |
| **HTTP/HTTPS** | HTTP GET, expects 200 OK |

### Probe Configuration

```
Probe Settings:
- Protocol: HTTP
- Port: 80
- Path: /health
- Interval: 5 seconds
- Unhealthy threshold: 2 failures

Health check: http://<backend-ip>:80/health
```

### Probe Behavior

```
Healthy VM (responds to probes)
├── Receives traffic from LB
└── Counted in active backend pool

Unhealthy VM (fails probes)
├── Removed from rotation
├── No new connections
└── Existing connections may continue
```

## Load Balancing Rules

### Distribution Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **5-tuple hash** | Source IP, port, dest IP, port, protocol | Default, most scenarios |
| **2-tuple hash** | Source IP, dest IP | Session affinity |
| **3-tuple hash** | Source IP, dest IP, protocol | Session affinity |

### Session Persistence

| Setting | Hash | Effect |
|---------|------|--------|
| None | 5-tuple | Requests distributed evenly |
| Client IP | 2-tuple | Same client → same backend |
| Client IP and protocol | 3-tuple | Same client+protocol → same backend |

## HA Ports

Load balance all TCP and UDP traffic on all ports (Standard SKU only).

```
Rule Configuration:
- Protocol: All
- Frontend Port: 0 (all ports)
- Backend Port: 0 (all ports)

Use cases:
- Network Virtual Appliances (NVA)
- Internal load balancing for any port
- SQL AlwaysOn
```

## Outbound Connectivity

### SNAT (Source NAT)

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  Backend VM │ ──▶  │Load Balancer│ ──▶  │  Internet   │
│ 10.0.0.4    │      │ 1.2.3.4     │      │(dest: 5.6.7.8)
└─────────────┘      └─────────────┘      └─────────────┘

Source IP: 10.0.0.4 → Translated to: 1.2.3.4
```

### Outbound Rules

Control outbound connectivity explicitly:

| Setting | Description |
|---------|-------------|
| Frontend IP | Public IP(s) for outbound |
| Backend pool | Which VMs can use this rule |
| Allocated ports | SNAT ports per instance |
| Idle timeout | TCP idle timeout (4-120 min) |

```
SNAT Port Allocation:
- Default: ~1000 ports per VM
- Explicit: Define in outbound rule
- More frontend IPs = more ports
```

## Inbound NAT Rules

Port forwarding to specific backend instances.

```
Frontend IP: 1.2.3.4
┌─────────────────────────────────┐
│ Port 50001 ──────▶ VM1:22 (SSH) │
│ Port 50002 ──────▶ VM2:22 (SSH) │
│ Port 50003 ──────▶ VM3:22 (SSH) │
└─────────────────────────────────┘

Connect: ssh user@1.2.3.4 -p 50001 → VM1
```

## Cross-Region Load Balancer

Global load balancing across regions (Standard SKU).

```
                     Global LB
                   ┌───────────┐
                   │           │
        ┌──────────┴───────────┴──────────┐
        │                                  │
   ┌────▼────┐                        ┌────▼────┐
   │Regional │                        │Regional │
   │LB (East)│                        │LB (West)│
   └────┬────┘                        └────┬────┘
        │                                  │
   ┌────▼────┐                        ┌────▼────┐
   │ Backend │                        │ Backend │
   │  Pool   │                        │  Pool   │
   └─────────┘                        └─────────┘
```

## CLI Quick Reference

```bash
# Create public IP
az network public-ip create \
  --name myPublicIP \
  --resource-group myRG \
  --sku Standard \
  --allocation-method Static

# Create load balancer
az network lb create \
  --name myLoadBalancer \
  --resource-group myRG \
  --sku Standard \
  --frontend-ip-name myFrontend \
  --public-ip-address myPublicIP \
  --backend-pool-name myBackendPool

# Create health probe
az network lb probe create \
  --name myHealthProbe \
  --lb-name myLoadBalancer \
  --resource-group myRG \
  --protocol Http \
  --port 80 \
  --path /health

# Create load balancing rule
az network lb rule create \
  --name myLBRule \
  --lb-name myLoadBalancer \
  --resource-group myRG \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name myFrontend \
  --backend-pool-name myBackendPool \
  --probe-name myHealthProbe

# Add VM to backend pool
az network nic ip-config address-pool add \
  --address-pool myBackendPool \
  --ip-config-name ipconfig1 \
  --nic-name myNIC \
  --resource-group myRG \
  --lb-name myLoadBalancer

# Create inbound NAT rule
az network lb inbound-nat-rule create \
  --name SSHtoVM1 \
  --lb-name myLoadBalancer \
  --resource-group myRG \
  --protocol Tcp \
  --frontend-port 50001 \
  --backend-port 22 \
  --frontend-ip-name myFrontend

# Create outbound rule
az network lb outbound-rule create \
  --name myOutboundRule \
  --lb-name myLoadBalancer \
  --resource-group myRG \
  --frontend-ip-configs myFrontend \
  --protocol All \
  --idle-timeout 4 \
  --outbound-ports 10000 \
  --address-pool myBackendPool
```

## Comparison with Other Azure Load Balancers

| Feature | Load Balancer | Application Gateway | Front Door |
|---------|---------------|---------------------|------------|
| Layer | 4 (TCP/UDP) | 7 (HTTP/HTTPS) | 7 (HTTP/HTTPS) |
| Scope | Regional | Regional | Global |
| SSL termination | No | Yes | Yes |
| URL-based routing | No | Yes | Yes |
| WAF | No | Yes | Yes |
| Session affinity | IP-based | Cookie-based | Cookie-based |
| Health probes | TCP, HTTP(S) | HTTP(S) | HTTP(S) |

## Exam Tips (AZ-104, AZ-305)

1. **Standard vs Basic**: Standard has SLA, zones, secure-by-default
2. **Health probes**: HTTP probes expect 200 response
3. **HA Ports**: All protocols, all ports (NVA scenarios)
4. **Session persistence**: None (5-tuple), Client IP (2-tuple)
5. **SNAT exhaustion**: Use outbound rules or NAT Gateway
6. **Cross-region LB**: Global tier for multi-region
7. **Basic SKU retirement**: Being phased out, use Standard
8. **Secure by default**: Standard LB requires NSG to allow traffic
9. **Backend pool limits**: Standard = 1000, Basic = 300
10. **Floating IP**: Required for SQL AlwaysOn, HA scenarios

## Gotchas

- Basic SKU has no SLA and is being retired
- Standard SKU is secure by default (needs NSG to allow traffic)
- Backend VMs need to be in same VNet (Standard can be any in VNet)
- Health probes come from 168.63.129.16 (Azure infrastructure)
- Must allow probe traffic in NSG
- SNAT port exhaustion with many outbound connections (use NAT Gateway)
- Floating IP changes how backend sees destination IP
- Cross-region LB requires Standard SKU regional LBs
- Internal LB needs explicit outbound connectivity configuration
- Basic LB doesn't support availability zones

## Limits

| Resource | Limit |
|----------|-------|
| Load balancers per subscription | 1000 per region |
| Frontend IPs per LB | 600 |
| Backend pool instances (Standard) | 1000 |
| Backend pool instances (Basic) | 300 |
| Load balancing rules per LB | 1500 |
| Inbound NAT rules per LB | 1000 |
| Health probes per LB | 1000 |
| Private frontend IPs per LB | 600 |
