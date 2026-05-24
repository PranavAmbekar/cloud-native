# Azure Blob Storage

> Massively scalable object storage for unstructured data in the cloud.

## Overview

Azure Blob Storage stores massive amounts of unstructured data—text or binary. Optimized for storing massive amounts of data, serving images/documents, streaming video/audio, storing data for backup and disaster recovery.

## Key Concepts

| Term | Definition |
|------|------------|
| Storage Account | Top-level namespace for all Azure Storage services |
| Container | Logical grouping of blobs (similar to a folder) |
| Blob | The actual file/object stored |
| Block Blob | Optimized for upload (text, binary files) |
| Append Blob | Optimized for append operations (logs) |
| Page Blob | Optimized for random read/write (VHDs) |

## Storage Account Types

| Type | Services | Performance | Replication |
|------|----------|-------------|-------------|
| **Standard general-purpose v2** | Blob, File, Queue, Table | Standard | LRS, ZRS, GRS, GZRS |
| **Premium block blobs** | Blob only | Premium | LRS, ZRS |
| **Premium file shares** | File only | Premium | LRS, ZRS |
| **Premium page blobs** | Page blobs only | Premium | LRS |

## Access Tiers

| Tier | Availability | Min Duration | Use Case |
|------|--------------|--------------|----------|
| **Hot** | 99.9% (99.99% RA) | None | Frequently accessed data |
| **Cool** | 99% (99.9% RA) | 30 days | Infrequent access, short-term backup |
| **Cold** | 99% (99.9% RA) | 90 days | Rarely accessed, long-term backup |
| **Archive** | Offline | 180 days | Long-term archive, compliance |

### Archive Tier Rehydration

| Priority | Time | Cost |
|----------|------|------|
| High | < 1 hour | Higher |
| Standard | < 15 hours | Lower |

```
Hot ←→ Cool ←→ Cold ←→ Archive
     (instant)  (instant)  (rehydration required)
```

## Redundancy Options

| Type | Description | Durability |
|------|-------------|------------|
| **LRS** | 3 copies in single datacenter | 11 9's |
| **ZRS** | 3 copies across availability zones | 12 9's |
| **GRS** | LRS + async copy to paired region | 16 9's |
| **GZRS** | ZRS + async copy to paired region | 16 9's |
| **RA-GRS** | GRS + read access to secondary | 16 9's |
| **RA-GZRS** | GZRS + read access to secondary | 16 9's |

```
Primary Region                  Secondary Region (Paired)
┌─────────────────┐             ┌─────────────────┐
│  Zone 1  Zone 2  Zone 3 │ ──────▶ │  LRS (3 copies) │
│   │       │       │     │ async   │                 │
│   └───────┴───────┘     │         │  (Read access   │
│      ZRS (3 copies)     │         │   with RA-*)    │
└─────────────────────────┘         └─────────────────┘
```

## Lifecycle Management

```json
{
  "rules": [
    {
      "name": "moveToArchive",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": {"daysAfterModificationGreaterThan": 30},
            "tierToArchive": {"daysAfterModificationGreaterThan": 90},
            "delete": {"daysAfterModificationGreaterThan": 365}
          }
        }
      }
    }
  ]
}
```

## Security

### Access Control Options

| Method | Description | Use Case |
|--------|-------------|----------|
| **Azure RBAC** | Role-based access control | Management + data plane |
| **Storage Account Keys** | Full access keys | Legacy, admin access |
| **SAS (Shared Access Signature)** | Time-limited, scoped access | Temporary external access |
| **Azure AD** | Identity-based access | Enterprise, managed identities |
| **Anonymous Access** | Public read access | Public websites, CDN |

### SAS Types

| Type | Scope |
|------|-------|
| **Account SAS** | Entire storage account |
| **Service SAS** | Specific service (blob, file, queue, table) |
| **User Delegation SAS** | Azure AD signed (most secure) |

```bash
# Generate SAS token
az storage blob generate-sas \
  --account-name myaccount \
  --container-name mycontainer \
  --name myblob.txt \
  --permissions r \
  --expiry 2024-12-31
```

### Encryption

| Type | Description |
|------|-------------|
| **Encryption at rest** | Always on, Microsoft-managed or customer-managed keys |
| **Encryption in transit** | HTTPS enforced (can require) |
| **Infrastructure encryption** | Double encryption (optional) |
| **Client-side encryption** | Encrypt before upload |

## Blob Versioning & Soft Delete

```
┌─────────────────────────────────────────────────────┐
│  Container: documents                                │
│                                                      │
│  report.pdf ──┬── Version 1 (previous)              │
│               ├── Version 2 (previous)              │
│               └── Version 3 (current)               │
│                                                      │
│  Soft Delete: 7-30 days retention (configurable)    │
│  Container Soft Delete: Recover deleted containers  │
└─────────────────────────────────────────────────────┘
```

## Data Protection Features

| Feature | Purpose |
|---------|---------|
| **Soft Delete** | Recover deleted blobs/containers |
| **Versioning** | Maintain previous versions of blobs |
| **Blob Snapshots** | Read-only point-in-time copies |
| **Immutable Storage** | WORM (Write Once Read Many) |
| **Point-in-time Restore** | Restore block blobs to earlier state |

### Immutability Policies

| Policy | Description |
|--------|-------------|
| **Time-based retention** | Locked for specified period |
| **Legal hold** | Locked until explicitly removed |

## Object Replication

Async copy blobs between storage accounts.

```
Source Account                    Destination Account
┌─────────────────┐              ┌─────────────────┐
│ Container: logs │  ──────────▶ │ Container: logs │
│                 │    async     │                 │
│ (Policy based)  │              │ (Replica)       │
└─────────────────┘              └─────────────────┘
```

Requirements:
- Versioning enabled on both
- Change feed enabled on source
- Same account type (GPv2 or Premium)

## Static Website Hosting

```
Primary endpoint:
https://<storage-account>.blob.core.windows.net/$web/

Static website endpoint:
https://<storage-account>.z13.web.core.windows.net/
```

- Set index document (index.html)
- Set error document (404.html)
- Use Azure CDN for custom domain + HTTPS

## Performance

### Scalability Targets

| Resource | Limit |
|----------|-------|
| Storage account capacity | 5 PiB |
| Max ingress (US) | 10 Gbps |
| Max egress (US) | 50 Gbps |
| Max request rate | 20,000 requests/second |
| Max bandwidth per blob | 60 MiB/s |

### Optimization

- **Premium block blobs**: Consistent low latency
- **Data Lake Storage Gen2**: Hierarchical namespace for analytics
- **Azure CDN**: Cache at edge locations
- **AzCopy**: Optimized data transfer tool
- **Block blobs**: Upload blocks in parallel

## Data Lake Storage Gen2

Hierarchical namespace on Blob Storage for big data analytics.

| Feature | Blob Storage | Data Lake Gen2 |
|---------|--------------|----------------|
| Namespace | Flat | Hierarchical |
| Directory operations | Slow (rename = copy) | Fast (atomic) |
| POSIX ACLs | No | Yes |
| Analytics integration | Basic | Optimized (Spark, Databricks) |

## CLI Quick Reference

```bash
# Create storage account
az storage account create \
  --name mystorageaccount \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS

# Create container
az storage container create \
  --name mycontainer \
  --account-name mystorageaccount

# Upload blob
az storage blob upload \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --name myblob.txt \
  --file ./local-file.txt

# List blobs
az storage blob list \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --output table

# Download blob
az storage blob download \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --name myblob.txt \
  --file ./downloaded.txt

# Set access tier
az storage blob set-tier \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --name myblob.txt \
  --tier Cool

# Copy with AzCopy (faster for large transfers)
azcopy copy './local-dir/*' 'https://mystorageaccount.blob.core.windows.net/mycontainer?SAS' --recursive
```

## Exam Tips (AZ-104, AZ-305)

1. **Hot vs Cool vs Archive**: Cool has 30-day minimum, Archive needs rehydration
2. **LRS vs ZRS vs GRS**: Know durability and availability differences
3. **RA-GRS**: Adds read access to secondary (higher SLA)
4. **Immutable storage**: Cannot delete/modify until retention expires
5. **User Delegation SAS**: Most secure SAS type (Azure AD signed)
6. **Static website**: Use $web container, separate endpoint
7. **Data Lake Gen2**: Enable hierarchical namespace at creation (cannot change)
8. **Soft delete**: Container soft delete is separate from blob soft delete
9. **Object replication**: Requires versioning and change feed
10. **Premium block blobs**: No access tiers (always hot equivalent)

## Gotchas

- Storage account names are globally unique (3-24 lowercase letters/numbers)
- Archive tier: Cannot read directly, must rehydrate first
- Cool/Cold/Archive: Early deletion charges apply
- Hierarchical namespace (Data Lake Gen2) cannot be enabled after creation
- GRS replication is asynchronous (RPO typically < 15 minutes)
- Blob versioning increases storage costs
- Anonymous access must be enabled at account level first
- Premium storage doesn't support GRS/GZRS

## Limits

| Resource | Limit |
|----------|-------|
| Storage accounts per subscription per region | 250 |
| Max storage account capacity | 5 PiB |
| Max blob size (block blob) | 190.7 TiB |
| Max block size | 4000 MiB |
| Max blocks per blob | 50,000 |
| Containers per storage account | Unlimited |
| Blobs per container | Unlimited |
