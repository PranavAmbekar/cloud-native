# Amazon CloudFront

> Global Content Delivery Network (CDN) for fast, secure delivery of data, videos, applications.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Distribution | CloudFront configuration |
| Origin | Source of content (S3, ALB, custom) |
| Edge Location | Point of Presence (PoP) for caching |
| Regional Edge Cache | Larger cache between edge and origin |
| Behavior | URL pattern matching and settings |
| TTL | Time content stays cached |
| Invalidation | Force removal of cached content |

---

## Architecture

```
+-----------------------------------------------------------------+
|                                                                 |
|   User --> Edge Location --> Regional Edge --> Origin           |
|            (closest)         Cache              (S3/ALB)        |
|                 |                                   |           |
|                 |         Cache HIT                 |           |
|                 +--------------------------------> Response     |
|                                                                 |
|                              Cache MISS                         |
|                 +------------------------------------>|         |
|                                                       |         |
|                 <-------------------------------------+         |
|                        Fetch from origin                        |
|                                                                 |
+-----------------------------------------------------------------+

Edge Locations: 400+ worldwide
Regional Edge Caches: 13 locations
```

---

## Origins

### S3 Bucket
```
CloudFront --> S3 Bucket
                  |
           Origin Access Control (OAC)
           (restricts S3 access to CloudFront only)
```

### Custom Origin (HTTP)
```
CloudFront --> ALB / EC2 / Any HTTP server
                  |
           Custom headers for verification
```

### Origin Groups (Failover)
```
Primary Origin ----> If fails ----> Secondary Origin
   (us-east-1)                        (us-west-2)
```

---

## Origin Access Control (OAC)

Restrict S3 access to CloudFront only.

```json
// S3 Bucket Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/EXXX"
      }
    }
  }]
}
```

OAC replaces Origin Access Identity (OAI).

---

## Behaviors

URL pattern matching with different settings.

```
Distribution: d123.cloudfront.net
|
+-- Behavior: /api/*
|   +-- Origin: ALB
|   +-- Cache: Disabled
|   +-- Viewer Protocol: HTTPS only
|
+-- Behavior: /images/*
|   +-- Origin: S3
|   +-- Cache: 86400 seconds
|   +-- Compress: Yes
|
+-- Behavior: Default (*)
    +-- Origin: S3
    +-- Cache: 3600 seconds
    +-- Viewer Protocol: Redirect to HTTPS
```

---

## Caching

### TTL Settings
| Setting | Default | Description |
|---------|---------|-------------|
| Minimum TTL | 0 | Minimum cache time |
| Maximum TTL | 31536000 (1 year) | Maximum cache time |
| Default TTL | 86400 (24 hours) | When no Cache-Control header |

### Cache Keys
What makes objects unique in cache:

- URL path
- Query strings (configurable)
- Headers (configurable)
- Cookies (configurable)

### Cache Policies

```
Cache Policy: Managed-CachingOptimized
+-- TTL: Min=1, Max=31536000, Default=86400
+-- Query strings: None
+-- Headers: None
+-- Cookies: None
+-- Compression: Gzip, Brotli
```

### Origin Request Policies

Control what's sent to origin:
- Query strings
- Headers
- Cookies

---

## Invalidation

Force removal of cached content.

```bash
# Invalidate specific path
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/images/logo.png"

# Invalidate all
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# Invalidate pattern
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/images/*" "/css/*"
```

- First 1,000 paths/month free
- $0.005 per path after
- Takes time to propagate

Alternative: Use versioned file names (`logo-v2.png`)

---

## Security

### HTTPS
```
Viewer --HTTPS--> CloudFront --HTTP/HTTPS--> Origin
```

Options:
- Viewer Protocol Policy: HTTP/HTTPS, Redirect, HTTPS only
- Origin Protocol Policy: HTTP only, HTTPS only, Match Viewer

### SSL Certificates
- Default CloudFront certificate (`*.cloudfront.net`)
- Custom SSL certificate (ACM - us-east-1 only)
- SNI (free) or Dedicated IP ($600/month)

### Field-Level Encryption
Encrypt specific POST fields at edge.

### Signed URLs / Signed Cookies
Restrict access to content.

```
Signed URL: For individual files
+--------------------------------------------------------------+
| https://d123.cloudfront.net/premium/video.mp4                |
|   ?Expires=1609459200                                        |
|   &Signature=xxxxx                                           |
|   &Key-Pair-Id=APKAXXXXX                                     |
+--------------------------------------------------------------+

Signed Cookies: For multiple files (e.g., all videos)
```

### Geo Restriction
- Allowlist: Only specified countries
- Blocklist: Block specified countries

### AWS WAF Integration
- Protect against common attacks
- Rate limiting
- IP blocking

---

## Lambda@Edge

Run code at edge locations.

```
+--------------------------------------------------------------+
|                                                              |
|   Viewer Request --> CloudFront --> Origin Request           |
|        |                                    |                |
|        v                                    v                |
|   Lambda@Edge                          Lambda@Edge           |
|                                                              |
|   Origin Response <-- CloudFront <-- Viewer Response         |
|        |                                    |                |
|        v                                    v                |
|   Lambda@Edge                          Lambda@Edge           |
|                                                              |
+--------------------------------------------------------------+
```

### Trigger Points
| Event | Use Case |
|-------|----------|
| Viewer Request | Auth, URL rewrites |
| Origin Request | Dynamic origin selection |
| Origin Response | Add headers, transform |
| Viewer Response | Add security headers |

### Limits
- Memory: 128 MB (viewer), 10 GB (origin)
- Timeout: 5s (viewer), 30s (origin)
- Package: 1 MB (viewer), 50 MB (origin)
- Region: us-east-1 only

---

## CloudFront Functions

Lightweight functions for simple transformations.

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|--------------|
| Runtime | JavaScript | Node.js, Python |
| Execution | <1ms | <5-30s |
| Memory | 2 MB | 128 MB - 10 GB |
| Package | 10 KB | 1-50 MB |
| Network | No | Yes |
| Body access | No | Yes |
| Cost | 1/6 of Lambda@Edge | Higher |
| Events | Viewer only | All |

Use CloudFront Functions for:
- Header manipulation
- URL rewrites/redirects
- Cache key normalization
- JWT validation

---

## Real-Time Logs

Stream logs to Kinesis Data Streams.

```
CloudFront --> Kinesis Data Streams --> Kinesis Firehose --> S3/OpenSearch
```

Fields available:
- Timestamp, client IP, URI
- Status code, bytes sent
- Time to first byte
- Cache status

---

## CLI Quick Reference

```bash
# Create distribution
aws cloudfront create-distribution \
  --distribution-config file://config.json

# List distributions
aws cloudfront list-distributions

# Get distribution
aws cloudfront get-distribution --id EXXX

# Create invalidation
aws cloudfront create-invalidation \
  --distribution-id EXXX \
  --paths "/*"

# Update distribution
aws cloudfront update-distribution \
  --id EXXX \
  --distribution-config file://config.json \
  --if-match ETAG

# Disable distribution (before delete)
aws cloudfront update-distribution \
  --id EXXX \
  --distribution-config file://disabled-config.json \
  --if-match ETAG

# Delete distribution (must be disabled)
aws cloudfront delete-distribution --id EXXX --if-match ETAG
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Data transfer out | $0.085/GB (first 10TB) |
| Requests (HTTP) | $0.0075/10,000 |
| Requests (HTTPS) | $0.01/10,000 |
| Invalidation | Free (first 1,000/month) |
| Lambda@Edge | Per request + duration |
| CloudFront Functions | $0.1/million |

Price varies by region (price classes).

### Price Classes
- All edge locations (best performance)
- Price Class 200 (exclude expensive regions)
- Price Class 100 (only cheapest regions)

---

## Common Patterns

### Static Website
```
S3 (static files) <-- OAC <-- CloudFront <-- Route 53
```

### Dynamic + Static
```
                    +-- /api/* --> ALB (dynamic)
Route 53 --> CloudFront --+
                    +-- /* --> S3 (static)
```

### Multi-Origin Failover
```
CloudFront --> Origin Group
                  +-- Primary: S3 (us-east-1)
                  +-- Secondary: S3 (us-west-2)
```

---

## Exam Tips

1. **OAC** - replaces OAI, restricts S3 to CloudFront
2. **Custom SSL** - ACM certificate must be in us-east-1
3. **Signed URLs** - single file access control
4. **Signed Cookies** - multiple files access control
5. **Lambda@Edge** - us-east-1 only, more powerful
6. **CloudFront Functions** - viewer events only, faster, cheaper
7. **Invalidation** - 1000 free/month, use versioning instead
8. **Geo restriction** - allowlist or blocklist
9. **Origin Groups** - failover between origins
10. **Cache behaviors** - path pattern matching, precedence order
11. **Price classes** - reduce cost by limiting edge locations
12. **Field-level encryption** - encrypt POST data at edge
