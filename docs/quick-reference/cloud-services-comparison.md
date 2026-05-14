# Cloud Services Comparison: AWS vs GCP vs Azure

> Quick reference to find equivalent services across cloud providers.

---

## Compute

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Virtual Machines | EC2 | Compute Engine | Virtual Machines | On-demand VMs |
| Managed VMs | Lightsail | — | — | Simplified VM management |
| Spot/Preemptible | Spot Instances | Preemptible VMs / Spot VMs | Spot VMs | Discounted interruptible instances |
| Dedicated Hosts | Dedicated Hosts | Sole-tenant Nodes | Dedicated Hosts | Physical server isolation |
| Auto Scaling | Auto Scaling Groups | Managed Instance Groups | VM Scale Sets | Automatic capacity adjustment |
| Batch Processing | AWS Batch | Batch | Azure Batch | Job scheduling & compute |

---

## Containers & Orchestration

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Managed Kubernetes | EKS | GKE | AKS | Managed K8s control plane |
| Container Service | ECS | Cloud Run (containers) | Container Instances | Run containers without K8s |
| Serverless Containers | Fargate | Cloud Run | Container Apps | Serverless container execution |
| Container Registry | ECR | Artifact Registry | Container Registry | Docker image storage |
| App Platform | App Runner | Cloud Run | Container Apps | Deploy from source/image |

---

## Serverless / Functions

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Functions | Lambda | Cloud Functions | Azure Functions | Event-driven serverless code |
| Step Functions | Step Functions | Workflows | Logic Apps / Durable Functions | Orchestrate serverless workflows |
| API Gateway | API Gateway | API Gateway | API Management | Manage & expose APIs |
| Event Bus | EventBridge | Eventarc | Event Grid | Event routing & filtering |

---

## Storage

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Object Storage | S3 | Cloud Storage | Blob Storage | Scalable object store |
| Archive Storage | S3 Glacier | Archive Storage | Archive Storage | Long-term cold storage |
| Block Storage | EBS | Persistent Disk | Managed Disks | VM-attached block storage |
| File Storage (NFS) | EFS | Filestore | Azure Files | Managed NFS/SMB |
| File Storage (SMB) | FSx for Windows | — | Azure Files | Windows file shares |
| High-perf File | FSx for Lustre | — | Azure NetApp Files | HPC file systems |
| Hybrid Storage | Storage Gateway | — | StorSimple | On-prem to cloud bridge |

---

## Databases - Relational

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Managed MySQL/PostgreSQL | RDS | Cloud SQL | Azure Database for MySQL/PostgreSQL | Managed open-source RDBMS |
| Managed SQL Server | RDS for SQL Server | Cloud SQL for SQL Server | Azure SQL Database | Managed SQL Server |
| Serverless SQL | Aurora Serverless | AlloyDB / Cloud SQL | Azure SQL Serverless | Auto-scaling relational DB |
| Enterprise RDBMS | Aurora | AlloyDB / Spanner | Azure SQL Hyperscale | High-performance relational |
| Global Distribution | Aurora Global | Spanner | Cosmos DB (SQL API) | Multi-region relational |

---

## Databases - NoSQL

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Key-Value / Document | DynamoDB | Firestore / Bigtable | Cosmos DB | Managed NoSQL |
| Wide Column | Keyspaces (Cassandra) | Bigtable | Cosmos DB (Cassandra API) | Wide-column store |
| Document DB | DocumentDB | Firestore | Cosmos DB (MongoDB API) | Document database |
| In-Memory Cache | ElastiCache | Memorystore | Azure Cache for Redis | Redis/Memcached |
| Graph Database | Neptune | — | Cosmos DB (Gremlin API) | Graph queries |
| Time Series | Timestream | — | Azure Data Explorer | Time-series data |

---

## Networking - Core

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Virtual Network | VPC | VPC | VNet | Isolated network |
| Subnets | Subnets | Subnets | Subnets | Network segmentation |
| Internet Gateway | Internet Gateway | Cloud NAT + Router | Internet Gateway | Internet access |
| NAT | NAT Gateway | Cloud NAT | NAT Gateway | Outbound internet for private |
| Peering | VPC Peering | VPC Peering | VNet Peering | Connect VPCs/VNets |
| Transit | Transit Gateway | Network Connectivity Center | Virtual WAN | Hub-spoke networking |
| Private Link | PrivateLink | Private Service Connect | Private Link | Private service access |

---

## Networking - Load Balancing

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Application LB (L7) | ALB | Global HTTP(S) LB | Application Gateway | HTTP/HTTPS load balancing |
| Network LB (L4) | NLB | Regional TCP/UDP LB | Load Balancer | TCP/UDP load balancing |
| Classic/Basic | CLB (legacy) | — | Basic Load Balancer | Legacy LB |
| Global LB | Global Accelerator | Global LB | Front Door | Global traffic distribution |

---

## Networking - DNS & CDN

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| DNS | Route 53 | Cloud DNS | Azure DNS | Managed DNS |
| CDN | CloudFront | Cloud CDN | Azure CDN / Front Door | Content delivery |
| DDoS Protection | Shield | Cloud Armor | DDoS Protection | DDoS mitigation |
| WAF | WAF | Cloud Armor | Web Application Firewall | App-layer firewall |

---

## Networking - Hybrid & VPN

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| VPN | Site-to-Site VPN | Cloud VPN | VPN Gateway | IPsec tunnels |
| Direct Connection | Direct Connect | Cloud Interconnect | ExpressRoute | Dedicated private connection |
| Client VPN | Client VPN | — | Point-to-Site VPN | Remote user VPN |

---

## Security & Identity

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| IAM | IAM | Cloud IAM | Azure AD / Entra ID | Identity management |
| SSO | IAM Identity Center | Cloud Identity | Azure AD SSO | Single sign-on |
| Secrets | Secrets Manager | Secret Manager | Key Vault (Secrets) | Secret storage |
| Key Management | KMS | Cloud KMS | Key Vault (Keys) | Encryption keys |
| Certificate Manager | ACM | Certificate Manager | App Service Certificates | SSL/TLS certs |
| Security Hub | Security Hub | Security Command Center | Microsoft Defender for Cloud | Security posture |
| Compliance | Artifact | Compliance Reports | Compliance Manager | Compliance docs |

---

## Messaging & Integration

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Message Queue | SQS | Cloud Tasks | Queue Storage / Service Bus | Message queuing |
| Pub/Sub | SNS | Pub/Sub | Service Bus Topics | Publish-subscribe |
| Streaming | Kinesis | Pub/Sub + Dataflow | Event Hubs | Real-time streaming |
| Kafka Managed | MSK | — | Event Hubs (Kafka) | Managed Kafka |
| Workflow | Step Functions | Workflows | Logic Apps | Orchestration |

---

## Analytics & Big Data

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Data Warehouse | Redshift | BigQuery | Synapse Analytics | Analytical queries |
| Data Lake | S3 + Lake Formation | Cloud Storage + BigLake | Data Lake Storage | Raw data storage |
| ETL | Glue | Dataflow / Dataproc | Data Factory | Data transformation |
| Hadoop/Spark | EMR | Dataproc | HDInsight | Big data processing |
| Real-time Analytics | Kinesis Analytics | Dataflow | Stream Analytics | Streaming analytics |
| BI / Visualization | QuickSight | Looker | Power BI | Business intelligence |

---

## AI & Machine Learning

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| ML Platform | SageMaker | Vertex AI | Azure ML | End-to-end ML |
| Pre-trained APIs | Rekognition, Comprehend, etc. | Vision AI, NLP, etc. | Cognitive Services | AI APIs |
| Speech | Transcribe, Polly | Speech-to-Text, Text-to-Speech | Speech Services | Voice AI |
| Translation | Translate | Translation AI | Translator | Language translation |
| Chatbots | Lex | Dialogflow | Bot Service | Conversational AI |
| LLM / GenAI | Bedrock | Vertex AI (Gemini) | Azure OpenAI | Foundation models |

---

## DevOps & Developer Tools

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Source Control | CodeCommit | Cloud Source Repos | Azure Repos | Git hosting |
| CI/CD | CodePipeline + CodeBuild | Cloud Build | Azure Pipelines | Build & deploy |
| Artifact Storage | CodeArtifact | Artifact Registry | Azure Artifacts | Package management |
| IaC | CloudFormation | Deployment Manager | ARM / Bicep | Infrastructure as code |
| CLI | AWS CLI | gcloud CLI | Azure CLI | Command-line tools |
| Cloud Shell | CloudShell | Cloud Shell | Cloud Shell | Browser-based terminal |

---

## Monitoring & Observability

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Monitoring | CloudWatch | Cloud Monitoring | Azure Monitor | Metrics & dashboards |
| Logging | CloudWatch Logs | Cloud Logging | Log Analytics | Centralized logging |
| Tracing | X-Ray | Cloud Trace | Application Insights | Distributed tracing |
| Alerting | CloudWatch Alarms | Cloud Alerting | Azure Alerts | Notification & alerts |

---

## Management & Governance

| Category | AWS | GCP | Azure | Description |
|----------|-----|-----|-------|-------------|
| Multi-Account | Organizations | Organization | Management Groups | Account hierarchy |
| Cost Management | Cost Explorer | Cost Management | Cost Management | Billing & budgets |
| Resource Tagging | Tags | Labels | Tags | Resource organization |
| Policy | Service Control Policies | Organization Policies | Azure Policy | Governance rules |
| Config Audit | Config | — | Azure Policy | Configuration compliance |

---

## Quick Tips by Certification

| Certification | Focus Areas |
|--------------|-------------|
| **AWS Solutions Architect** | VPC, EC2, S3, RDS, Lambda, IAM, CloudFront, Route 53 |
| **AWS Developer** | Lambda, API Gateway, DynamoDB, SQS/SNS, CodePipeline |
| **AWS SysOps** | CloudWatch, Auto Scaling, Systems Manager, Config |
| **GCP Cloud Architect** | VPC, GCE, GCS, Cloud SQL, BigQuery, IAM, GKE |
| **GCP Cloud Engineer** | Compute Engine, Cloud Storage, IAM, gcloud CLI |
| **Azure Administrator** | VMs, VNet, Storage, Azure AD, Monitor, ARM |
| **Azure Solutions Architect** | All above + Cosmos DB, AKS, Front Door, Governance |

---

## Notes

- Services evolve rapidly; verify current offerings in official docs
- Pricing models differ significantly between providers
- Feature parity is approximate; always check specific capabilities
