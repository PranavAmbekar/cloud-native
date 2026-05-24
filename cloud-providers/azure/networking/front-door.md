# Azure Front Door

> Global, scalable entry point for fast delivery of web applications with intelligent traffic routing.

## Overview

Azure Front Door is a global, scalable entry point that uses Microsoft's global edge network to create fast, secure, and widely scalable web applications. It provides Layer 7 load balancing, SSL offloading, caching, WAF, and intelligent traffic routing.

## Key Concepts

| Term | Definition |
|------|------------|
| Front Door Profile | Container for configuration |
| Endpoint | Global hostname (*.z01.azurefd.net) |
| Origin | Backend server/application |
| Origin Group | Collection of origins with load balancing |
| Route | Maps endpoint path to origin group |
| Rule Set | Custom processing rules for requests |
| WAF Policy | Web Application Firewall rules |

## Architecture

```
                            Global Edge Network
                           (150+ Edge Locations)
                                    |
Users ------------------------------v------------------------------------
        |                    +--------------+                     |
   (Europe)                  |  Front Door  |               (Asia)
        |                    |   (Anycast)  |                     |
        +------------------->|              |<--------------------+
                             +------+-------+
                                    |
               +--------------------+--------------------+
               |                    |                    |
               v                    v                    v
        +--------------+   +--------------+   +--------------+
        | Origin Group |   | Origin Group |   | Origin Group |
        |   (US East)  |   |  (US West)   |   |   (Europe)   |
        |  App Service |   |  App Service |   |    VM Pool   |
        +--------------+   +--------------+   +--------------+
```

## Tiers

| Tier | Features | Use Case |
|------|----------|----------|
| **Standard** | CDN, global LB, SSL, caching | Content delivery, basic routing |
| **Premium** | Standard + WAF, Private Link, Bot protection | Enterprise, security-focused |

## Traffic Routing Methods

| Method | Description |
|--------|-------------|
| **Latency** | Route to lowest latency origin (default) |
| **Priority** | Primary/secondary failover |
| **Weighted** | Distribute by percentage |
| **Session Affinity** | Stick to same origin |

### Latency-Based Routing

```
User in Tokyo
    |
    v (measures latency)
Front Door Edge
    |
    +-- Origin East US: 150ms
    +-- Origin West Europe: 200ms
    +-- Origin Japan: 20ms <- Selected (lowest)
```

### Priority-Based Routing

```
Origin Group:
+-- Priority 1: East US (Active)
|   +-- If healthy -> Route here
+-- Priority 2: West US (Standby)
|   +-- If Priority 1 down -> Route here
+-- Priority 3: Europe (DR)
    +-- Last resort
```

### Weighted Distribution

```
Origin Group (100% total):
+-- East US: 70%
+-- West US: 20%
+-- Europe: 10%
```

## Health Probes

```json
{
    "path": "/health",
    "protocol": "HTTPS",
    "intervalInSeconds": 30,
    "probeMethod": "HEAD",
    "enabledState": "Enabled"
}
```

| Setting | Options |
|---------|---------|
| Protocol | HTTP, HTTPS |
| Method | GET, HEAD |
| Path | Custom health endpoint |
| Interval | 5-255 seconds |

## Caching

### Cache Behavior

| Setting | Description |
|---------|-------------|
| **Honor origin** | Use origin Cache-Control headers |
| **Override always** | Always cache with specified TTL |
| **Override if origin missing** | Cache only if origin doesn't specify |

### Query String Handling

| Mode | Behavior |
|------|----------|
| **Ignore** | Same cache regardless of query |
| **Include** | Cache varies by query string |
| **Exclude specified** | Ignore certain query params |
| **Include specified** | Only consider certain params |

### Cache Key

```
Cache Key = URL + Query Params (based on setting) + Vary headers

Example:
https://example.com/api/users?page=1&sort=name
- Ignore query: Cache key = /api/users
- Include query: Cache key = /api/users?page=1&sort=name
```

## Rules Engine

Custom request/response processing.

### Conditions

| Condition | Description |
|-----------|-------------|
| Request URL | Path, query string, full URL |
| Request Method | GET, POST, etc. |
| Request Header | Header name/value |
| Client IP | Source IP address |
| Device | Mobile, Desktop, etc. |
| Geographic | Country, region |

### Actions

| Action | Description |
|--------|-------------|
| URL Rewrite | Change request path |
| URL Redirect | Send redirect response |
| Modify Request Header | Add/remove/modify headers |
| Modify Response Header | Add/remove/modify headers |
| Override Origin Group | Route to different origin |
| Override Cache | Custom caching behavior |

### Example: Redirect HTTP to HTTPS

```
Condition:
  - Request Protocol = HTTP

Action:
  - URL Redirect
  - Type: Permanent (301)
  - Protocol: HTTPS
```

### Example: Path-based Routing

```
Rule 1:
  Condition: Path starts with /api
  Action: Route to API Origin Group

Rule 2:
  Condition: Path starts with /images
  Action: Route to CDN Origin Group
```

## Web Application Firewall (WAF)

Premium tier only.

### Rule Types

| Type | Description |
|------|-------------|
| **Managed Rules** | OWASP, Microsoft rules |
| **Custom Rules** | Your own rules |
| **Bot Protection** | Good/bad bot detection |

### Managed Rule Sets

- **DRS (Default Rule Set)**: Microsoft-managed OWASP rules
- **Bot Manager**: Bot detection and mitigation

### WAF Modes

| Mode | Behavior |
|------|----------|
| **Detection** | Log only, don't block |
| **Prevention** | Block matching requests |

### Custom Rule Example

```json
{
    "name": "BlockSpecificCountry",
    "priority": 1,
    "ruleType": "MatchRule",
    "matchConditions": [{
        "matchVariable": "RemoteAddr",
        "operator": "GeoMatch",
        "matchValue": ["XX"]
    }],
    "action": "Block"
}
```

## Private Link Origins

Connect to private backends (Premium tier).

```
Front Door --------------> Private Endpoint --> Internal App Service
                              (Private Link)
            (No public endpoint needed)
```

## SSL/TLS

| Feature | Description |
|---------|-------------|
| Managed certificates | Free, auto-renewed |
| Custom certificates | BYOK from Key Vault |
| End-to-end encryption | HTTPS to origins |
| TLS version | Configurable min version |

## CLI Quick Reference

```bash
# Create Front Door profile
az afd profile create \
  --profile-name myFrontDoor \
  --resource-group myRG \
  --sku Standard_AzureFrontDoor

# Create endpoint
az afd endpoint create \
  --endpoint-name myEndpoint \
  --profile-name myFrontDoor \
  --resource-group myRG

# Create origin group
az afd origin-group create \
  --origin-group-name myOriginGroup \
  --profile-name myFrontDoor \
  --resource-group myRG \
  --probe-path "/health" \
  --probe-protocol Https \
  --probe-interval-in-seconds 30

# Add origin
az afd origin create \
  --origin-name myOrigin \
  --origin-group-name myOriginGroup \
  --profile-name myFrontDoor \
  --resource-group myRG \
  --host-name myapp.azurewebsites.net \
  --origin-host-header myapp.azurewebsites.net \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 100

# Create route
az afd route create \
  --route-name myRoute \
  --endpoint-name myEndpoint \
  --profile-name myFrontDoor \
  --resource-group myRG \
  --origin-group myOriginGroup \
  --supported-protocols Https \
  --https-redirect Enabled \
  --patterns-to-match "/*"

# Create WAF policy (Premium)
az network front-door waf-policy create \
  --name myWAFPolicy \
  --resource-group myRG \
  --sku Premium_AzureFrontDoor
```

## Comparison: Front Door vs CDN vs Traffic Manager

| Feature | Front Door | CDN | Traffic Manager |
|---------|------------|-----|-----------------|
| Layer | 7 (HTTP/S) | 7 (HTTP/S) | DNS |
| Global LB | Yes | No | Yes (DNS) |
| Caching | Yes | Yes (primary purpose) | No |
| SSL termination | Yes | Yes | No |
| WAF | Yes (Premium) | Yes | No |
| Real-time failover | Yes | No | Minutes (DNS TTL) |
| Session affinity | Yes | No | No |
| URL-based routing | Yes | No | No |

## Exam Tips (AZ-104, AZ-305)

1. **Standard vs Premium**: Premium adds WAF, Private Link, Bot protection
2. **Latency routing**: Default, routes to lowest latency origin
3. **Health probes**: Required for failover to work
4. **Rules Engine**: Process requests before routing
5. **Caching**: Edge caching reduces origin load
6. **WAF**: Only in Premium tier
7. **Anycast**: Single IP, routed to nearest edge
8. **Origin types**: App Service, Storage, VMs, external endpoints
9. **Managed certificates**: Free SSL for custom domains
10. **Private Link**: Access private backends without public exposure

## Gotchas

- Standard tier doesn't include WAF (need Premium)
- Caching rules apply before routing rules
- Health probe path must return 200 for origin to be healthy
- Custom domains require CNAME validation
- Managed certificates take time to provision
- Private Link origins only in Premium
- Query string caching defaults to ignore (may cause stale content)
- Rules Engine changes can take 10+ minutes to propagate
- WAF managed rules can have false positives
- Origin response must include proper CORS headers for browser requests

## Limits

| Resource | Limit |
|----------|-------|
| Profiles per subscription | 500 |
| Endpoints per profile | 500 |
| Origin groups per profile | 500 |
| Origins per origin group | 100 |
| Routes per endpoint | 500 |
| Rule sets per profile | 500 |
| Rules per rule set | 100 |
| Custom domains per endpoint | 500 |
| Managed certificates | 500 per profile |
