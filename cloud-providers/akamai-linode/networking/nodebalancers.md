# Linode NodeBalancers

> High-availability load balancers for distributing traffic across Linodes.

## Overview

NodeBalancers are highly-available, managed load balancers that distribute incoming traffic across multiple Linodes. They support TCP, HTTP, and HTTPS protocols with health checking and session persistence.

## Key Concepts

| Term | Definition |
|------|------------|
| NodeBalancer | Load balancer instance |
| Config | Port/protocol configuration |
| Node | Backend Linode with IP:port |
| Health Check | Backend availability monitoring |
| Sticky Session | Session affinity/persistence |
| Throttle | Connection rate limiting |

## Architecture

```
                          Internet
                              |
                              v
+---------------------------------------------------------------+
|                      NodeBalancer                             |
|                     203.0.113.10                              |
|                                                               |
|  +------------------------+  +------------------------+       |
|  |   Config: Port 443     |  |   Config: Port 80      |       |
|  |   Protocol: HTTPS      |  |   Protocol: HTTP       |       |
|  |   SSL Termination      |  |   Redirect to 443      |       |
|  +------------------------+  +------------------------+       |
|               |                                               |
+---------------+-----------------------------------------------+
                |
    +-----------+-----------+
    |           |           |
    v           v           v
+-------+   +-------+   +-------+
| Node  |   | Node  |   | Node  |
| Web-1 |   | Web-2 |   | Web-3 |
| :8080 |   | :8080 |   | :8080 |
+-------+   +-------+   +-------+
```

## Pricing

| Component | Cost |
|-----------|------|
| NodeBalancer | $10/month |
| Transfer out | Included in Linode pool |
| SSL/TLS | Free (bring your own cert) |
| Additional configs | Free (same NodeBalancer) |

## Create NodeBalancer

### CLI

```bash
# Create NodeBalancer
linode-cli nodebalancers create \
  --region us-east \
  --label my-nodebalancer

# Output: NodeBalancer ID and public IP

# Add config (port/protocol)
linode-cli nodebalancers config-create 12345 \
  --port 80 \
  --protocol http \
  --algorithm roundrobin \
  --check http \
  --check_path /health \
  --check_interval 10 \
  --check_timeout 5 \
  --check_attempts 3

# Add backend nodes
linode-cli nodebalancers node-create 12345 54321 \
  --label web-1 \
  --address 192.168.1.100:8080 \
  --weight 100 \
  --mode accept

linode-cli nodebalancers node-create 12345 54321 \
  --label web-2 \
  --address 192.168.1.101:8080 \
  --weight 100 \
  --mode accept
```

### API

```bash
# Create NodeBalancer
curl -X POST https://api.linode.com/v4/nodebalancers \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "region": "us-east",
    "label": "my-nodebalancer"
  }'

# Add config
curl -X POST https://api.linode.com/v4/nodebalancers/12345/configs \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "port": 80,
    "protocol": "http",
    "algorithm": "roundrobin",
    "check": "http",
    "check_path": "/health"
  }'
```

### Terraform

```hcl
resource "linode_nodebalancer" "web" {
  label  = "my-nodebalancer"
  region = "us-east"
}

resource "linode_nodebalancer_config" "http" {
  nodebalancer_id = linode_nodebalancer.web.id
  port            = 80
  protocol        = "http"
  algorithm       = "roundrobin"
  check           = "http"
  check_path      = "/health"
  check_interval  = 10
  check_timeout   = 5
  check_attempts  = 3
  stickiness      = "none"
}

resource "linode_nodebalancer_node" "web1" {
  nodebalancer_id = linode_nodebalancer.web.id
  config_id       = linode_nodebalancer_config.http.id
  label           = "web-1"
  address         = "${linode_instance.web1.private_ip_address}:8080"
  weight          = 100
  mode            = "accept"
}
```

## Protocols

| Protocol | Port | Features |
|----------|------|----------|
| **TCP** | Any | Raw TCP passthrough |
| **HTTP** | 80 | X-Forwarded headers, sticky sessions |
| **HTTPS** | 443 | SSL termination, certificates |

### HTTPS Configuration

```bash
# Create HTTPS config with SSL certificate
linode-cli nodebalancers config-create 12345 \
  --port 443 \
  --protocol https \
  --ssl_cert "$(cat server.crt)" \
  --ssl_key "$(cat server.key)" \
  --algorithm roundrobin \
  --check http \
  --check_path /health
```

## Load Balancing Algorithms

| Algorithm | Description |
|-----------|-------------|
| **roundrobin** | Equal distribution in rotation |
| **leastconn** | Fewest active connections |
| **source** | Hash of client IP (session affinity) |

```bash
# Update algorithm
linode-cli nodebalancers config-update 12345 54321 \
  --algorithm leastconn
```

## Health Checks

| Check Type | Description |
|------------|-------------|
| **none** | No health checks |
| **connection** | TCP connection test |
| **http** | HTTP request, expect 2xx/3xx |
| **http_body** | HTTP request with body regex match |

```bash
# Configure HTTP health check
linode-cli nodebalancers config-update 12345 54321 \
  --check http \
  --check_path /health \
  --check_interval 10 \
  --check_timeout 5 \
  --check_attempts 3

# Configure HTTP body check
linode-cli nodebalancers config-update 12345 54321 \
  --check http_body \
  --check_path /health \
  --check_body "OK"
```

### Health Check Flow

```
NodeBalancer                    Backend Node
     |                               |
     |------ GET /health ----------->|
     |                               |
     |<----- 200 OK "healthy" -------|
     |                               |
     |  Node marked: UP              |
     |                               |
     |------ GET /health ----------->|
     |                               |
     |<----- 500 Error --------------|
     |                               |
     |  Attempt 2 of 3               |
     |                               |
```

## Sticky Sessions

| Stickiness | Description |
|------------|-------------|
| **none** | No persistence |
| **table** | IP-based persistence (in-memory) |
| **http_cookie** | Cookie-based persistence |

```bash
# Enable cookie-based sticky sessions
linode-cli nodebalancers config-update 12345 54321 \
  --stickiness http_cookie \
  --cookie_name SERVERID \
  --cookie_ttl 300
```

## Node Modes

| Mode | Description |
|------|-------------|
| **accept** | Accept traffic |
| **reject** | Reject new connections, drain existing |
| **drain** | Stop new connections, complete existing |
| **backup** | Only used when all others are down |

```bash
# Put node in drain mode for maintenance
linode-cli nodebalancers node-update 12345 54321 67890 \
  --mode drain

# Mark as backup
linode-cli nodebalancers node-update 12345 54321 67890 \
  --mode backup
```

## Connection Throttling

```bash
# Limit connections per second (0 = unlimited)
linode-cli nodebalancers config-update 12345 54321 \
  --throttle 10000
```

## Proxy Protocol

```bash
# Enable Proxy Protocol v1
linode-cli nodebalancers config-update 12345 54321 \
  --proxy_protocol v1

# Enable Proxy Protocol v2
linode-cli nodebalancers config-update 12345 54321 \
  --proxy_protocol v2
```

Backend configuration (nginx):

```nginx
server {
    listen 8080 proxy_protocol;

    real_ip_header proxy_protocol;
    set_real_ip_from 192.168.0.0/16;
}
```

## X-Forwarded Headers

NodeBalancer adds these headers (HTTP/HTTPS):

| Header | Value |
|--------|-------|
| X-Forwarded-For | Client IP address |
| X-Forwarded-Proto | http or https |
| X-Forwarded-Port | Original port |

## SSL/TLS Certificates

### Upload Certificate

```bash
# Create HTTPS config with certificate
linode-cli nodebalancers config-create 12345 \
  --port 443 \
  --protocol https \
  --ssl_cert "-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAJC1...
-----END CERTIFICATE-----" \
  --ssl_key "-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQ...
-----END PRIVATE KEY-----"
```

### Let's Encrypt with certbot

```bash
# On a Linode, get certificate
certbot certonly --standalone -d example.com

# Upload to NodeBalancer
linode-cli nodebalancers config-update 12345 54321 \
  --ssl_cert "$(cat /etc/letsencrypt/live/example.com/fullchain.pem)" \
  --ssl_key "$(cat /etc/letsencrypt/live/example.com/privkey.pem)"
```

## CLI Quick Reference

```bash
# NodeBalancers
linode-cli nodebalancers list
linode-cli nodebalancers create --region us-east --label my-nb
linode-cli nodebalancers view 12345
linode-cli nodebalancers delete 12345

# Configs
linode-cli nodebalancers configs-list 12345
linode-cli nodebalancers config-create 12345 --port 80 --protocol http
linode-cli nodebalancers config-view 12345 54321
linode-cli nodebalancers config-update 12345 54321 --algorithm leastconn
linode-cli nodebalancers config-delete 12345 54321

# Nodes
linode-cli nodebalancers nodes-list 12345 54321
linode-cli nodebalancers node-create 12345 54321 --address 192.168.1.1:80 --label web1
linode-cli nodebalancers node-update 12345 54321 67890 --weight 50
linode-cli nodebalancers node-delete 12345 54321 67890

# Statistics
linode-cli nodebalancers stats 12345
```

## Best Practices

```
1. High Availability
   +-- Use multiple backend nodes
   +-- Spread nodes across Linodes
   +-- Configure proper health checks
   +-- Set appropriate check intervals

2. Performance
   +-- Use private IPs for backends
   +-- Enable HTTP Keep-Alive on backends
   +-- Configure appropriate throttling
   +-- Use connection pooling

3. Security
   +-- Terminate SSL at NodeBalancer
   +-- Use strong TLS configuration
   +-- Restrict backend access with Firewall
   +-- Enable Proxy Protocol for real IPs

4. Maintenance
   +-- Use drain mode for graceful removal
   +-- Monitor NodeBalancer stats
   +-- Keep certificates updated
```

## Gotchas

- NodeBalancers have single public IP (no IPv6 balancing)
- Maximum 10,000 concurrent connections per NodeBalancer
- Health check failures don't alert automatically
- SSL certificates must be uploaded (no auto-renewal)
- Backend nodes must be in same region
- No WebSocket support in HTTP mode (use TCP)
- Config changes may briefly interrupt connections
- Cannot load balance UDP traffic

## Limits

| Resource | Limit |
|----------|-------|
| NodeBalancers per account | 20 |
| Configs per NodeBalancer | 20 |
| Nodes per Config | 200 |
| Concurrent connections | 10,000 |
| Throttle range | 0-20 connections/second/IP |
| SSL certificate size | 16 KB |
| SSL key size | 8 KB |
