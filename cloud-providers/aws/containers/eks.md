# Amazon EKS (Elastic Kubernetes Service)

> Managed Kubernetes service to run K8s on AWS without managing control plane.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Cluster | EKS managed Kubernetes control plane |
| Node | EC2 instance or Fargate running pods |
| Node Group | Group of EC2 nodes (managed or self-managed) |
| Pod | Smallest deployable K8s unit |
| Service | Expose pods via network endpoint |
| Ingress | HTTP/HTTPS routing to services |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        EKS Cluster                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Control Plane (AWS Managed)                    │ │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐                │ │
│  │   │API Server│  │Controller│  │  etcd    │                │ │
│  │   │          │  │ Manager  │  │(encrypted)│               │ │
│  │   └──────────┘  └──────────┘  └──────────┘                │ │
│  │                (Multi-AZ, highly available)                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Data Plane (You Manage)                  │ │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐                │ │
│  │   │  Node 1  │  │  Node 2  │  │  Node 3  │                │ │
│  │   │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │                │ │
│  │   │ │ Pod  │ │  │ │ Pod  │ │  │ │ Pod  │ │                │ │
│  │   │ │ Pod  │ │  │ │ Pod  │ │  │ │ Pod  │ │                │ │
│  │   │ └──────┘ │  │ └──────┘ │  │ └──────┘ │                │ │
│  │   └──────────┘  └──────────┘  └──────────┘                │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Node Types

### Managed Node Groups
```
EKS ──▶ Auto Scaling Group ──▶ EC2 Instances
              (AWS managed)
```
- AWS handles node provisioning
- Automatic updates available
- Uses EC2 Auto Scaling Groups

### Self-Managed Nodes
- You manage EC2 instances
- More control, more responsibility
- Use EKS-optimized AMI

### Fargate
```
┌─────────────────────────────────────────┐
│  Pod 1 (Fargate)    Pod 2 (Fargate)    │
│  ┌─────────────┐    ┌─────────────┐    │
│  │ Container   │    │ Container   │    │
│  │ (isolated)  │    │ (isolated)  │    │
│  └─────────────┘    └─────────────┘    │
│       No EC2 instances to manage        │
└─────────────────────────────────────────┘
```
- Serverless pods
- Per-pod isolation
- Define Fargate Profile

### Comparison
| Feature | Managed | Self-Managed | Fargate |
|---------|---------|--------------|---------|
| Control | Medium | Full | Low |
| Maintenance | Low | High | None |
| Cost | EC2 | EC2 | Per pod |
| DaemonSets | Yes | Yes | No |
| GPU | Yes | Yes | No |

---

## Networking

### VPC CNI Plugin
- Pods get VPC IP addresses
- Native VPC networking
- Security Groups for pods

```
┌─────────────────────────────────────────┐
│              VPC Subnet                  │
│                                         │
│   Node: 10.0.1.10                       │
│   ├── Pod: 10.0.1.11                    │
│   ├── Pod: 10.0.1.12                    │
│   └── Pod: 10.0.1.13                    │
│                                         │
│   Node: 10.0.1.20                       │
│   ├── Pod: 10.0.1.21                    │
│   └── Pod: 10.0.1.22                    │
└─────────────────────────────────────────┘
```

### Service Types
| Type | Description |
|------|-------------|
| ClusterIP | Internal only |
| NodePort | Expose on node port |
| LoadBalancer | Provisions AWS ELB |

### AWS Load Balancer Controller
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  ports:
  - port: 80
```

### Ingress with ALB
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## IAM Integration

### IRSA (IAM Roles for Service Accounts)
```
┌─────────────────────────────────────────┐
│ Pod                                     │
│ ├── ServiceAccount: my-sa              │
│ └── Annotation:                        │
│     eks.amazonaws.com/role-arn: arn:...│
│                    │                    │
│                    ▼                    │
│     Assumes IAM Role via OIDC          │
│     (No credentials in pod)            │
└─────────────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::xxx:role/my-pod-role
```

### Node IAM Role
- Attached to EC2 nodes
- Used for node-level operations
- ECR pull, EC2 describe, etc.

---

## Storage

### EBS CSI Driver
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
```

### EFS CSI Driver
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxx
```

### FSx for Lustre
For high-performance workloads.

---

## Cluster Autoscaler

Adjusts node count based on pending pods.

```
Pending Pod (unschedulable)
         │
         ▼
Cluster Autoscaler
         │
         ▼
Add Node to Node Group
         │
         ▼
Pod Scheduled
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
spec:
  # ... autoscaler deployment config
```

### Karpenter
AWS-developed alternative to Cluster Autoscaler.
- Faster scaling
- Direct EC2 provisioning
- Right-sizing nodes

---

## Fargate Profiles

Define which pods run on Fargate.

```yaml
# Fargate Profile
name: my-profile
selectors:
  - namespace: production
    labels:
      app: web
  - namespace: batch
```

Matching pods → Fargate
Non-matching pods → EC2 nodes

---

## Logging & Monitoring

### Control Plane Logging
```bash
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

Logs to CloudWatch:
- API server
- Audit
- Authenticator
- Controller Manager
- Scheduler

### Container Insights
```bash
# Enable Container Insights
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability
```

### Prometheus/Grafana
- Amazon Managed Prometheus
- Amazon Managed Grafana

---

## Security

### Cluster Security
- Private endpoint (no public access)
- Encryption at rest (etcd)
- Secrets encryption with KMS

### Pod Security
- Pod Security Standards (PSS)
- Network Policies
- Security Groups for pods

### Authentication
- IAM users/roles
- OIDC identity providers
- aws-auth ConfigMap

```yaml
# aws-auth ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::xxx:role/NodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::xxx:user/admin
      username: admin
      groups:
        - system:masters
```

---

## CLI Quick Reference

```bash
# Create cluster
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name standard \
  --node-type t3.medium \
  --nodes 3

# Or with AWS CLI
aws eks create-cluster \
  --name my-cluster \
  --role-arn arn:aws:iam::xxx:role/EKSClusterRole \
  --resources-vpc-config subnetIds=subnet-xxx,securityGroupIds=sg-xxx

# Update kubeconfig
aws eks update-kubeconfig --name my-cluster

# Create node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes \
  --node-role arn:aws:iam::xxx:role/NodeRole \
  --subnets subnet-xxx subnet-yyy \
  --instance-types t3.medium

# Create Fargate profile
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name my-profile \
  --pod-execution-role-arn arn:aws:iam::xxx:role/FargatePodRole \
  --subnets subnet-xxx \
  --selectors namespace=production

# List clusters
aws eks list-clusters

# Describe cluster
aws eks describe-cluster --name my-cluster

# List addons
aws eks list-addons --cluster-name my-cluster

# Install addon
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni
```

---

## Add-ons

| Add-on | Purpose |
|--------|---------|
| vpc-cni | VPC networking |
| coredns | Cluster DNS |
| kube-proxy | Network proxy |
| aws-ebs-csi-driver | EBS volumes |
| aws-efs-csi-driver | EFS volumes |
| adot | OpenTelemetry |

---

## Pricing

| Component | Cost |
|-----------|------|
| EKS Cluster | $0.10/hour (~$73/month) |
| EC2 Nodes | Standard EC2 pricing |
| Fargate | vCPU + Memory |

---

## EKS vs ECS

| Feature | EKS | ECS |
|---------|-----|-----|
| Orchestrator | Kubernetes | AWS native |
| Learning curve | Higher | Lower |
| Portability | Multi-cloud | AWS only |
| Cluster cost | $0.10/hr | Free |
| Ecosystem | K8s ecosystem | AWS native |
| Complexity | Higher | Lower |

---

## Exam Tips

1. **Control plane** - fully managed, multi-AZ
2. **Managed node groups** - easiest node management
3. **Fargate** - serverless pods, no DaemonSets
4. **IRSA** - IAM roles for pods via service accounts
5. **VPC CNI** - pods get VPC IPs
6. **AWS Load Balancer Controller** - creates ALB/NLB
7. **Cluster Autoscaler** - scales nodes
8. **Karpenter** - faster, smarter autoscaling
9. **aws-auth ConfigMap** - IAM to K8s RBAC mapping
10. **Private cluster** - no public endpoint
11. **Add-ons** - managed K8s components
12. **eksctl** - simplest way to create clusters
