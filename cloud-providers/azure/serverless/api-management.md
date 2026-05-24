# Azure API Management (APIM)

> Full lifecycle API management platform for publishing, securing, and analyzing APIs.

## Overview

Azure API Management (APIM) is a hybrid, multicloud management platform for APIs. It provides a gateway, developer portal, and management interface to publish, secure, transform, maintain, and monitor APIs.

## Key Concepts

| Term | Definition |
|------|------------|
| API Gateway | Entry point for API calls, enforces policies |
| Developer Portal | Self-service portal for API consumers |
| Management Plane | Azure portal interface for configuration |
| Product | Bundle of APIs with access policies |
| Subscription | Access key for API consumers |
| Policy | XML rules applied to requests/responses |
| Backend | The actual API service being proxied |
| Named Value | Reusable configuration values |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Azure API Management                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   API Gateway                          │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │  │
│  │  │ Inbound │→ │ Backend │→ │Outbound │→ │ On-Error│ │  │
│  │  │ Policy  │  │ Request │  │ Policy  │  │ Policy  │ │  │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│  ┌────────────────────────▼────────────────────────────┐   │
│  │                 Developer Portal                      │   │
│  │   • API Documentation  • Try APIs  • Subscribe       │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │     Backend APIs        │
              │  • Azure Functions      │
              │  • App Service          │
              │  • Logic Apps           │
              │  • External APIs        │
              └─────────────────────────┘
```

## Tiers

| Tier | Use Case | Gateway Instances | SLA |
|------|----------|-------------------|-----|
| **Consumption** | Serverless, low traffic | Auto-scaled | 99.95% |
| **Developer** | Dev/test, non-production | 1 | No SLA |
| **Basic** | Entry-level production | 2 | 99.95% |
| **Standard** | Medium traffic production | 4 | 99.95% |
| **Premium** | Enterprise, multi-region | 12+ per region | 99.99% |

### Tier Features

| Feature | Consumption | Developer | Basic | Standard | Premium |
|---------|-------------|-----------|-------|----------|---------|
| VNet integration | No | Yes | No | No | Yes |
| Multi-region | No | No | No | No | Yes |
| Availability Zones | No | No | No | No | Yes |
| Custom domains | 1 | 1 | 1 | 1 | Unlimited |
| Dev portal | Yes | Yes | Yes | Yes | Yes |
| Built-in cache | No | 10 MB | 50 MB | 1 GB | 5 GB+ |

## API Organization

```
APIM Instance
├── Products
│   ├── Free Tier (rate limited)
│   │   ├── Weather API
│   │   └── Maps API (read-only)
│   └── Premium (higher limits)
│       ├── Weather API (full)
│       ├── Maps API (full)
│       └── Analytics API
├── APIs
│   ├── Weather API
│   │   ├── GET /current
│   │   ├── GET /forecast
│   │   └── POST /alerts
│   └── Maps API
│       ├── GET /geocode
│       └── GET /directions
└── Subscriptions
    ├── Developer A → Free Tier
    └── Company B → Premium
```

## Policies

Policies are XML statements executed on request/response.

### Policy Structure

```xml
<policies>
    <inbound>
        <!-- Applied before request goes to backend -->
    </inbound>
    <backend>
        <!-- Applied to backend request -->
    </backend>
    <outbound>
        <!-- Applied after response from backend -->
    </outbound>
    <on-error>
        <!-- Applied when an error occurs -->
    </on-error>
</policies>
```

### Common Policies

```xml
<!-- Rate limiting -->
<rate-limit calls="100" renewal-period="60" />

<!-- Quota -->
<quota calls="10000" renewal-period="86400" />

<!-- IP filtering -->
<ip-filter action="allow">
    <address>192.168.1.0/24</address>
</ip-filter>

<!-- JWT validation -->
<validate-jwt header-name="Authorization" require-scheme="Bearer">
    <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
    <audiences>
        <audience>api://my-api</audience>
    </audiences>
</validate-jwt>

<!-- Transform request -->
<set-header name="X-Custom-Header" exists-action="override">
    <value>CustomValue</value>
</set-header>

<!-- Cache response -->
<cache-lookup vary-by-developer="false" vary-by-developer-groups="false" />
<cache-store duration="3600" />

<!-- Rewrite URL -->
<rewrite-uri template="/api/v2/{path}" />

<!-- CORS -->
<cors allow-credentials="true">
    <allowed-origins>
        <origin>https://myapp.com</origin>
    </allowed-origins>
    <allowed-methods>
        <method>GET</method>
        <method>POST</method>
    </allowed-methods>
</cors>

<!-- Mock response -->
<mock-response status-code="200" content-type="application/json" />

<!-- Set backend -->
<set-backend-service base-url="https://mybackend.com" />
```

### Policy Expressions

```xml
<!-- C# expressions in policies -->
<set-header name="X-Request-Id" exists-action="skip">
    <value>@(Guid.NewGuid().ToString())</value>
</set-header>

<!-- Conditional logic -->
<choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault("User-Agent","").Contains("Mobile"))">
        <set-backend-service base-url="https://mobile-api.com" />
    </when>
    <otherwise>
        <set-backend-service base-url="https://web-api.com" />
    </otherwise>
</choose>

<!-- Access context variables -->
<set-variable name="userId" value="@(context.Request.Headers.GetValueOrDefault("X-User-Id","anonymous"))" />
```

## Authentication Methods

| Method | Description |
|--------|-------------|
| **Subscription Key** | API key in header or query string |
| **OAuth 2.0** | Token validation (Azure AD, etc.) |
| **Client Certificate** | Mutual TLS authentication |
| **Basic Auth** | Username/password (policy-based) |
| **Managed Identity** | Backend auth to Azure services |

### Subscription Key

```bash
# Header (recommended)
Ocp-Apim-Subscription-Key: <key>

# Query string
?subscription-key=<key>
```

## Versioning and Revisions

### Versions

Different API versions coexist (v1, v2, etc.)

| Scheme | Example |
|--------|---------|
| Path | /api/v1/users, /api/v2/users |
| Header | api-version: 1.0 |
| Query | ?api-version=1.0 |

### Revisions

Non-breaking changes within a version.

```
API: Weather v1
├── Revision 1 (original)
├── Revision 2 (bug fix)
└── Revision 3 (current) ← Live

Test revisions via: ;rev=2 suffix
```

## Developer Portal

Self-service portal features:
- API documentation (auto-generated from OpenAPI)
- Interactive API console (try APIs)
- Subscription management
- Analytics and usage
- Custom branding and content

```bash
# Developer portal URL
https://{apim-name}.developer.azure-api.net
```

## Integration Patterns

### Azure Functions Backend

```xml
<policies>
    <inbound>
        <set-backend-service id="function-backend"
            backend-id="azure-functions-backend" />
    </inbound>
</policies>
```

### Logic Apps Backend

```xml
<policies>
    <inbound>
        <set-backend-service
            base-url="https://prod-xx.region.logic.azure.com/workflows/{workflowId}/triggers/manual/paths/invoke" />
        <set-header name="Content-Type" exists-action="override">
            <value>application/json</value>
        </set-header>
    </inbound>
</policies>
```

## Monitoring and Analytics

### Built-in Analytics

- Request/response metrics
- Error rates and types
- Latency percentiles
- Geographic distribution
- Top APIs and operations

### Application Insights Integration

```xml
<!-- Log to Application Insights -->
<policies>
    <inbound>
        <trace source="APIM" severity="information">
            <message>@($"Incoming request for {context.Operation.Name}")</message>
        </trace>
    </inbound>
</policies>
```

## CLI Quick Reference

```bash
# Create APIM instance
az apim create \
  --name myapim \
  --resource-group myRG \
  --publisher-name "My Company" \
  --publisher-email admin@mycompany.com \
  --sku-name Consumption

# Import API from OpenAPI
az apim api import \
  --resource-group myRG \
  --service-name myapim \
  --api-id my-api \
  --path /api \
  --specification-format OpenApi \
  --specification-url https://example.com/openapi.json

# Create product
az apim product create \
  --resource-group myRG \
  --service-name myapim \
  --product-id free-tier \
  --display-name "Free Tier" \
  --subscription-required true

# Create subscription
az apim subscription create \
  --resource-group myRG \
  --service-name myapim \
  --subscription-id sub1 \
  --display-name "Developer Subscription" \
  --scope /products/free-tier

# Apply policy
az apim api operation policy create \
  --resource-group myRG \
  --service-name myapim \
  --api-id my-api \
  --operation-id get-users \
  --xml-policy @policy.xml
```

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **Consumption tier**: Serverless, auto-scale, no VNet, per-call billing
2. **Premium tier**: Required for VNet integration, multi-region, availability zones
3. **Policies**: Inbound → Backend → Outbound → On-Error execution order
4. **Rate-limit vs Quota**: Rate-limit = per time window, Quota = total calls
5. **Subscription key**: Header preferred over query string (security)
6. **Revisions**: Non-breaking changes; Versions: Breaking changes
7. **Developer portal**: Customizable, auto-generates docs from OpenAPI
8. **Named values**: Store secrets, use Key Vault references
9. **Backend**: Can be Function, App Service, Logic App, external URL
10. **Self-hosted gateway**: Run gateway on-premises or in other clouds

## Gotchas

- Consumption tier doesn't support VNet integration or built-in cache
- Developer tier has no SLA (don't use for production)
- Subscription keys rotate but existing keys continue to work
- Policy expressions use C# syntax (not JavaScript)
- Rate limits are approximate (eventual consistency)
- Deploying a new APIM instance takes 30-45 minutes
- Premium tier required for availability zones
- Maximum request/response body size is 256 KB (can increase to 4 MB)
- Self-hosted gateway requires Premium tier

## Limits

| Resource | Limit |
|----------|-------|
| APIs per instance | Unlimited |
| Operations per API | 1000 |
| Policies size | 256 KB |
| Request/response body | 256 KB (configurable to 4 MB) |
| Subscriptions per product | Unlimited |
| Cache size | Varies by tier (10 MB - 5 GB) |
| Backend timeout | 240 seconds |
| Request headers | 32 KB total |
