# Google Kubernetes Engine (GKE)

> Managed Kubernetes service for running containerized applications at scale.

## Overview

Google Kubernetes Engine (GKE) is a managed, production-ready environment for running containerized applications. GKE provides the operational power of Kubernetes while managing many of the underlying infrastructure details.

## Key Concepts

| Term | Definition |
|------|------------|
| Cluster | Set of nodes running containerized applications |
| Control Plane | Manages cluster (API server, scheduler, etc.) |
| Node | Worker machine (VM) running pods |
| Node Pool | Group of nodes with same configuration |
| Pod | Smallest deployable unit (1+ containers) |
| Workload | Application running on the cluster |

## Cluster Modes

| Mode | Control Plane | Nodes | Use Case |
|------|---------------|-------|----------|
| **Autopilot** | Managed | Managed (pay per pod) | Simplified operations |
| **Standard** | Managed | You manage | Full control |

### Autopilot vs Standard

| Feature | Autopilot | Standard |
|---------|-----------|----------|
| Node management | Automatic | Manual |
| Node provisioning | Automatic | You configure |
| Pricing | Per pod resources | Per node |
| Security | Pre-configured | You configure |
| Node access | Limited | Full SSH |
| Customization | Limited | Full |

## Architecture

```
+---------------------------------------------------------------+
|                         GKE Cluster                           |
|                                                               |
|  +---------------------------------------------------------+  |
|  |             Control Plane (Google Managed)              |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  |  |API Server| |Scheduler | |Controller| |   etcd   |    |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                              v                                |
|  +---------------------------------------------------------+  |
|  |                       Node Pools                        |  |
|  |                                                         |  |
|  |  +---------------------+    +---------------------+     |  |
|  |  | Node Pool: default  |    | Node Pool: gpu-pool |     |  |
|  |  | e2-standard-4       |    | n1-standard-8 + GPU |     |  |
|  |  | +----+ +----+ +----+|    | +----+ +----+       |     |  |
|  |  | |Node| |Node| |Node||    | |Node| |Node|       |     |  |
|  |  | |[P] | |[P] | |[P] ||    | |[P] | |[P] |       |     |  |
|  |  | +----+ +----+ +----+|    | +----+ +----+       |     |  |
|  |  +---------------------+    +---------------------+     |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

## Create Cluster

### Standard Cluster

```bash
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --enable-autorepair \
  --enable-autoupgrade

# Get credentials
gcloud container clusters get-credentials my-cluster --zone=us-central1-a
```

### Autopilot Cluster

```bash
gcloud container clusters create-auto my-autopilot \
  --region=us-central1

# Get credentials
gcloud container clusters get-credentials my-autopilot --region=us-central1
```

### Regional Cluster (HA)

```bash
gcloud container clusters create my-regional-cluster \
  --region=us-central1 \
  --num-nodes=1 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5
```

## Node Pools

```bash
# Create node pool
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --machine-type=n1-standard-8 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --num-nodes=2 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=5

# Create spot node pool
gcloud container node-pools create spot-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --spot \
  --machine-type=e2-standard-4 \
  --num-nodes=3

# Update node pool
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10

# Delete node pool
gcloud container node-pools delete old-pool \
  --cluster=my-cluster \
  --zone=us-central1-a
```

## Networking

### VPC-Native Clusters

```bash
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --enable-ip-alias \
  --network=my-vpc \
  --subnetwork=my-subnet \
  --cluster-secondary-range-name=pods \
  --services-secondary-range-name=services
```

### Private Clusters

```bash
gcloud container clusters create private-cluster \
  --zone=us-central1-a \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-ip-alias \
  --network=my-vpc \
  --subnetwork=my-subnet
```

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

```bash
# Enable network policy enforcement
gcloud container clusters update my-cluster \
  --zone=us-central1-a \
  --enable-network-policy
```

## Workload Identity

Connect Kubernetes service accounts to GCP IAM.

```bash
# Enable Workload Identity
gcloud container clusters update my-cluster \
  --zone=us-central1-a \
  --workload-pool=my-project.svc.id.goog

# Create GCP service account
gcloud iam service-accounts create my-gsa

# Bind KSA to GSA
gcloud iam service-accounts add-iam-policy-binding my-gsa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project.svc.id.goog[my-namespace/my-ksa]"
```

```yaml
# Annotate Kubernetes service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-ksa
  annotations:
    iam.gke.io/gcp-service-account: my-gsa@my-project.iam.gserviceaccount.com
```

## GKE Ingress

### HTTP(S) Load Balancer

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api/*
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: my-cert
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        value: /api
    backendRefs:
    - name: api-service
      port: 80
```

## Autoscaling

### Cluster Autoscaler

```bash
# Enable on node pool
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10
```

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  updatePolicy:
    updateMode: "Auto"
```

## Security

### Binary Authorization

```bash
# Enable Binary Authorization
gcloud container clusters update my-cluster \
  --zone=us-central1-a \
  --enable-binauthz
```

### Shielded GKE Nodes

```bash
gcloud container clusters create secure-cluster \
  --zone=us-central1-a \
  --shielded-secure-boot \
  --shielded-integrity-monitoring
```

### GKE Sandbox (gVisor)

```bash
gcloud container node-pools create sandbox-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --sandbox=type=gvisor
```

## CLI Quick Reference

```bash
# Create cluster
gcloud container clusters create my-cluster --zone=us-central1-a

# Get credentials
gcloud container clusters get-credentials my-cluster --zone=us-central1-a

# List clusters
gcloud container clusters list

# Describe cluster
gcloud container clusters describe my-cluster --zone=us-central1-a

# Resize node pool
gcloud container clusters resize my-cluster --zone=us-central1-a --num-nodes=5

# Upgrade cluster
gcloud container clusters upgrade my-cluster --zone=us-central1-a --master
gcloud container clusters upgrade my-cluster --zone=us-central1-a --node-pool=default-pool

# Delete cluster
gcloud container clusters delete my-cluster --zone=us-central1-a

# kubectl commands
kubectl get nodes
kubectl get pods -A
kubectl apply -f manifest.yaml
kubectl logs pod-name
kubectl exec -it pod-name -- /bin/bash
```

## Pricing

| Component | Standard | Autopilot |
|-----------|----------|-----------|
| Cluster management | $0.10/hr | $0.10/hr |
| Nodes | VM pricing | Pod resource pricing |
| Zonal/Regional | Zonal cheaper | Regional only |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Autopilot**: Pay per pod, Google manages nodes
2. **Standard**: Pay per node, full control
3. **Regional cluster**: Control plane across 3 zones
4. **Private cluster**: No public IPs on nodes
5. **VPC-native**: Use alias IPs for pods
6. **Workload Identity**: Connect KSA to GCP IAM
7. **Network Policy**: Requires enabling on cluster
8. **Node auto-repair**: Automatic unhealthy node replacement
9. **Release channels**: Rapid, Regular, Stable
10. **Binary Authorization**: Enforce image signing

## Gotchas

- Zonal clusters have single point of failure (control plane)
- Private clusters need Cloud NAT for outbound
- Autopilot has pod resource minimums
- Node pools cannot change machine type (create new)
- Workload Identity requires cluster-level enable
- Network policies need Calico/Cilium
- VPC-native required for private clusters
- Upgrades can cause workload disruption
- Pod IP ranges cannot be changed after creation
- Autopilot doesn't support all features (DaemonSets limited)

## Limits

| Resource | Limit |
|----------|-------|
| Clusters per zone/region | 50 |
| Nodes per cluster | 15,000 |
| Nodes per node pool | 1,000 |
| Pods per node | 110 |
| Node pools per cluster | 100 |
| Services per cluster | 10,000 |
| Pods per cluster | 200,000 |
| Containers per pod | No hard limit |
