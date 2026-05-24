# Akamai EdgeWorkers & EdgeKV

> Serverless computing at the edge using Akamai's global network.

## Overview

Akamai EdgeWorkers allows you to run JavaScript functions at the edge, close to users. Combined with EdgeKV (key-value storage), you can build low-latency serverless applications distributed globally.

## Key Concepts

| Term | Definition |
|------|------------|
| EdgeWorker | JavaScript function running at edge |
| EdgeKV | Distributed key-value store |
| Code Bundle | Packaged EdgeWorker code |
| Activation | Deploying to edge network |
| Event Handler | Trigger point in request lifecycle |
| Property | Akamai configuration for delivery |

## Architecture

```
+---------------------------------------------------------------+
|                    Akamai Edge Network                        |
|                                                               |
|  User Request                                                 |
|       |                                                       |
|       v                                                       |
|  +-----------------------------------------------------------+|
|  |              Edge Server (PoP)                            ||
|  |                                                           ||
|  |  +-----------------------------------------------+        ||
|  |  |              EdgeWorker                       |        ||
|  |  |                                               |        ||
|  |  |  Event Handlers:                              |        ||
|  |  |  +-- onClientRequest()                        |        ||
|  |  |  +-- onOriginRequest()                        |        ||
|  |  |  +-- onOriginResponse()                       |        ||
|  |  |  +-- onClientResponse()                       |        ||
|  |  |  +-- responseProvider()                       |        ||
|  |  |                                               |        ||
|  |  |  +------------------+                         |        ||
|  |  |  | EdgeKV Storage   |                         |        ||
|  |  |  | (Global K-V)     |                         |        ||
|  |  |  +------------------+                         |        ||
|  |  +-----------------------------------------------+        ||
|  +-----------------------------------------------------------+|
|                    |                                          |
|                    v                                          |
|              Origin Server                                    |
+---------------------------------------------------------------+
```

## Event Handlers

| Handler | Description | Use Case |
|---------|-------------|----------|
| **onClientRequest** | When request arrives | Routing, auth |
| **onOriginRequest** | Before origin fetch | Header modification |
| **onOriginResponse** | After origin response | Transform response |
| **onClientResponse** | Before sending to client | Add headers |
| **responseProvider** | Generate response | API responses |

## EdgeWorker Example

### main.js

```javascript
import { logger } from 'log';
import { createResponse } from 'create-response';
import { EdgeKV } from './edgekv.js';

// Initialize EdgeKV
const edgeKv = new EdgeKV({ namespace: 'default', group: 'config' });

export async function onClientRequest(request) {
  const path = request.path;

  // A/B Testing
  const variant = Math.random() < 0.5 ? 'A' : 'B';
  request.setVariable('PMUSER_VARIANT', variant);

  // Geolocation-based routing
  const country = request.userLocation.country;
  if (country === 'CN') {
    request.route({ origin: 'china-origin' });
  }

  logger.log(`Request: ${path}, Country: ${country}, Variant: ${variant}`);
}

export async function responseProvider(request) {
  // Return custom response without hitting origin
  try {
    const key = request.path.split('/')[2];
    const value = await edgeKv.getText({ item: key });

    return createResponse(200, {
      'Content-Type': 'application/json'
    }, JSON.stringify({ key, value }));
  } catch (error) {
    return createResponse(404, {}, 'Not found');
  }
}

export function onClientResponse(request, response) {
  // Add security headers
  response.setHeader('X-Frame-Options', 'DENY');
  response.setHeader('X-Content-Type-Options', 'nosniff');
  response.setHeader('Strict-Transport-Security', 'max-age=31536000');
}
```

### bundle.json

```json
{
  "edgeworker-version": "1.0.0",
  "description": "My EdgeWorker",
  "main": "main.js",
  "dependencies": {}
}
```

## EdgeKV

### Setup

```javascript
import { EdgeKV } from './edgekv.js';

const edgeKv = new EdgeKV({
  namespace: 'default',
  group: 'products'
});
```

### Operations

```javascript
// Write
await edgeKv.putText({
  item: 'product-123',
  value: JSON.stringify({ name: 'Widget', price: 29.99 })
});

// Read
const text = await edgeKv.getText({ item: 'product-123' });
const product = JSON.parse(text);

// Delete
await edgeKv.delete({ item: 'product-123' });

// Read JSON directly
const data = await edgeKv.getJson({ item: 'config' });
```

### EdgeKV Structure

```
Namespace: default
+-- Group: config
|   +-- item: settings -> { "feature_flag": true }
|   +-- item: limits -> { "rate_limit": 100 }
|
+-- Group: users
|   +-- item: user-123 -> { "name": "John", "tier": "premium" }
|   +-- item: user-456 -> { "name": "Jane", "tier": "basic" }
```

## Common Use Cases

### A/B Testing

```javascript
export function onClientRequest(request) {
  const bucket = hashUserId(request.getHeader('Cookie')) % 100;

  if (bucket < 50) {
    request.setVariable('PMUSER_EXPERIMENT', 'control');
    request.route({ origin: 'control-origin' });
  } else {
    request.setVariable('PMUSER_EXPERIMENT', 'treatment');
    request.route({ origin: 'treatment-origin' });
  }
}
```

### Geolocation Routing

```javascript
export function onClientRequest(request) {
  const country = request.userLocation.country;
  const continent = request.userLocation.continent;

  const origins = {
    'NA': 'us-origin',
    'EU': 'eu-origin',
    'AS': 'ap-origin'
  };

  request.route({ origin: origins[continent] || 'default-origin' });
}
```

### API Gateway

```javascript
import { createResponse } from 'create-response';
import { httpRequest } from 'http-request';

export async function responseProvider(request) {
  // Rate limiting with EdgeKV
  const clientIP = request.clientIp;
  const requests = await edgeKv.getJson({ item: `rate-${clientIP}` }) || { count: 0 };

  if (requests.count > 100) {
    return createResponse(429, {}, 'Rate limit exceeded');
  }

  // Authentication
  const token = request.getHeader('Authorization');
  if (!validateToken(token)) {
    return createResponse(401, {}, 'Unauthorized');
  }

  // Proxy to backend
  const backendResponse = await httpRequest('/api' + request.path, {
    method: request.method,
    headers: request.getHeaders()
  });

  return createResponse(
    backendResponse.status,
    backendResponse.headers,
    backendResponse.body
  );
}
```

### Response Transformation

```javascript
import { TextEncoderStream, TextDecoderStream } from 'streams';
import { createResponse } from 'create-response';

export async function onOriginResponse(request, response) {
  if (response.getHeader('Content-Type').includes('application/json')) {
    const body = await response.json();

    // Remove sensitive fields
    delete body.internal_id;
    delete body.debug_info;

    // Add computed fields
    body.processed_at = Date.now();

    return createResponse(
      response.status,
      response.getHeaders(),
      JSON.stringify(body)
    );
  }
}
```

## Deploy EdgeWorker

### CLI (Akamai CLI)

```bash
# Install Akamai CLI EdgeWorker package
akamai install edgeworkers

# Create EdgeWorker
akamai edgeworkers create-id "my-edgeworker" 12345

# Create code bundle
tar -czvf edgeworker.tgz main.js bundle.json

# Upload version
akamai edgeworkers upload --edgeworkerId 12345 --bundle edgeworker.tgz

# Activate on staging
akamai edgeworkers activate 12345 1 staging

# Activate on production
akamai edgeworkers activate 12345 1 production
```

### EdgeKV CLI

```bash
# Initialize EdgeKV
akamai edgekv init

# Create namespace
akamai edgekv create ns production

# Write item
akamai edgekv write production config settings '{"key": "value"}'

# Read item
akamai edgekv read production config settings

# List items
akamai edgekv list production config
```

## Limits

### EdgeWorkers

| Resource | Limit |
|----------|-------|
| Memory | 512 KB |
| Execution time | 50ms (event handlers) |
| Response time | 4s (responseProvider) |
| Code bundle size | 1 MB |
| HTTP sub-requests | 4 |
| Concurrent executions | Unlimited |

### EdgeKV

| Resource | Limit |
|----------|-------|
| Item size | 1 MB |
| Key length | 512 bytes |
| Namespaces per account | 100 |
| Groups per namespace | 1000 |
| Items per group | Unlimited |
| Read latency | ~1ms (cached) |
| Write propagation | Eventually consistent |

## Best Practices

```
1. Performance
   +-- Keep code small and fast
   +-- Use EdgeKV for caching
   +-- Minimize sub-requests
   +-- Avoid synchronous operations

2. Error Handling
   +-- Always catch exceptions
   +-- Return graceful fallbacks
   +-- Log errors for debugging

3. Security
   +-- Validate all input
   +-- Don't expose secrets in code
   +-- Use EdgeKV for dynamic config
   +-- Implement rate limiting

4. Testing
   +-- Test in staging first
   +-- Use Akamai Sandbox
   +-- Monitor after deployment
```

## Gotchas

- No Node.js APIs (subset of Web APIs)
- No file system access
- Limited crypto APIs
- EdgeKV is eventually consistent
- Cold starts can add latency
- Sub-request limitations
- Cannot modify request body in onClientRequest
- responseProvider skips origin entirely

## Pricing

| Component | Cost |
|-----------|------|
| EdgeWorkers | Per-invocation + compute time |
| EdgeKV | Per-operation + storage |
| Varies by contract tier | Contact Akamai |
