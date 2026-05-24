# Google Cloud Load Balancing

> Fully distributed, software-defined managed service for load balancing traffic to your applications.

## Overview

Cloud Load Balancing is a fully distributed, software-defined managed service. It's not instance or device-based, so you don't need to manage physical load balancing infrastructure.

## Key Concepts

| Term | Definition |
|------|------------|
| Backend Service | Group of backends that receive traffic |
| Backend | Instance group, NEG, or Cloud Storage bucket |
| Health Check | Verifies backend availability |
| Frontend | IP address, port, and protocol facing users |
| URL Map | Routes requests to backend services |
| Target Proxy | Routes traffic based on URL map |

## Load Balancer Types

| Type | Layer | Scope | Use Case |
|------|-------|-------|----------|
| **Global External HTTP(S)** | 7 | Global | Web applications |
| **Global External TCP/SSL Proxy** | 4 | Global | Non-HTTP TCP traffic |
| **Regional External HTTP(S)** | 7 | Regional | Regional web apps |
| **Regional External TCP/UDP** | 4 | Regional | Network load balancing |
| **Internal HTTP(S)** | 7 | Regional | Internal microservices |
| **Internal TCP/UDP** | 4 | Regional | Internal services |
| **Cross-region Internal** | 7 | Global | Multi-region internal |

## Architecture

### Global HTTP(S) Load Balancer

```
                              Global Anycast IP
                                     |
                              +------v------+
                              |   Google    |
                              |   Edge      |
                              |   Network   |
                              +------+------+
                                     |
              +----------------------+----------------------+
              |                      |                      |
       +------v------+        +------v------+       +------v------+
       |   Target    |        |   URL Map   |       |   Target    |
       |   Proxy     |<-------|   (Routes)  |------>|   Proxy     |
       +------+------+        +-------------+       +------+------+
              |                                            |
       +------v--------------------------------------------v------+
       |                    Backend Services                      |
       |                                                          |
       |   +-------------+    +-------------+    +---------+      |
       |   |us-central1  |    |europe-west1 |    | Cloud   |      |
       |   | MIG         |    | MIG         |    | Storage |      |
       |   +-------------+    +-------------+    +---------+      |
       +----------------------------------------------------------+
```

### Network Load Balancer (TCP/UDP)

```
                         Regional External IP
                                  |
                           +------v------+
                           |  Forwarding |
                           |    Rule     |
                           +------+------+
                                  |
                           +------v------+
                           |   Backend   |
                           |   Service   |
                           +------+------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
       +------v------+     +------v------+     +------v------+
       |    VM 1     |     |    VM 2     |     |    VM 3     |
       |  10.0.1.2   |     |  10.0.1.3   |     |  10.0.1.4   |
       +-------------+     +-------------+     +-------------+
```

## Global HTTP(S) Load Balancer

### Create Load Balancer

```bash
# 1. Create instance template
gcloud compute instance-templates create web-template \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
    apt-get update && apt-get install -y nginx
    echo "Hello from $(hostname)" > /var/www/html/index.html'

# 2. Create managed instance group
gcloud compute instance-groups managed create web-mig \
  --template=web-template \
  --size=2 \
  --zone=us-central1-a

# 3. Create health check
gcloud compute health-checks create http http-health-check \
  --port=80 \
  --request-path=/

# 4. Create backend service
gcloud compute backend-services create web-backend \
  --protocol=HTTP \
  --health-checks=http-health-check \
  --global

# 5. Add backend
gcloud compute backend-services add-backend web-backend \
  --instance-group=web-mig \
  --instance-group-zone=us-central1-a \
  --global

# 6. Create URL map
gcloud compute url-maps create web-map \
  --default-service=web-backend

# 7. Create target proxy
gcloud compute target-http-proxies create http-lb-proxy \
  --url-map=web-map

# 8. Create forwarding rule
gcloud compute forwarding-rules create http-lb-rule \
  --global \
  --target-http-proxy=http-lb-proxy \
  --ports=80
```

### URL Map (Path-Based Routing)

```yaml
# url-map.yaml
name: web-map
defaultService: projects/my-project/global/backendServices/default-backend
hostRules:
- hosts:
  - "api.example.com"
  pathMatcher: api-paths
- hosts:
  - "www.example.com"
  pathMatcher: www-paths
pathMatchers:
- name: api-paths
  defaultService: projects/my-project/global/backendServices/api-backend
  pathRules:
  - paths:
    - /v1/*
    service: projects/my-project/global/backendServices/api-v1-backend
  - paths:
    - /v2/*
    service: projects/my-project/global/backendServices/api-v2-backend
- name: www-paths
  defaultService: projects/my-project/global/backendServices/www-backend
  pathRules:
  - paths:
    - /images/*
    service: projects/my-project/global/backendServices/images-backend
```

## Health Checks

### Types

| Protocol | Use Case |
|----------|----------|
| HTTP | Web applications |
| HTTPS | Secure web apps |
| HTTP/2 | gRPC services |
| TCP | Any TCP service |
| SSL | SSL/TLS services |

### Configuration

```bash
gcloud compute health-checks create http my-health-check \
  --port=80 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3
```

## Backend Types

| Type | Description |
|------|-------------|
| Instance Groups | VMs (managed or unmanaged) |
| Network Endpoint Groups (NEGs) | Individual endpoints |
| Cloud Storage Buckets | Static content |
| Serverless NEGs | Cloud Run, Functions, App Engine |

### Serverless NEG

```bash
# Create serverless NEG for Cloud Run
gcloud compute network-endpoint-groups create my-neg \
  --network-endpoint-type=serverless \
  --cloud-run-service=my-service \
  --region=us-central1

# Add to backend service
gcloud compute backend-services add-backend my-backend \
  --network-endpoint-group=my-neg \
  --network-endpoint-group-region=us-central1 \
  --global
```

## Cloud CDN

Enable caching at edge locations.

```bash
# Enable Cloud CDN on backend service
gcloud compute backend-services update my-backend \
  --enable-cdn \
  --global

# Configure cache settings
gcloud compute backend-services update my-backend \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600 \
  --max-ttl=86400 \
  --global
```

### Cache Modes

| Mode | Description |
|------|-------------|
| CACHE_ALL_STATIC | Cache static content automatically |
| USE_ORIGIN_HEADERS | Honor Cache-Control headers |
| FORCE_CACHE_ALL | Cache everything (with TTL) |

## Cloud Armor

DDoS protection and WAF.

```bash
# Create security policy
gcloud compute security-policies create my-policy \
  --description="My security policy"

# Add rule to block IP
gcloud compute security-policies rules create 1000 \
  --security-policy=my-policy \
  --src-ip-ranges=1.2.3.4/32 \
  --action=deny-403

# Add rate limiting
gcloud compute security-policies rules create 2000 \
  --security-policy=my-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=300

# Attach to backend service
gcloud compute backend-services update my-backend \
  --security-policy=my-policy \
  --global
```

## SSL/TLS

### Managed Certificates

```bash
# Create managed certificate
gcloud compute ssl-certificates create my-cert \
  --domains=example.com,www.example.com \
  --global

# Create HTTPS target proxy
gcloud compute target-https-proxies create https-proxy \
  --url-map=web-map \
  --ssl-certificates=my-cert
```

## Session Affinity

| Type | Description |
|------|-------------|
| NONE | No affinity (default) |
| CLIENT_IP | Based on client IP |
| CLIENT_IP_PROTO | IP + protocol |
| CLIENT_IP_PORT_PROTO | IP + port + protocol |
| GENERATED_COOKIE | Cookie-based |
| HEADER_FIELD | Custom header |
| HTTP_COOKIE | Existing cookie |

```bash
gcloud compute backend-services update my-backend \
  --session-affinity=GENERATED_COOKIE \
  --affinity-cookie-ttl=3600 \
  --global
```

## CLI Quick Reference

```bash
# List load balancers (forwarding rules)
gcloud compute forwarding-rules list

# Describe load balancer
gcloud compute forwarding-rules describe my-rule --global

# List backend services
gcloud compute backend-services list

# Get backend health
gcloud compute backend-services get-health my-backend --global

# Update backend capacity
gcloud compute backend-services update-backend my-backend \
  --instance-group=my-mig \
  --instance-group-zone=us-central1-a \
  --balancing-mode=RATE \
  --max-rate-per-instance=100 \
  --global

# Delete load balancer components
gcloud compute forwarding-rules delete my-rule --global
gcloud compute target-http-proxies delete my-proxy
gcloud compute url-maps delete my-map
gcloud compute backend-services delete my-backend --global
```

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Global HTTP(S)**: Layer 7, anycast IP, SSL termination
2. **Regional Network**: Layer 4, preserves client IP
3. **URL Map**: Path-based routing for HTTP(S)
4. **Cloud CDN**: Enable on backend service
5. **Cloud Armor**: WAF and DDoS protection
6. **Health checks**: Different types for different protocols
7. **NEGs**: Use for Cloud Run, Functions
8. **Managed certs**: Auto-renewing SSL certificates
9. **Session affinity**: For stateful applications
10. **Premium vs Standard tier**: Affects routing and features

## Gotchas

- Global LB requires premium network tier
- URL maps only for HTTP(S) load balancers
- Health checks must match backend protocol
- Managed certificates require DNS verification
- Cloud Armor only on global HTTP(S) LB
- Session affinity affects caching
- Forwarding rules have different scopes (global/regional)
- Instance groups must be named ports for HTTP(S)
- Backend service protocol must match target proxy
- Autoscaling based on LB utilization is separate config

## Limits

| Resource | Limit |
|----------|-------|
| Forwarding rules per project | 50 |
| Backend services per project | 50 |
| Backends per backend service | 150 |
| URL maps per project | 50 |
| Path rules per URL map | 100 |
| Health checks per project | 100 |
| SSL certificates per project | 100 |
| Target proxies per project | 50 |
| Security policies per project | 10 |
