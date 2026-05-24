# Azure Kubernetes Service (AKS)

> Managed Kubernetes service for deploying, scaling, and managing containerized applications.

## Overview

Azure Kubernetes Service (AKS) simplifies deploying a managed Kubernetes cluster in Azure. Azure handles critical tasks like health monitoring and maintenance. You only manage and maintain the agent nodes.

## Key Concepts

| Term | Definition |
|------|------------|
| Cluster | Kubernetes control plane + node pools |
| Node Pool | Group of nodes with same configuration |
| System Node Pool | Runs critical system pods |
| User Node Pool | Runs application workloads |
| Virtual Nodes | Serverless nodes using ACI |
| Managed Identity | Identity for cluster operations |
| AGIC | Application Gateway Ingress Controller |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AKS Cluster                               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Control Plane (Azure Managed)                  │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │ │
│  │  │API Server│ │Controller│ │Scheduler │ │  etcd    │      │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │ │
│  │                      FREE (Azure manages)                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Node Pools (You pay)                     │ │
│  │                                                              │ │
│  │  System Pool              User Pool 1         User Pool 2   │ │
│  │  ┌────┐ ┌────┐          ┌────┐ ┌────┐      ┌────┐ ┌────┐  │ │
│  │  │Node│ │Node│          │Node│ │Node│      │Node│ │Node│  │ │
│  │  └────┘ └────┘          └────┘ └────┘      └────┘ └────┘  │ │
│  │  (CoreDNS,              (Apps)             (GPU Apps)      │ │
│  │   kube-proxy)                                               │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Cluster Configuration

### Tiers

| Tier | SLA | Features | Use Case |
|------|-----|----------|----------|
| **Free** | No SLA | Basic cluster | Dev/test |
| **Standard** | 99.95% (single zone) | Uptime SLA | Production |
| **Premium** | 99.95%+ | Long-term support | Enterprise |

### Network Models

| Model | Description | Use Case |
|-------|-------------|----------|
| **Kubenet** | Basic, NAT-based | Simple deployments |
| **Azure CNI** | Pod IPs from VNet subnet | VNet integration |
| **Azure CNI Overlay** | Pod IPs from overlay network | Large clusters |
| **Azure CNI Powered by Cilium** | eBPF-based networking | Advanced networking |

### Kubenet vs Azure CNI

```
Kubenet:
┌─────────────────────────────────────┐
│ Node IP: 10.0.1.4                   │
│  ├── Pod: 10.244.0.1 (NAT'd)       │
│  └── Pod: 10.244.0.2 (NAT'd)       │
└─────────────────────────────────────┘
Pods use different IP range, NAT to node IP

Azure CNI:
┌─────────────────────────────────────┐
│ Node IP: 10.0.1.4                   │
│  ├── Pod: 10.0.1.5 (VNet IP)       │
│  └── Pod: 10.0.1.6 (VNet IP)       │
└─────────────────────────────────────┘
Pods get IPs from VNet subnet directly
```

## Node Pools

### System vs User Pools

| Type | Purpose | Requirements |
|------|---------|--------------|
| **System** | Cluster services (CoreDNS, tunnelfront) | At least 1, mode=System |
| **User** | Application workloads | Optional, mode=User |

### Node Pool Configuration

```bash
# Create GPU node pool
az aks nodepool add \
  --cluster-name myAKS \
  --resource-group myRG \
  --name gpupool \
  --node-count 2 \
  --node-vm-size Standard_NC6 \
  --mode User \
  --labels workload=gpu \
  --taints nvidia.com/gpu=true:NoSchedule
```

### Auto-Scaling

```
Cluster Autoscaler:
┌─────────────────────────────────────────────────────┐
│  Node Pool: apps                                     │
│  Min: 2 nodes                                        │
│  Max: 10 nodes                                       │
│  Current: 5 nodes                                    │
│                                                      │
│  Scale up: Pending pods need resources              │
│  Scale down: Nodes underutilized (< 50% for 10min) │
└─────────────────────────────────────────────────────┘
```

## Virtual Nodes

Serverless Kubernetes using Azure Container Instances.

```
┌─────────────────────────────────────────────────────┐
│                    AKS Cluster                       │
│                                                      │
│  Regular Node Pool      Virtual Node (ACI)          │
│  ┌────┐ ┌────┐         ┌─────────────────────┐     │
│  │Node│ │Node│         │ Serverless Pods     │     │
│  │    │ │    │         │ (burst to ACI)      │     │
│  └────┘ └────┘         └─────────────────────┘     │
│                                                      │
│  For: Long-running      For: Burst workloads,      │
│       workloads               quick scaling         │
└─────────────────────────────────────────────────────┘
```

## Networking

### Ingress Options

| Option | Layer | Features |
|--------|-------|----------|
| **NGINX Ingress** | 7 | Community standard |
| **AGIC** | 7 | Azure App Gateway integration |
| **Azure Service Mesh** | 7 | Istio-based service mesh |
| **Traefik** | 7 | Popular alternative |

### Service Types

| Type | Access |
|------|--------|
| ClusterIP | Internal only |
| NodePort | Node IP + port |
| LoadBalancer | Azure Load Balancer |
| ExternalName | DNS alias |

```yaml
# LoadBalancer service (creates Azure LB)
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: my-app
```

### Internal Load Balancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-app
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: internal-app
```

## Storage

### Storage Classes

| Class | Backing | Use Case |
|-------|---------|----------|
| **managed-csi** | Azure Disk (LRS) | General purpose |
| **managed-csi-premium** | Azure Disk (Premium SSD) | Production |
| **azurefile-csi** | Azure Files | Shared access |
| **azurefile-csi-premium** | Azure Files (Premium) | High performance shared |

### Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi-premium
  resources:
    requests:
      storage: 100Gi
```

## Identity and Security

### Managed Identity Types

| Type | Use Case |
|------|----------|
| **System-assigned** | Default, cluster identity |
| **User-assigned** | Custom, pre-created identity |
| **Kubelet Identity** | Node identity for pulling images |
| **Workload Identity** | Pod-level Azure AD identity |

### Workload Identity

```yaml
# Service Account with Azure AD identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  annotations:
    azure.workload.identity/client-id: <client-id>
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: my-service-account
  containers:
    - name: app
      image: myapp:latest
```

### RBAC

| Level | Scope |
|-------|-------|
| Azure RBAC | Cluster management |
| Kubernetes RBAC | In-cluster resources |
| Azure AD | User authentication |

## Azure Policy for AKS

Enforce cluster compliance using Azure Policy.

| Policy | Effect |
|--------|--------|
| No privileged containers | Deny |
| Require resource limits | Deny/Audit |
| Allowed container registries | Deny |
| No host networking | Deny |
| Required labels | Deny |

## Monitoring

### Container Insights

```
Azure Monitor
    │
    ├── Container Insights
    │   ├── Metrics (CPU, Memory, Network)
    │   ├── Logs (stdout, stderr)
    │   └── Inventory (pods, nodes, containers)
    │
    └── Log Analytics Workspace
        └── KQL queries for analysis
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| Node CPU/Memory | Node resource usage |
| Pod CPU/Memory | Pod resource usage |
| Container restart count | Stability indicator |
| Pod ready percentage | Health indicator |

## GitOps with Flux

```
Git Repository                    AKS Cluster
┌─────────────────┐   Flux      ┌─────────────────┐
│ manifests/      │ ─────────▶ │ Deployed        │
│   deployment.yaml│  sync      │ resources       │
│   service.yaml  │            │                 │
└─────────────────┘            └─────────────────┘
```

## CLI Quick Reference

```bash
# Create AKS cluster
az aks create \
  --name myAKS \
  --resource-group myRG \
  --node-count 3 \
  --enable-managed-identity \
  --network-plugin azure \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --name myAKS --resource-group myRG

# Scale node pool
az aks scale --name myAKS --resource-group myRG --node-count 5

# Enable cluster autoscaler
az aks update \
  --name myAKS \
  --resource-group myRG \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10

# Add node pool
az aks nodepool add \
  --cluster-name myAKS \
  --resource-group myRG \
  --name userpool \
  --node-count 3 \
  --mode User

# Upgrade cluster
az aks upgrade --name myAKS --resource-group myRG --kubernetes-version 1.28.0

# Get available versions
az aks get-versions --location eastus --output table

# Start/Stop cluster
az aks stop --name myAKS --resource-group myRG
az aks start --name myAKS --resource-group myRG

# Enable virtual nodes
az aks enable-addons \
  --name myAKS \
  --resource-group myRG \
  --addons virtual-node \
  --subnet-name VirtualNodeSubnet
```

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **Control plane**: Free, Azure-managed
2. **System node pool**: Required, runs cluster services
3. **Azure CNI**: Pods get VNet IPs; Kubenet: NAT-based
4. **Cluster autoscaler**: Scales nodes; HPA scales pods
5. **Virtual nodes**: Burst to ACI for serverless pods
6. **AGIC**: Use App Gateway as ingress controller
7. **Workload Identity**: Azure AD identity for pods
8. **Managed Identity**: Required for cluster operations
9. **Container Insights**: Metrics and logs to Log Analytics
10. **Standard tier**: Required for production SLA

## Gotchas

- Control plane is free; you pay for nodes
- Azure CNI requires larger subnets (IP per pod)
- System node pool cannot be deleted (need at least one)
- Cluster autoscaler won't scale below min or above max
- Virtual nodes only work with Azure CNI
- Upgrading affects pods (may restart)
- Stop/start resets dynamic IPs
- Container Insights agent uses resources on each node
- Kubelet identity is separate from cluster identity
- RBAC changes require cluster recreation to disable

## Limits

| Resource | Limit |
|----------|-------|
| Nodes per cluster | 5000 |
| Pods per node | 250 (Azure CNI), 110 (Kubenet) |
| Node pools per cluster | 100 |
| Clusters per subscription | 5000 |
| Nodes per node pool | 1000 |
| System pods per cluster | ~30 (varies by addons) |
| Max containers per pod | 300 |
| Kubernetes versions supported | N-2 (current minus 2) |
