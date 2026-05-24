# Linode Block Storage

> Scalable, high-performance SSD block storage volumes for Linodes.

## Overview

Block Storage provides additional SSD-based storage that can be attached to Linodes. Volumes persist independently of Linodes and can be resized, moved, and cloned.

## Key Concepts

| Term | Definition |
|------|------------|
| Volume | Block storage device |
| Attachment | Connection between volume and Linode |
| Size | Storage capacity (10 GB - 10 TB) |
| IOPS | I/O operations per second |
| Throughput | Data transfer rate |

## Architecture

```
+---------------------------------------------------------------+
|                         Linode                                |
|                                                               |
|  Boot Disk          Block Storage Volumes                     |
|  +----------+       +----------+  +----------+                |
|  | /dev/sda |       | /dev/sdc |  | /dev/sdd |                |
|  | (OS)     |       | (Data)   |  | (Logs)   |                |
|  | 50 GB    |       | 250 GB   |  | 100 GB   |                |
|  +----------+       +----------+  +----------+                |
|                          |             |                      |
|                     NVMe Storage Backend                      |
+---------------------------------------------------------------+
```

## Performance

| Metric | Specification |
|--------|---------------|
| IOPS | Up to 10,000 |
| Throughput | Up to 350 MB/s |
| Latency | Sub-millisecond |
| Durability | 99.99% |
| Encryption | AES-256 at rest |

## Pricing

| Size | Price/mo |
|------|----------|
| 10 GB | $1 |
| 100 GB | $10 |
| 250 GB | $25 |
| 500 GB | $50 |
| 1 TB | $100 |

**Formula**: $0.10 per GB per month

## Create Volume

### CLI

```bash
# Create volume
linode-cli volumes create \
  --label my-volume \
  --size 100 \
  --region us-east

# Create and attach to Linode
linode-cli volumes create \
  --label my-volume \
  --size 100 \
  --linode_id 12345678

# List volumes
linode-cli volumes list

# View volume
linode-cli volumes view 54321
```

### API

```bash
curl -X POST https://api.linode.com/v4/volumes \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "my-volume",
    "size": 100,
    "region": "us-east"
  }'
```

### Terraform

```hcl
resource "linode_volume" "data" {
  label     = "my-volume"
  size      = 100
  region    = "us-east"
  linode_id = linode_instance.web.id
}
```

## Attach & Mount

### Attach Volume

```bash
# Attach to Linode
linode-cli volumes attach 54321 \
  --linode_id 12345678

# Detach from Linode
linode-cli volumes detach 54321
```

### Mount Volume (Linux)

```bash
# First time setup - create filesystem
mkfs.ext4 /dev/disk/by-id/scsi-0Linode_Volume_my-volume

# Create mount point
mkdir /mnt/my-volume

# Mount
mount /dev/disk/by-id/scsi-0Linode_Volume_my-volume /mnt/my-volume

# Add to fstab for persistent mount
echo '/dev/disk/by-id/scsi-0Linode_Volume_my-volume /mnt/my-volume ext4 defaults 0 2' >> /etc/fstab

# Verify
df -h /mnt/my-volume
```

### Recommended fstab Options

```bash
# For general use
/dev/disk/by-id/scsi-0Linode_Volume_my-volume /mnt/data ext4 defaults,noatime 0 2

# For databases
/dev/disk/by-id/scsi-0Linode_Volume_db /mnt/database ext4 defaults,noatime,data=ordered 0 2

# XFS filesystem
/dev/disk/by-id/scsi-0Linode_Volume_my-volume /mnt/data xfs defaults,noatime 0 2
```

## Resize Volume

```bash
# Resize volume (only increase, no decrease)
linode-cli volumes resize 54321 --size 200

# After resize, extend filesystem
# For ext4:
resize2fs /dev/disk/by-id/scsi-0Linode_Volume_my-volume

# For XFS:
xfs_growfs /mnt/my-volume
```

## Clone Volume

```bash
# Clone volume (same region)
linode-cli volumes clone 54321 \
  --label my-volume-clone
```

## Use Cases

### Database Storage

```bash
# Create volume for PostgreSQL
linode-cli volumes create \
  --label postgres-data \
  --size 500 \
  --linode_id 12345678

# Mount and configure PostgreSQL
mkdir -p /mnt/postgres-data
mount /dev/disk/by-id/scsi-0Linode_Volume_postgres-data /mnt/postgres-data
chown postgres:postgres /mnt/postgres-data

# Update postgresql.conf
# data_directory = '/mnt/postgres-data'
```

### Application Logs

```bash
# Create volume for logs
linode-cli volumes create \
  --label app-logs \
  --size 100 \
  --linode_id 12345678

# Mount
mkdir -p /mnt/logs
mount /dev/disk/by-id/scsi-0Linode_Volume_app-logs /mnt/logs

# Configure log rotation
cat > /etc/logrotate.d/myapp << EOF
/mnt/logs/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
}
EOF
```

### Shared Storage with GlusterFS

```bash
# On each node, create and mount volume
linode-cli volumes create --label gluster-brick --size 500 --linode_id $LINODE_ID
mkfs.xfs /dev/disk/by-id/scsi-0Linode_Volume_gluster-brick
mkdir -p /mnt/brick
mount /dev/disk/by-id/scsi-0Linode_Volume_gluster-brick /mnt/brick

# Create GlusterFS volume
gluster volume create gv0 replica 3 \
  node1:/mnt/brick/gv0 \
  node2:/mnt/brick/gv0 \
  node3:/mnt/brick/gv0
gluster volume start gv0
```

## CLI Quick Reference

```bash
# Create volume
linode-cli volumes create --label my-vol --size 100 --region us-east

# List volumes
linode-cli volumes list

# View volume
linode-cli volumes view 54321

# Attach volume
linode-cli volumes attach 54321 --linode_id 12345678

# Detach volume
linode-cli volumes detach 54321

# Resize volume
linode-cli volumes resize 54321 --size 200

# Clone volume
linode-cli volumes clone 54321 --label my-vol-clone

# Delete volume
linode-cli volumes delete 54321
```

## Best Practices

```
1. Filesystem Selection
   +-- ext4: General purpose, good recovery tools
   +-- XFS: Large files, better scaling
   +-- Don't use ext3 (limited to 16 TB)

2. Performance
   +-- Use noatime mount option
   +-- Align partitions to 4K boundaries
   +-- Consider RAID for redundancy

3. Backup
   +-- Volumes not included in Linode backups
   +-- Use rsync, Restic, or similar tools
   +-- Clone volumes for point-in-time copies

4. Monitoring
   +-- Monitor disk space usage
   +-- Set up alerts for low space
   +-- Monitor I/O metrics
```

## Gotchas

- Cannot decrease volume size (only increase)
- Maximum 8 volumes per Linode
- Volume and Linode must be in same region
- Detaching requires unmount first
- Volume not included in Linode backups
- Cannot attach to multiple Linodes simultaneously
- Cloning creates new independent volume
- Device path may change; use /dev/disk/by-id/

## Limits

| Resource | Limit |
|----------|-------|
| Volumes per Linode | 8 |
| Volume size | 10 GB - 10 TB |
| Volumes per account | Varies by region |
| Total storage per region | 10 TB default |
