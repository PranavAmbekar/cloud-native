# Amazon ECS (Elastic Container Service)

> Fully managed container orchestration service for running Docker containers.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Cluster | Logical grouping of tasks/services |
| Task Definition | Blueprint for containers (like Dockerfile) |
| Task | Running instance of task definition |
| Service | Maintains desired count of tasks |
| Container Instance | EC2 instance running ECS agent |
| Capacity Provider | EC2 or Fargate compute |

---

## Launch Types

### EC2 Launch Type
```
┌─────────────────────────────────────────────────────────────┐
│                       ECS Cluster                           │
│  ┌────────────────────┐    ┌────────────────────┐          │
│  │   EC2 Instance     │    │   EC2 Instance     │          │
│  │   (ECS Agent)      │    │   (ECS Agent)      │          │
│  │  ┌──────┐ ┌──────┐ │    │  ┌──────┐ ┌──────┐ │          │
│  │  │Task 1│ │Task 2│ │    │  │Task 3│ │Task 4│ │          │
│  │  └──────┘ └──────┘ │    │  └──────┘ └──────┘ │          │
│  └────────────────────┘    └────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

- You manage EC2 instances
- Full control over infrastructure
- Pay for EC2 instances

### Fargate Launch Type
```
┌─────────────────────────────────────────────────────────────┐
│                       ECS Cluster                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Task 1  │  │  Task 2  │  │  Task 3  │  │  Task 4  │   │
│  │(Fargate) │  │(Fargate) │  │(Fargate) │  │(Fargate) │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                             │
│            (No EC2 management - serverless)                │
└─────────────────────────────────────────────────────────────┘
```

- Serverless - no EC2 management
- Pay per task vCPU/memory
- Each task has own isolation

---

## Task Definition

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::xxx:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::xxx:role/myTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        {"name": "ENV", "value": "production"}
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:xxx:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Key Settings
| Setting | Description |
|---------|-------------|
| family | Task definition name |
| networkMode | awsvpc (recommended), bridge, host |
| cpu/memory | Task-level resources |
| executionRoleArn | For ECS agent (pull images, logs) |
| taskRoleArn | For container code (AWS API calls) |
| containerDefinitions | Container configurations |

---

## IAM Roles

### Task Execution Role
What ECS agent needs:
- Pull images from ECR
- Send logs to CloudWatch
- Retrieve secrets

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

### Task Role
What your application needs:
- S3 access
- DynamoDB access
- Any AWS API calls from code

---

## Network Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| awsvpc | Task gets own ENI | Fargate, recommended |
| bridge | Docker bridge network | EC2, port mapping |
| host | Use host network | EC2, high performance |
| none | No networking | Batch jobs |

### awsvpc Mode
```
┌─────────────────────────────────────────┐
│              VPC Subnet                  │
│  ┌─────────────┐    ┌─────────────┐     │
│  │   Task 1    │    │   Task 2    │     │
│  │ ENI: 10.0.1.5│   │ ENI: 10.0.1.6│    │
│  │ SG: sg-xxx  │    │ SG: sg-xxx  │     │
│  └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────┘
```

- Each task gets private IP
- Apply Security Groups per task
- Required for Fargate

---

## ECS Services

Maintain desired count of tasks.

```yaml
Service: my-service
├── Desired Count: 3
├── Task Definition: my-app:5
├── Launch Type: FARGATE
├── Network:
│   ├── Subnets: [subnet-a, subnet-b]
│   └── Security Groups: [sg-xxx]
├── Load Balancer:
│   ├── Target Group: my-tg
│   └── Container: app:8080
└── Auto Scaling:
    ├── Min: 2
    ├── Max: 10
    └── Policy: Target CPU 70%
```

### Deployment Strategies
| Strategy | Description |
|----------|-------------|
| Rolling Update | Replace tasks gradually |
| Blue/Green | Deploy to new target group |
| External | Third-party controller |

### Rolling Update Settings
```
minimumHealthyPercent: 50
maximumPercent: 200

Desired: 4 tasks
- Min healthy: 2 (50%)
- Max running: 8 (200%)
- Replaces 2 at a time
```

---

## Service Discovery

Automatic DNS registration.

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloud Map Namespace                       │
│                   (internal.myapp.local)                    │
│                                                             │
│   Service: api                    Service: worker           │
│   DNS: api.internal.myapp.local   DNS: worker.internal...   │
│        │                               │                    │
│   ┌────┴────┐                    ┌────┴────┐               │
│   │ Task A  │                    │ Task X  │               │
│   │10.0.1.5 │                    │10.0.1.8 │               │
│   │ Task B  │                    │ Task Y  │               │
│   │10.0.1.6 │                    │10.0.1.9 │               │
│   └─────────┘                    └─────────┘               │
└─────────────────────────────────────────────────────────────┘
```

- DNS A records or SRV records
- Automatic registration/deregistration
- Health checks integration

---

## Auto Scaling

### Service Auto Scaling
```
CloudWatch Metric ──▶ Scaling Policy ──▶ Adjust Desired Count
```

Scaling policies:
- Target Tracking (maintain CPU at 70%)
- Step Scaling (if CPU > 80%, add 2)
- Scheduled Scaling (scale up at 9am)

### Cluster Auto Scaling (EC2)
```
ECS Capacity Provider ──▶ Auto Scaling Group
                              │
                    Add/remove EC2 instances
```

---

## Load Balancer Integration

```
┌─────────────────────────────────────────────────────────────┐
│                           ALB                               │
│                            │                                │
│              ┌─────────────┴─────────────┐                 │
│              ▼                           ▼                  │
│        Target Group 1              Target Group 2          │
│        (path: /api/*)             (path: /web/*)           │
│              │                           │                  │
│    ┌─────────┴─────────┐       ┌────────┴────────┐        │
│    ▼         ▼         ▼       ▼        ▼        ▼        │
│  Task 1   Task 2    Task 3   Task 4  Task 5   Task 6      │
│  (API)    (API)     (API)    (Web)   (Web)    (Web)       │
└─────────────────────────────────────────────────────────────┘
```

- ALB/NLB supported
- Dynamic port mapping (EC2)
- Health checks from ALB

---

## Logging

### awslogs Driver
```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/my-app",
    "awslogs-region": "us-east-1",
    "awslogs-stream-prefix": "ecs"
  }
}
```

### Other Drivers
- splunk
- fluentd
- awsfirelens (custom routing)

---

## CLI Quick Reference

```bash
# Create cluster
aws ecs create-cluster --cluster-name my-cluster

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Run task
aws ecs run-task \
  --cluster my-cluster \
  --task-definition my-app:1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"

# Create service
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-service \
  --task-definition my-app:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx]}"

# Update service
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --task-definition my-app:2

# List services
aws ecs list-services --cluster my-cluster

# Describe tasks
aws ecs describe-tasks --cluster my-cluster --tasks task-arn

# Stop task
aws ecs stop-task --cluster my-cluster --task task-arn

# Execute command (ECS Exec)
aws ecs execute-command \
  --cluster my-cluster \
  --task task-arn \
  --container app \
  --interactive \
  --command "/bin/sh"
```

---

## ECS vs EKS

| Feature | ECS | EKS |
|---------|-----|-----|
| Orchestrator | AWS native | Kubernetes |
| Learning curve | Lower | Higher |
| Portability | AWS only | Multi-cloud |
| Cost | Free (pay compute) | $0.10/hr + compute |
| Ecosystem | AWS native | K8s ecosystem |
| Task definition | JSON | YAML (K8s manifests) |

---

## Pricing

| Component | Cost |
|-----------|------|
| ECS | Free |
| EC2 | Standard EC2 pricing |
| Fargate vCPU | $0.04048/hour |
| Fargate Memory | $0.004445/GB/hour |

### Fargate Spot
- Up to 70% discount
- Can be interrupted
- Good for fault-tolerant workloads

---

## Best Practices

1. **Use Fargate** for simplicity
2. **awsvpc network mode** for security
3. **Separate execution and task roles**
4. **Use Secrets Manager** for sensitive data
5. **Enable container insights** for monitoring
6. **Use service auto scaling**
7. **Set resource limits** on containers
8. **Use ECR** for container images
9. **Enable ECS Exec** for debugging
10. **Use capacity providers** for mixed compute

---

## Exam Tips

1. **Task Execution Role** - for ECS agent (pull images, logs)
2. **Task Role** - for container code (AWS API)
3. **Fargate** - serverless, no EC2 management
4. **awsvpc** - required for Fargate, each task gets ENI
5. **Service** - maintains desired count, handles deployments
6. **Capacity Provider** - manages EC2 Auto Scaling or Fargate
7. **Service Discovery** - Cloud Map integration
8. **ECS Exec** - SSH into running containers
9. **Rolling update** - minimumHealthyPercent, maximumPercent
10. **Fargate Spot** - cheaper, interruptible tasks
