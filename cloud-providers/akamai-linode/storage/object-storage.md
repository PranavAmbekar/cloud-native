# Linode Object Storage

> S3-compatible object storage for unstructured data at scale.

## Overview

Linode Object Storage provides S3-compatible storage for files, backups, static websites, and application data. It offers high availability, durability, and global distribution through Akamai's edge network.

## Key Concepts

| Term | Definition |
|------|------------|
| Bucket | Container for objects |
| Object | File stored with metadata |
| Access Key | Credentials for S3 API access |
| Cluster | Regional storage endpoint |
| ACL | Access Control List |
| CORS | Cross-Origin Resource Sharing |

## S3 Compatibility

| Feature | Supported |
|---------|-----------|
| PUT/GET/DELETE objects | Yes |
| Multipart upload | Yes |
| Presigned URLs | Yes |
| Bucket policies | Yes |
| Object versioning | No |
| Object lock | No |
| Lifecycle policies | Limited |
| Server-side encryption | Yes (default) |

## Architecture

```
+---------------------------------------------------------------+
|                    Linode Object Storage                      |
|                                                               |
|  +-------------------+  +-------------------+                 |
|  | Cluster: us-east-1|  | Cluster: eu-central-1|              |
|  | us-east-1.linodeobjects.com                |               |
|  |                   |  |                   |                 |
|  |  +-------------+  |  |  +-------------+  |                 |
|  |  | my-bucket   |  |  |  | eu-bucket   |  |                 |
|  |  |  +-------+  |  |  |  |  +-------+  |  |                 |
|  |  |  |Objects|  |  |  |  |  |Objects|  |  |                 |
|  |  |  +-------+  |  |  |  |  +-------+  |  |                 |
|  |  +-------------+  |  |  +-------------+  |                 |
|  +-------------------+  +-------------------+                 |
|                                                               |
|  CDN: Integrated with Akamai Edge Network                     |
+---------------------------------------------------------------+
```

## Clusters (Regions)

| Cluster ID | Location | Endpoint |
|------------|----------|----------|
| us-east-1 | Newark, NJ | us-east-1.linodeobjects.com |
| eu-central-1 | Frankfurt | eu-central-1.linodeobjects.com |
| ap-south-1 | Singapore | ap-south-1.linodeobjects.com |
| us-southeast-1 | Atlanta | us-southeast-1.linodeobjects.com |

## Pricing

| Component | Cost |
|-----------|------|
| Storage | $5/mo per 250 GB |
| Outbound transfer | $0.005/GB after 1 TB/mo |
| Inbound transfer | Free |
| Requests | Free |

**Included**: 250 GB storage + 1 TB outbound transfer for $5/mo

## Create Access Keys

```bash
# Create access key pair
linode-cli object-storage keys-create \
  --label "my-app-key"

# Output:
# access_key: AKIAIOSFODNN7EXAMPLE
# secret_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# List keys
linode-cli object-storage keys-list

# Delete key
linode-cli object-storage keys-delete 12345
```

## Create Bucket

### CLI

```bash
# Create bucket
linode-cli object-storage buckets-create \
  --cluster us-east-1 \
  --label my-bucket

# List buckets
linode-cli object-storage buckets-list

# Delete bucket (must be empty)
linode-cli object-storage buckets-delete us-east-1 my-bucket
```

### AWS CLI (S3-compatible)

```bash
# Configure AWS CLI for Linode
aws configure --profile linode-obj
# AWS Access Key ID: <access_key>
# AWS Secret Access Key: <secret_key>
# Default region: us-east-1
# Default output format: json

# Create bucket
aws s3 mb s3://my-bucket \
  --endpoint-url https://us-east-1.linodeobjects.com \
  --profile linode-obj

# List buckets
aws s3 ls \
  --endpoint-url https://us-east-1.linodeobjects.com \
  --profile linode-obj
```

### s3cmd

```bash
# Configure s3cmd
cat > ~/.s3cfg << EOF
[default]
access_key = YOUR_ACCESS_KEY
secret_key = YOUR_SECRET_KEY
host_base = us-east-1.linodeobjects.com
host_bucket = %(bucket)s.us-east-1.linodeobjects.com
use_https = True
EOF

# Create bucket
s3cmd mb s3://my-bucket

# List buckets
s3cmd ls
```

## Upload/Download Objects

### AWS CLI

```bash
# Upload file
aws s3 cp file.txt s3://my-bucket/ \
  --endpoint-url https://us-east-1.linodeobjects.com

# Upload directory
aws s3 sync ./local-dir s3://my-bucket/prefix/ \
  --endpoint-url https://us-east-1.linodeobjects.com

# Download file
aws s3 cp s3://my-bucket/file.txt ./local-file.txt \
  --endpoint-url https://us-east-1.linodeobjects.com

# List objects
aws s3 ls s3://my-bucket/ \
  --endpoint-url https://us-east-1.linodeobjects.com

# Delete object
aws s3 rm s3://my-bucket/file.txt \
  --endpoint-url https://us-east-1.linodeobjects.com
```

### Python (boto3)

```python
import boto3

# Create client
s3 = boto3.client(
    's3',
    endpoint_url='https://us-east-1.linodeobjects.com',
    aws_access_key_id='YOUR_ACCESS_KEY',
    aws_secret_access_key='YOUR_SECRET_KEY'
)

# Upload file
s3.upload_file('local-file.txt', 'my-bucket', 'remote-file.txt')

# Download file
s3.download_file('my-bucket', 'remote-file.txt', 'local-file.txt')

# List objects
response = s3.list_objects_v2(Bucket='my-bucket')
for obj in response.get('Contents', []):
    print(obj['Key'], obj['Size'])

# Generate presigned URL
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'file.txt'},
    ExpiresIn=3600
)
```

### Node.js

```javascript
const { S3Client, PutObjectCommand, GetObjectCommand } = require('@aws-sdk/client-s3');

const client = new S3Client({
  endpoint: 'https://us-east-1.linodeobjects.com',
  region: 'us-east-1',
  credentials: {
    accessKeyId: 'YOUR_ACCESS_KEY',
    secretAccessKey: 'YOUR_SECRET_KEY'
  }
});

// Upload
async function upload(bucket, key, body) {
  await client.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: body
  }));
}

// Download
async function download(bucket, key) {
  const response = await client.send(new GetObjectCommand({
    Bucket: bucket,
    Key: key
  }));
  return response.Body;
}
```

## Bucket Access Control

### Public Bucket (Static Website)

```bash
# Set bucket ACL to public-read
aws s3api put-bucket-acl \
  --bucket my-bucket \
  --acl public-read \
  --endpoint-url https://us-east-1.linodeobjects.com

# Objects are accessible at:
# https://my-bucket.us-east-1.linodeobjects.com/file.txt
```

### Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

```bash
# Apply bucket policy
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://policy.json \
  --endpoint-url https://us-east-1.linodeobjects.com
```

## Static Website Hosting

```bash
# Upload website files
aws s3 sync ./website s3://my-website/ \
  --endpoint-url https://us-east-1.linodeobjects.com

# Set public-read ACL
aws s3api put-bucket-acl \
  --bucket my-website \
  --acl public-read \
  --endpoint-url https://us-east-1.linodeobjects.com

# Configure index document
aws s3 website s3://my-website/ \
  --index-document index.html \
  --error-document error.html \
  --endpoint-url https://us-east-1.linodeobjects.com
```

Website URL: `https://my-website.us-east-1.linodeobjects.com`

## CORS Configuration

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://example.com"],
      "AllowedMethods": ["GET", "PUT", "POST"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

```bash
aws s3api put-bucket-cors \
  --bucket my-bucket \
  --cors-configuration file://cors.json \
  --endpoint-url https://us-east-1.linodeobjects.com
```

## Lifecycle Rules

```bash
# Delete objects after 30 days
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "DeleteOldFiles",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Expiration": {"Days": 30}
      }
    ]
  }' \
  --endpoint-url https://us-east-1.linodeobjects.com
```

## CLI Quick Reference

```bash
# Access keys
linode-cli object-storage keys-create --label my-key
linode-cli object-storage keys-list
linode-cli object-storage keys-delete 12345

# Buckets (Linode CLI)
linode-cli object-storage buckets-create --cluster us-east-1 --label my-bucket
linode-cli object-storage buckets-list
linode-cli object-storage buckets-delete us-east-1 my-bucket

# Objects (AWS CLI with endpoint)
aws s3 cp file.txt s3://bucket/ --endpoint-url https://us-east-1.linodeobjects.com
aws s3 ls s3://bucket/ --endpoint-url https://us-east-1.linodeobjects.com
aws s3 rm s3://bucket/file.txt --endpoint-url https://us-east-1.linodeobjects.com
aws s3 sync ./dir s3://bucket/prefix/ --endpoint-url https://us-east-1.linodeobjects.com
```

## Best Practices

```
1. Security
   +-- Use separate access keys per application
   +-- Rotate access keys regularly
   +-- Use presigned URLs for temporary access
   +-- Keep buckets private unless needed public

2. Performance
   +-- Use multipart upload for large files (>100MB)
   +-- Enable transfer acceleration with Akamai CDN
   +-- Use appropriate region for latency

3. Cost
   +-- Monitor outbound transfer
   +-- Clean up unused objects
   +-- Use lifecycle rules for automatic deletion

4. Organization
   +-- Use meaningful bucket names
   +-- Organize objects with prefixes
   +-- Tag objects for management
```

## Gotchas

- No object versioning support
- No object lock/retention policies
- Bucket names globally unique per cluster
- Maximum object size: 5 TB
- Maximum multipart upload parts: 10,000
- Bucket must be empty before deletion
- Access keys are account-wide, not per-bucket
- No server-side copy between clusters

## Limits

| Resource | Limit |
|----------|-------|
| Buckets per cluster | 1,000 |
| Objects per bucket | Unlimited |
| Object size | 5 TB |
| Object metadata | 2 KB |
| Access keys | 100 |
| Multipart parts | 10,000 |
| Part size | 5 MB - 5 GB |
