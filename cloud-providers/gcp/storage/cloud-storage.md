# Google Cloud Storage (GCS)

> Unified object storage for developers and enterprises, from live data serving to data analytics/ML.

## Overview

Cloud Storage is a RESTful online file storage service for storing and accessing data on Google Cloud Platform infrastructure. It offers durability, availability, and scalability with strong consistency.

## Key Concepts

| Term | Definition |
|------|------------|
| Bucket | Container for objects (globally unique name) |
| Object | File stored in a bucket |
| Object Name | Full path to object (key) |
| Generation | Version number of an object |
| ACL | Access Control List (legacy) |
| IAM | Identity and Access Management (recommended) |

## Storage Classes

| Class | Availability | Min Duration | Use Case |
|-------|--------------|--------------|----------|
| **Standard** | 99.99% (multi-region) | None | Frequently accessed, hot data |
| **Nearline** | 99.9% | 30 days | Once per month access |
| **Coldline** | 99.9% | 90 days | Once per quarter access |
| **Archive** | 99.9% | 365 days | Long-term archive, yearly access |

### Storage Class Comparison

| Feature | Standard | Nearline | Coldline | Archive |
|---------|----------|----------|----------|---------|
| Storage cost | Highest | Lower | Lower | Lowest |
| Retrieval cost | None | Yes | Higher | Highest |
| Min storage duration | None | 30 days | 90 days | 365 days |
| Availability SLA | 99.95%/99.99% | 99.9% | 99.9% | 99.9% |

## Location Types

```
Multi-Regional:                Regional:              Dual-Region:
+-----------------+           +---------+           +-----------------+
|  US / EU / ASIA |           |us-east1 |           | nam4 (us-east1  |
|                 |           |         |           |    + us-west1)  |
| Multiple DCs    |           | 1 region|           |                 |
| across continent|           |         |           | 2 specific      |
|                 |           |         |           | regions         |
+-----------------+           +---------+           +-----------------+
    Highest                      Lower                  Balance
 availability                   latency              availability/cost
```

| Type | Description | Use Case |
|------|-------------|----------|
| **Region** | Single region | Data residency, lower cost |
| **Dual-region** | 2 specific regions | DR with control over locations |
| **Multi-region** | Continent (US, EU, ASIA) | Highest availability |

## Bucket Configuration

### Create Bucket

```bash
# Create bucket with settings
gsutil mb -p my-project \
  -c standard \
  -l us-central1 \
  -b on \
  gs://my-unique-bucket-name

# Using gcloud
gcloud storage buckets create gs://my-bucket \
  --project=my-project \
  --location=us-central1 \
  --default-storage-class=standard \
  --uniform-bucket-level-access
```

### Bucket Settings

| Setting | Description |
|---------|-------------|
| Location | Region/multi-region |
| Storage class | Default for new objects |
| Uniform bucket-level access | IAM-only (recommended) |
| Public access prevention | Block all public access |
| Versioning | Keep object history |
| Retention policy | Minimum retention period |
| Lifecycle | Automate class changes/deletion |

## Object Versioning

```
Bucket: my-bucket (versioning enabled)
+-- report.pdf
    +-- Generation: 1620000000000001 (v1)
    +-- Generation: 1620000000000002 (v2)
    +-- Generation: 1620000000000003 (current)

List versions:
gsutil ls -a gs://my-bucket/report.pdf

Restore version:
gsutil cp gs://my-bucket/report.pdf#1620000000000001 gs://my-bucket/report.pdf
```

## Lifecycle Management

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"isLive": false, "numNewerVersions": 3}
      }
    ]
  }
}
```

```bash
# Apply lifecycle policy
gsutil lifecycle set lifecycle.json gs://my-bucket

# View lifecycle policy
gsutil lifecycle get gs://my-bucket
```

## Access Control

### IAM (Recommended)

| Role | Description |
|------|-------------|
| `roles/storage.admin` | Full control of buckets and objects |
| `roles/storage.objectAdmin` | Full control of objects |
| `roles/storage.objectCreator` | Create objects |
| `roles/storage.objectViewer` | Read objects |
| `roles/storage.legacyBucketReader` | List bucket contents |

```bash
# Grant access
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=user:alice@example.com \
  --role=roles/storage.objectViewer

# Make bucket public (be careful!)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

### Uniform vs Fine-grained Access

| Type | Description |
|------|-------------|
| **Uniform** | IAM only, no object ACLs |
| **Fine-grained** | IAM + object-level ACLs |

```bash
# Enable uniform access
gcloud storage buckets update gs://my-bucket --uniform-bucket-level-access
```

### Signed URLs

Temporary access without requiring authentication.

```bash
# Create signed URL (valid 1 hour)
gsutil signurl -d 1h keyfile.json gs://my-bucket/object.txt

# Using gcloud
gcloud storage sign-url gs://my-bucket/object.txt --duration=1h
```

```python
from google.cloud import storage
from datetime import timedelta

client = storage.Client()
bucket = client.bucket("my-bucket")
blob = bucket.blob("object.txt")

url = blob.generate_signed_url(
    version="v4",
    expiration=timedelta(hours=1),
    method="GET"
)
print(url)
```

## Encryption

| Type | Description |
|------|-------------|
| **Google-managed** | Default, automatic |
| **CMEK** | Customer-managed keys in Cloud KMS |
| **CSEK** | Customer-supplied encryption keys |

```bash
# Create bucket with CMEK
gcloud storage buckets create gs://my-bucket \
  --default-encryption-key=projects/my-project/locations/us/keyRings/my-ring/cryptoKeys/my-key

# Upload with CSEK
gsutil -o "GSUtil:encryption_key=your-base64-key" cp file.txt gs://my-bucket/
```

## Object Retention & Holds

### Retention Policy

```bash
# Set retention policy (minimum 1 day)
gcloud storage buckets update gs://my-bucket \
  --retention-period=365d

# Lock retention policy (PERMANENT!)
gcloud storage buckets update gs://my-bucket \
  --lock-retention-period
```

### Object Holds

| Hold Type | Description |
|-----------|-------------|
| **Event-based** | Starts retention on release |
| **Temporary** | Prevent deletion temporarily |

```bash
# Set temporary hold
gcloud storage objects update gs://my-bucket/object.txt --temporary-hold

# Remove hold
gcloud storage objects update gs://my-bucket/object.txt --no-temporary-hold
```

## Pub/Sub Notifications

```bash
# Create notification
gcloud storage buckets notifications create gs://my-bucket \
  --topic=my-topic \
  --event-types=OBJECT_FINALIZE,OBJECT_DELETE

# List notifications
gcloud storage buckets notifications list gs://my-bucket
```

## Transfer Service

### Transfer Methods

| Method | Use Case |
|--------|----------|
| **gsutil** | Interactive, smaller transfers |
| **gcloud storage** | CLI operations |
| **Storage Transfer Service** | Scheduled, large-scale |
| **Transfer Appliance** | Offline, massive datasets |

```bash
# gsutil parallel upload
gsutil -m cp -r local-dir/ gs://my-bucket/

# Rsync (sync directories)
gsutil -m rsync -r local-dir/ gs://my-bucket/remote-dir/

# Transfer from S3
gcloud transfer jobs create s3://source-bucket gs://dest-bucket \
  --source-creds-file=aws-creds.json
```

## Performance Optimization

### Naming Best Practices

```
Bad (sequential):
+-- 2024-01-01-file1.csv
+-- 2024-01-01-file2.csv
+-- 2024-01-02-file1.csv
+-- (hotspot on partition)

Good (distributed):
+-- a1b2c3-2024-01-01-file1.csv
+-- d4e5f6-2024-01-01-file2.csv
+-- g7h8i9-2024-01-02-file1.csv
+-- (distributed across partitions)
```

### Parallel Composite Uploads

```bash
# Enable parallel composite upload for large files
gsutil -o GSUtil:parallel_composite_upload_threshold=150M cp large-file.zip gs://my-bucket/
```

## CLI Quick Reference

```bash
# Create bucket
gsutil mb gs://my-bucket
gcloud storage buckets create gs://my-bucket

# Upload file
gsutil cp file.txt gs://my-bucket/
gcloud storage cp file.txt gs://my-bucket/

# Download file
gsutil cp gs://my-bucket/file.txt ./
gcloud storage cp gs://my-bucket/file.txt ./

# List objects
gsutil ls gs://my-bucket/
gcloud storage ls gs://my-bucket/

# Delete object
gsutil rm gs://my-bucket/file.txt
gcloud storage rm gs://my-bucket/file.txt

# Copy between buckets
gsutil cp gs://source-bucket/file.txt gs://dest-bucket/

# Sync directory
gsutil -m rsync -r ./local-dir gs://my-bucket/remote-dir

# Set storage class
gsutil rewrite -s NEARLINE gs://my-bucket/file.txt

# Get object metadata
gsutil stat gs://my-bucket/file.txt

# Enable versioning
gsutil versioning set on gs://my-bucket

# Make public
gsutil acl ch -u AllUsers:R gs://my-bucket/file.txt

# Get signed URL
gsutil signurl -d 1h key.json gs://my-bucket/file.txt
```

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Multi-region**: Highest availability, redundant across regions
2. **Nearline/Coldline/Archive**: Have minimum storage durations
3. **Uniform bucket-level access**: Recommended, IAM only
4. **Signed URLs**: Temporary access without authentication
5. **Lifecycle rules**: Automate storage class transitions
6. **Versioning**: Preserves all object versions
7. **Retention policy lock**: Cannot be removed once locked
8. **CMEK**: Customer-managed keys in Cloud KMS
9. **Transfer Service**: For scheduled, large-scale transfers
10. **Object naming**: Avoid sequential prefixes for performance

## Gotchas

- Bucket names are globally unique across all GCP
- Cannot rename buckets (must create new and copy)
- Minimum storage duration charges apply (30/90/365 days)
- Nearline/Coldline/Archive have retrieval costs
- Uniform bucket-level access cannot be disabled after 90 days
- Retention policy lock is permanent
- Signed URLs require service account key or impersonation
- Object names are case-sensitive
- Cannot change bucket location after creation
- Public access prevention overrides IAM settings

## Limits

| Resource | Limit |
|----------|-------|
| Buckets per project | ~100,000 (soft) |
| Objects per bucket | No limit |
| Object size | 5 TiB |
| Object name length | 1,024 bytes |
| Labels per bucket | 64 |
| Lifecycle rules per bucket | 100 |
| Write rate per bucket | 1 write/second per object |
| Read rate per bucket | No limit (auto-scales) |
| Bucket operations | 1 per 2 seconds |
| Signed URL expiration | 7 days max |
