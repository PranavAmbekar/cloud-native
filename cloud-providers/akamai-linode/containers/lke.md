# Linode Kubernetes Engine (LKE)

> Managed Kubernetes service for deploying containerized applications at scale.

## Overview

Linode Kubernetes Engine (LKE) is a fully-managed container orchestration service. Linode manages the control plane while you manage worker nodes. Deploy production-ready Kubernetes clusters in minutes.

## Key Concepts

| Term | Definition |
|------|------------|
| Cluster | Kubernetes control plane + worker nodes |
| Node Pool | Group of Linodes with same configuration |
| Control Plane | Managed Kubernetes masters (free) |
| Worker Node | Linode running your workloads |
| kubeconfig | Cluster access credentials |
| High Availability | Multi-master control plane |

## Architecture

```
+---------------------------------------------------------------+
|                      LKE Cluster                              |
|                                                               |
|  +---------------------------+                                |
|  |  Control Plane (Managed)  |                                |
|  |  +-------+ +-------+      |                                |
|  |  |  API  | | etcd  |      |                                |
|  |  |Server | |       |      |  FREE - Linode Managed         |
|  |  +-------+ +-------+      |                                |
|  +---------------------------+                                |
|               |                                               |
|               v                                               |
|  +---------------------------+  +---------------------------+ |
|  |     Node Pool: default    |  |    Node Pool: gpu         | |
|  |   g6-standard-4 (3 nodes) |  |   g6-gpu-rtx (2 nodes)    | |
|  |   +------+ +------+ +----+|  |   +------+ +------+       | |
|  |   | Node | | Node | |Node||  |   | Node | | Node |       | |
|  |   |[pods]| |[pods]| |[po]||  |   |[pods]| |[pods]|       | |
|  |   +------+ +------+ +----+|  |   +------+ +------+       | |
|  +---------------------------+  +---------------------------+ |
|                                                               |
|  NodeBalancer: Automatic LoadBalancer provisioning            |
+---------------------------------------------------------------+
```

## Pricing

| Component | Cost |
|-----------|------|
| Control Plane | Free |
| Worker Nodes | Standard Linode pricing |
| NodeBalancer | $10/mo per LoadBalancer service |
| Block Storage | $0.10/GB/mo |

**Example**: 3-node cluster with g6-standard-4 = $72/mo ($24/node × 3)

## Create Cluster

### CLI

```bash
# Create cluster
linode-cli lke cluster-create \
  --label my-cluster \
  --region us-east \
  --k8s_version 1.29 \
  --control_plane.high_availability true \
  --node_pools.type g6-standard-4 \
  --node_pools.count 3 \
  --node_pools.autoscaler.enabled true \
  --node_pools.autoscaler.min 3 \
  --node_pools.autoscaler.max 10

# Get kubeconfig
linode-cli lke kubeconfig-view 12345 --text | base64 -d > ~/.kube/config

# Or download directly
linode-cli lke kubeconfig-view 12345 > kubeconfig.yaml
export KUBECONFIG=kubeconfig.yaml
```

### API

```bash
curl -X POST https://api.linode.com/v4/lke/clusters \
  -H "Authorization: Bearer $LINODE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "my-cluster",
    "region": "us-east",
    "k8s_version": "1.29",
    "control_plane": {"high_availability": true},
    "node_pools": [{
      "type": "g6-standard-4",
      "count": 3,
      "autoscaler": {"enabled": true, "min": 3, "max": 10}
    }]
  }'
```

### Terraform

```hcl
resource "linode_lke_cluster" "main" {
  label       = "my-cluster"
  k8s_version = "1.29"
  region      = "us-east"

  control_plane {
    high_availability = true
  }

  pool {
    type  = "g6-standard-4"
    count = 3

    autoscaler {
      min = 3
      max = 10
    }
  }
}

# Output kubeconfig
output "kubeconfig" {
  value     = base64decode(linode_lke_cluster.main.kubeconfig)
  sensitive = true
}
```

## Kubernetes Versions

| Version | Status |
|---------|--------|
| 1.29 | Latest |
| 1.28 | Supported |
| 1.27 | Supported |
| 1.26 | Deprecated |

```bash
# List available versions
linode-cli lke versions-list

# Upgrade cluster
linode-cli lke cluster-update 12345 --k8s_version 1.29
```

## Node Pools

### Add Node Pool

```bash
# Add node pool
linode-cli lke pool-create 12345 \
  --type g6-dedicated-8 \
  --count 2 \
  --autoscaler.enabled true \
  --autoscaler.min 2 \
  --autoscaler.max 5

# List node pools
linode-cli lke pools-list 12345

# Scale node pool
linode-cli lke pool-update 12345 54321 --count 5

# Delete node pool
linode-cli lke pool-delete 12345 54321
```

### Node Pool Recommendations

| Workload | Recommended Plan |
|----------|------------------|
| General | g6-standard-4 or g6-standard-8 |
| CPU-intensive | g6-dedicated-8, g6-dedicated-16 |
| Memory-intensive | g6-highmem-8, g6-highmem-16 |
| ML/AI | g6-gpu-rtx |
| Production | Dedicated CPU plans |

## Cluster Autoscaler

```bash
# Enable autoscaler on pool
linode-cli lke pool-update 12345 54321 \
  --autoscaler.enabled true \
  --autoscaler.min 3 \
  --autoscaler.max 10
```

```
Autoscaler Behavior:
+-- Scale Up: When pods are pending due to insufficient resources
+-- Scale Down: When nodes are underutilized (< 50% for 10+ min)
+-- Cooldown: 5 minutes between scale operations
+-- Node deletion: Graceful with pod eviction
```

## High Availability

```bash
# Enable HA control plane (new cluster)
linode-cli lke cluster-create \
  --label ha-cluster \
  --region us-east \
  --k8s_version 1.29 \
  --control_plane.high_availability true \
  --node_pools.type g6-standard-4 \
  --node_pools.count 3

# Enable HA on existing cluster (costs $60/mo)
linode-cli lke cluster-update 12345 \
  --control_plane.high_availability true
```

| HA Feature | Description |
|------------|-------------|
| Multi-master | 3 API server replicas |
| etcd cluster | Distributed key-value store |
| Uptime SLA | 99.99% |
| Cost | $60/mo additional |

## Load Balancer (NodeBalancer)

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    service.beta.kubernetes.io/linode-loadbalancer-throttle: "4"
    service.beta.kubernetes.io/linode-loadbalancer-check-type: "http"
    service.beta.kubernetes.io/linode-loadbalancer-check-path: "/health"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: my-app
```

### NodeBalancer Annotations

| Annotation | Description |
|------------|-------------|
| `linode-loadbalancer-throttle` | Connections per second limit |
| `linode-loadbalancer-check-type` | Health check type (http, tcp) |
| `linode-loadbalancer-check-path` | HTTP health check path |
| `linode-loadbalancer-check-interval` | Check interval (seconds) |
| `linode-loadbalancer-proxy-protocol` | Enable proxy protocol (v1, v2) |

## Persistent Storage

### Block Storage CSI

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: linode-block-storage-retain
```

### Storage Classes

| StorageClass | Reclaim Policy |
|--------------|----------------|
| linode-block-storage | Delete |
| linode-block-storage-retain | Retain |

```yaml
# Using in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: app
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: my-pvc
```

## Container Registry

Use Docker Hub, GitHub Container Registry, or private registry.

```bash
# Create registry secret
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=$GITHUB_USER \
  --docker-password=$GITHUB_TOKEN \
  --docker-email=user@example.com

# Use in deployment
# spec.template.spec.imagePullSecrets:
#   - name: regcred
```

## Ingress Controller

### NGINX Ingress

```bash
# Install NGINX Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# Wait for LoadBalancer IP
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

## Monitoring

### Metrics Server

```bash
# Usually pre-installed, but if needed:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check metrics
kubectl top nodes
kubectl top pods
```

### Prometheus + Grafana

```bash
# Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

## CLI Quick Reference

```bash
# Clusters
linode-cli lke clusters-list
linode-cli lke cluster-view 12345
linode-cli lke cluster-create --label my-cluster --region us-east --k8s_version 1.29
linode-cli lke cluster-delete 12345

# Kubeconfig
linode-cli lke kubeconfig-view 12345 --text | base64 -d > kubeconfig.yaml
export KUBECONFIG=kubeconfig.yaml

# Node pools
linode-cli lke pools-list 12345
linode-cli lke pool-create 12345 --type g6-standard-4 --count 3
linode-cli lke pool-update 12345 54321 --count 5
linode-cli lke pool-delete 12345 54321

# Versions
linode-cli lke versions-list
```

## Best Practices

```
1. High Availability
   +-- Enable HA control plane for production
   +-- Spread workers across multiple node pools
   +-- Use pod disruption budgets

2. Security
   +-- Use RBAC for access control
   +-- Enable network policies
   +-- Regularly update Kubernetes version
   +-- Use secrets for sensitive data

3. Cost Optimization
   +-- Enable cluster autoscaler
   +-- Right-size node pools
   +-- Use spot instances for fault-tolerant workloads

4. Monitoring
   +-- Deploy Prometheus/Grafana
   +-- Set up alerting
   +-- Monitor resource utilization
```

## Gotchas

- Control plane IP changes on upgrade (use kubeconfig refresh)
- Node pools cannot span regions
- GPU nodes limited to specific regions
- Maximum 250 nodes per cluster
- etcd backup not user-accessible
- No built-in container registry
- Persistent volumes are region-specific
- NodeBalancer costs $10/mo per Service type: LoadBalancer

## Limits

| Resource | Limit |
|----------|-------|
| Clusters per account | 100 |
| Node pools per cluster | 200 |
| Nodes per pool | 100 |
| Nodes per cluster | 250 |
| Services (LoadBalancer) | 20 per cluster |
| Persistent volumes | Based on Block Storage limits |
