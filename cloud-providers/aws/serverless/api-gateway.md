# Amazon API Gateway

> Fully managed service to create, publish, and manage APIs at any scale.

---

## API Types

| Type | Protocol | Use Case |
|------|----------|----------|
| REST API | HTTP/HTTPS | Full-featured REST APIs |
| HTTP API | HTTP/HTTPS | Simple, low-latency, cheaper |
| WebSocket API | WebSocket | Real-time, bidirectional |

### REST vs HTTP API

| Feature | REST API | HTTP API |
|---------|----------|----------|
| Cost | $3.50/million | $1.00/million |
| Latency | Higher | ~60% lower |
| Features | Full | Basic |
| Caching | Yes | No |
| Request validation | Yes | No |
| WAF integration | Yes | No |
| Private integrations | Yes | Yes |
| API keys/Usage plans | Yes | No |

---

## Architecture

```
+------------------------------------------------------------------+
|                          API Gateway                             |
|                                                                  |
|   +--------------+    +--------------+    +--------------+       |
|   |   Stage:     |    |   Stage:     |    |   Stage:     |       |
|   |    dev       |    |   staging    |    |    prod      |       |
|   +--------------+    +--------------+    +--------------+       |
|          |                   |                   |               |
|          +-------------------+-------------------+               |
|                              |                                   |
|   +--------------------------------------------------------+    |
|   |                       Resources                        |    |
|   |   /users                                               |    |
|   |   +-- GET  -> Lambda: list-users                       |    |
|   |   +-- POST -> Lambda: create-user                      |    |
|   |   +-- /{id}                                            |    |
|   |       +-- GET    -> Lambda: get-user                   |    |
|   |       +-- PUT    -> Lambda: update-user                |    |
|   |       +-- DELETE -> Lambda: delete-user                |    |
|   +--------------------------------------------------------+    |
+------------------------------------------------------------------+
```

---

## Integration Types

| Type | Description |
|------|-------------|
| Lambda | Invoke Lambda function |
| Lambda Proxy | Pass full request to Lambda |
| HTTP | Forward to HTTP endpoint |
| HTTP Proxy | Pass full request to HTTP endpoint |
| AWS Service | Direct AWS service integration |
| Mock | Return static response |
| VPC Link | Access private resources |

### Lambda Proxy Integration
```
Request -> API Gateway -> Lambda (receives full event)
                               |
                    {
                      "httpMethod": "GET",
                      "path": "/users/123",
                      "headers": {...},
                      "queryStringParameters": {...},
                      "pathParameters": {"id": "123"},
                      "body": "...",
                      "requestContext": {...}
                    }
```

### Lambda Response Format
```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"message\": \"Hello\"}"
}
```

---

## Stages & Deployment

```
API Definition
     |
     v
Deployment
     |
     +-- Stage: dev     -> https://xxx.execute-api.region.amazonaws.com/dev
     +-- Stage: staging -> https://xxx.execute-api.region.amazonaws.com/staging
     +-- Stage: prod    -> https://xxx.execute-api.region.amazonaws.com/prod
```

### Stage Variables
```
${stageVariables.lambdaAlias}
${stageVariables.tableName}
```

Use for:
- Lambda aliases
- HTTP endpoints
- Configuration values

### Canary Deployments
```
prod stage:
+-- 90% -> Current deployment
+-- 10% -> Canary deployment

Promote after testing
```

---

## Authentication & Authorization

### IAM
```
Client -> SigV4 signed request -> API Gateway -> Validate IAM
```

### Lambda Authorizer (Custom)
```
Client -> Token -> API Gateway -> Lambda Authorizer -> Policy
                                                          |
                                                    Allow/Deny
```

**Token-based:**
```javascript
exports.handler = async (event) => {
  const token = event.authorizationToken;
  // Validate token
  return {
    principalId: 'user123',
    policyDocument: {
      Statement: [{
        Action: 'execute-api:Invoke',
        Effect: 'Allow',
        Resource: event.methodArn
      }]
    }
  };
};
```

**Request-based:**
- Access headers, query params, path params
- More flexible validation

### Cognito User Pools
```
Client -> JWT token -> API Gateway -> Cognito -> Validate
```

### API Keys
```
Client -> x-api-key header -> API Gateway -> Validate key
```

For:
- Rate limiting
- Usage tracking
- NOT for authentication

---

## Throttling & Quotas

### Account Limits
- 10,000 requests/second (soft limit)
- 5,000 concurrent requests

### Stage Limits
```
Stage: prod
+-- Rate limit: 1000 req/sec
+-- Burst limit: 2000 requests
```

### Method Limits
```
GET /users
+-- Rate limit: 100 req/sec
+-- Burst limit: 200 requests
```

### Usage Plans
```
Usage Plan: Basic
+-- API Keys: [key1, key2]
+-- Throttle: 100 req/sec
+-- Burst: 200
+-- Quota: 10,000 req/month
```

---

## Caching (REST API only)

```
Client -> API Gateway -> Cache HIT -> Return cached response
                            |
                       Cache MISS
                            |
                            v
                        Backend -> Cache response -> Return
```

- Cache size: 0.5 GB to 237 GB
- TTL: 0 to 3600 seconds (default 300)
- Per-stage configuration
- Cache key: Method + Resource path

### Cache Invalidation
```
Header: Cache-Control: max-age=0

OR

aws apigateway flush-stage-cache \
  --rest-api-id xxx \
  --stage-name prod
```

---

## Request/Response Transformation

### Mapping Templates (Velocity)
```velocity
#set($inputRoot = $input.path('$'))
{
  "userId": "$inputRoot.user_id",
  "userName": "$inputRoot.user_name",
  "timestamp": "$context.requestTime"
}
```

### Request Validation
```json
{
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name": {"type": "string"},
    "email": {"type": "string", "format": "email"}
  }
}
```

---

## CORS

```
Browser -> Preflight (OPTIONS) -> API Gateway
                                      |
                             Access-Control-Allow-Origin: *
                             Access-Control-Allow-Methods: GET, POST
                             Access-Control-Allow-Headers: Content-Type
```

Enable CORS:
1. Create OPTIONS method
2. Return appropriate headers
3. Or enable with one click in console

---

## WebSocket API

Bidirectional communication.

```
+------------------------------------------------------------------+
|                       WebSocket API                              |
|                                                                  |
|   $connect    -> Lambda: onConnect (store connectionId)          |
|   $disconnect -> Lambda: onDisconnect (remove connectionId)      |
|   $default    -> Lambda: onMessage (handle messages)             |
|   sendmessage -> Lambda: sendMessage (custom route)              |
|                                                                  |
+------------------------------------------------------------------+
```

### Send message to client
```python
import boto3

client = boto3.client('apigatewaymanagementapi',
    endpoint_url='https://xxx.execute-api.region.amazonaws.com/prod')

client.post_to_connection(
    ConnectionId='abc123',
    Data=b'{"message": "Hello"}'
)
```

---

## Private APIs

API accessible only from VPC.

```
+------------------------------------------+
|                   VPC                    |
|                                          |
|   +-----------+     +-----------------+  |
|   |    EC2    |---->|  VPC Endpoint   |  |
|   |           |     |   (Interface)   |  |
|   +-----------+     +--------+--------+  |
|                              |           |
+------------------------------|------------+
                               |
                               v
                     Private API Gateway
```

Resource policy required:
```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "execute-api:Invoke",
  "Resource": "arn:aws:execute-api:region:account:api-id/*",
  "Condition": {
    "StringEquals": {
      "aws:sourceVpce": "vpce-xxx"
    }
  }
}
```

---

## CLI Quick Reference

```bash
# Create REST API
aws apigateway create-rest-api --name my-api

# Get resources
aws apigateway get-resources --rest-api-id xxx

# Create resource
aws apigateway create-resource \
  --rest-api-id xxx \
  --parent-id yyy \
  --path-part users

# Create method
aws apigateway put-method \
  --rest-api-id xxx \
  --resource-id yyy \
  --http-method GET \
  --authorization-type NONE

# Create integration
aws apigateway put-integration \
  --rest-api-id xxx \
  --resource-id yyy \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:region:lambda:path/2015-03-31/functions/arn:aws:lambda:xxx/invocations

# Create deployment
aws apigateway create-deployment \
  --rest-api-id xxx \
  --stage-name prod

# Create HTTP API (simpler)
aws apigatewayv2 create-api \
  --name my-http-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:xxx
```

---

## Pricing

| API Type | Cost |
|----------|------|
| REST API | $3.50/million requests |
| HTTP API | $1.00/million requests |
| WebSocket | $1.00/million messages + $0.25/million connection minutes |
| Caching | $0.02-$3.80/hour by size |

---

## Exam Tips

1. **HTTP API** - cheaper, faster, simpler (no caching/WAF)
2. **REST API** - full features, higher cost
3. **Lambda Proxy** - full request to Lambda, format response
4. **Lambda Authorizer** - custom auth logic
5. **Cognito** - managed user authentication
6. **API Keys** - throttling, NOT authentication
7. **Usage Plans** - quotas and throttling per API key
8. **Stage Variables** - configuration per stage
9. **Caching** - REST API only, per-stage
10. **CORS** - enable for browser access
11. **Private API** - VPC endpoint required
12. **WebSocket** - $connect, $disconnect, $default routes
