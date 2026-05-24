# Azure Container Instances (ACI)

> Run containers without managing servers. Fastest and simplest way to run a container in Azure.

## Overview

Azure Container Instances (ACI) offers the fastest and simplest way to run a container in Azure, without having to manage virtual machines or adopt Kubernetes. It's ideal for isolated containers, batch jobs, and burst scenarios.

## Key Concepts

| Term | Definition |
|------|------------|
| Container Group | Collection of containers scheduled on same host |
| Container | Running instance of container image |
| Resource Request | CPU and memory allocated |
| Volume Mount | Storage attached to containers |
| Init Container | Runs before main containers |
| Restart Policy | Behavior on container exit |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Container Group                          │
│                                                              │
│  ┌───────────────────┐  ┌───────────────────┐              │
│  │    Container 1    │  │    Container 2    │              │
│  │   (Web Server)    │  │   (Sidecar/Log)   │              │
│  │   Port: 80        │  │                   │              │
│  └───────────────────┘  └───────────────────┘              │
│            │                     │                          │
│            └──────────┬──────────┘                          │
│                       │                                      │
│              Shared Resources:                              │
│              • Network (same IP)                            │
│              • Volumes                                       │
│              • Lifecycle                                     │
│                                                              │
│  Public IP: 1.2.3.4  or  Private (VNet)                    │
└─────────────────────────────────────────────────────────────┘
```

## Container Group Configuration

### Restart Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Always** | Restart on any exit | Long-running services |
| **OnFailure** | Restart only on failure (exit != 0) | Batch jobs with retry |
| **Never** | Don't restart | One-time tasks |

### Resource Allocation

```yaml
resources:
  requests:
    cpu: 2           # 1-4 CPUs
    memoryInGB: 4    # 0.5-16 GB
  limits:
    cpu: 4
    memoryInGB: 8
    gpu:
      count: 1
      sku: K80       # K80, P100, V100
```

## Deployment Options

### Azure CLI

```bash
# Simple container
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image nginx:latest \
  --cpu 1 \
  --memory 1.5 \
  --ports 80 \
  --dns-name-label myapp

# With environment variables
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myapp:latest \
  --environment-variables \
    DB_HOST=mydb.database.windows.net \
    APP_ENV=production \
  --secure-environment-variables \
    DB_PASSWORD=secretpassword
```

### YAML Deployment

```yaml
apiVersion: 2019-12-01
location: eastus
name: myContainerGroup
properties:
  containers:
  - name: web
    properties:
      image: nginx:latest
      ports:
      - port: 80
      resources:
        requests:
          cpu: 1
          memoryInGB: 1.5
      environmentVariables:
      - name: APP_ENV
        value: production
      - name: SECRET_KEY
        secureValue: supersecret
  - name: sidecar
    properties:
      image: busybox
      command: ["/bin/sh", "-c", "while true; do echo hello; sleep 10; done"]
      resources:
        requests:
          cpu: 0.5
          memoryInGB: 0.5
  osType: Linux
  ipAddress:
    type: Public
    ports:
    - port: 80
      protocol: TCP
    dnsNameLabel: myapp
  restartPolicy: Always
```

## Networking

### Public IP

```bash
# Create with public IP and DNS label
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image nginx \
  --dns-name-label myuniquename \
  --ports 80 443

# Access via: myuniquename.eastus.azurecontainer.io
```

### VNet Integration

```bash
# Create in virtual network
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myapp:latest \
  --vnet myVNet \
  --subnet mySubnet \
  --ports 80

# Container gets private IP from subnet
# No public IP, accessible only within VNet
```

### Network Profile (for subnet delegation)

```
VNet: 10.0.0.0/16
└── Subnet: 10.0.1.0/24
    └── Delegated to: Microsoft.ContainerInstance/containerGroups
        └── Container Group: 10.0.1.4 (private IP)
```

## Storage

### Volume Types

| Type | Description | Use Case |
|------|-------------|----------|
| **emptyDir** | Temporary, shared | Scratch space |
| **Azure Files** | Persistent file share | Shared data |
| **Secret** | Mount secrets as files | Credentials |
| **gitRepo** | Clone git repo (deprecated) | Code |

### Azure Files Mount

```yaml
properties:
  containers:
  - name: mycontainer
    properties:
      image: myapp
      volumeMounts:
      - name: data
        mountPath: /app/data
  volumes:
  - name: data
    azureFile:
      shareName: myshare
      storageAccountName: mystorageaccount
      storageAccountKey: <key>
```

### Secret Volume

```yaml
volumes:
- name: secrets
  secret:
    mysecret.txt: base64encodedvalue==

containers:
- name: myapp
  volumeMounts:
  - name: secrets
    mountPath: /run/secrets
```

## Init Containers

Run before main containers start.

```yaml
properties:
  initContainers:
  - name: init-db
    properties:
      image: busybox
      command: ["sh", "-c", "until nc -z db 5432; do sleep 1; done"]
  containers:
  - name: app
    properties:
      image: myapp
```

## Container Registry Authentication

### ACR Integration

```bash
# With managed identity
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myacr.azurecr.io/myapp:latest \
  --acr-identity [system] \
  --assign-identity

# With service principal
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myacr.azurecr.io/myapp:latest \
  --registry-login-server myacr.azurecr.io \
  --registry-username <sp-id> \
  --registry-password <sp-password>
```

## Managed Identity

```bash
# System-assigned identity
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myapp \
  --assign-identity

# User-assigned identity
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myapp \
  --assign-identity /subscriptions/.../userAssignedIdentities/myIdentity
```

Access Azure resources without credentials:

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

# Uses managed identity automatically
credential = DefaultAzureCredential()
blob_client = BlobServiceClient(account_url="https://mystorageaccount.blob.core.windows.net", credential=credential)
```

## GPU Support

```bash
az container create \
  --name gpucontainer \
  --resource-group myRG \
  --image tensorflow/tensorflow:latest-gpu \
  --cpu 4 \
  --memory 14 \
  --gpu 1 \
  --gpu-type K80
```

| GPU Type | Memory | Use Case |
|----------|--------|----------|
| K80 | 12 GB | Basic ML |
| P100 | 16 GB | Training |
| V100 | 16 GB | High-performance |

## Monitoring

### Logs

```bash
# View logs
az container logs --name mycontainer --resource-group myRG

# Stream logs
az container logs --name mycontainer --resource-group myRG --follow

# Logs for specific container in group
az container logs --name mycontainer --resource-group myRG --container-name sidecar
```

### Metrics

| Metric | Description |
|--------|-------------|
| CpuUsage | CPU cores used |
| MemoryUsage | Memory bytes used |
| NetworkBytesReceivedPerSecond | Inbound network |
| NetworkBytesTransmittedPerSecond | Outbound network |

### Execute Command

```bash
# Start interactive shell
az container exec \
  --name mycontainer \
  --resource-group myRG \
  --exec-command "/bin/sh"

# Run single command
az container exec \
  --name mycontainer \
  --resource-group myRG \
  --exec-command "ls -la /app"
```

## Use Cases

### Batch Processing

```bash
az container create \
  --name batchjob \
  --resource-group myRG \
  --image mybatch:latest \
  --restart-policy Never \
  --environment-variables \
    INPUT_FILE=input.csv \
    OUTPUT_BLOB=results.json
```

### CI/CD Agents

```yaml
# Azure DevOps agent in ACI
containers:
- name: agent
  image: mcr.microsoft.com/azure-pipelines/vsts-agent:latest
  environmentVariables:
  - name: AZP_URL
    value: https://dev.azure.com/myorg
  - name: AZP_TOKEN
    secureValue: <pat>
  - name: AZP_POOL
    value: ACI-Pool
```

### Virtual Nodes (AKS Burst)

```yaml
# Pod scheduled to virtual-node (ACI)
nodeSelector:
  kubernetes.io/role: agent
  type: virtual-kubelet
tolerations:
- key: virtual-kubelet.io/provider
  operator: Exists
```

## CLI Quick Reference

```bash
# Create container
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image nginx

# List containers
az container list --resource-group myRG --output table

# Show details
az container show --name mycontainer --resource-group myRG

# Get logs
az container logs --name mycontainer --resource-group myRG

# Restart container
az container restart --name mycontainer --resource-group myRG

# Stop container
az container stop --name mycontainer --resource-group myRG

# Start container
az container start --name mycontainer --resource-group myRG

# Delete container
az container delete --name mycontainer --resource-group myRG --yes

# Export to YAML
az container export --name mycontainer --resource-group myRG --file container.yaml
```

## Pricing

```
Billing: Per second (minimum 1 minute)

Components:
- vCPU: ~$0.000012/second per vCPU
- Memory: ~$0.000001/second per GB
- GPU: Additional charge per GPU

Example (1 vCPU, 1.5 GB, 1 hour):
vCPU:   0.000012 × 3600 × 1   = $0.043
Memory: 0.000001 × 3600 × 1.5 = $0.005
Total:                          ~$0.048/hour
```

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **Container Group**: All containers share network and lifecycle
2. **Restart policies**: Always (services), Never (jobs), OnFailure (retry)
3. **VNet integration**: Requires subnet delegation
4. **No public IP + VNet**: Private access only
5. **GPU**: Available in limited regions
6. **Init containers**: Run before main containers
7. **Managed identity**: Access Azure resources securely
8. **Azure Files**: Persistent storage across restarts
9. **Billing**: Per-second, minimum 1 minute
10. **Virtual nodes**: ACI as Kubernetes burst capacity

## Gotchas

- Container groups share single IP (all ports must be unique)
- VNet containers can't have public IP
- GPU availability is region-limited
- emptyDir lost on container restart
- Windows containers have different resource limits
- Subnet delegation required for VNet
- Can't update running container group (must recreate)
- Secret volumes are base64 encoded
- No built-in load balancing (use App Gateway or Front Door)
- Linux and Windows containers can't be mixed in same group

## Limits

| Resource | Linux | Windows |
|----------|-------|---------|
| CPU per container group | 4 | 4 |
| Memory per container group | 16 GB | 16 GB |
| Containers per group | 60 | 60 |
| Volumes per group | 20 | 20 |
| Ports per group | 100 | 100 |
| GPU per container | 4 | N/A |
| Container image size | 15 GB | 15 GB |
