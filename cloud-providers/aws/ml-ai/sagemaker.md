# Amazon SageMaker

> Fully managed service to build, train, and deploy machine learning models.

---

## Key Concepts

| Term | Definition |
|------|------------|
| Notebook Instance | Jupyter environment for development |
| Training Job | Process to train a model |
| Model | Trained ML artifact |
| Endpoint | Deployed model for inference |
| Pipeline | Automated ML workflow |
| Feature Store | Centralized feature repository |
| Model Registry | Versioned model catalog |

---

## ML Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    SageMaker ML Workflow                        │
│                                                                 │
│   1. PREPARE          2. BUILD           3. TRAIN & TUNE       │
│   ┌──────────┐       ┌──────────┐       ┌──────────┐           │
│   │Ground    │       │Studio    │       │Training  │           │
│   │Truth     │──────▶│Notebooks │──────▶│Jobs      │           │
│   │(labeling)│       │          │       │          │           │
│   └──────────┘       └──────────┘       └──────────┘           │
│        │                  │                  │                  │
│        ▼                  ▼                  ▼                  │
│   ┌──────────┐       ┌──────────┐       ┌──────────┐           │
│   │Data      │       │Feature   │       │Hyper-    │           │
│   │Wrangler  │       │Store     │       │parameter │           │
│   │          │       │          │       │Tuning    │           │
│   └──────────┘       └──────────┘       └──────────┘           │
│                                              │                  │
│   4. DEPLOY           5. MONITOR                               │
│   ┌──────────┐       ┌──────────┐                              │
│   │Endpoints │◀──────│Model     │                              │
│   │(real-time│       │Registry  │                              │
│   │ batch)   │       │          │                              │
│   └──────────┘       └──────────┘                              │
│        │                                                        │
│        ▼                                                        │
│   ┌──────────┐                                                  │
│   │Model     │                                                  │
│   │Monitor   │                                                  │
│   └──────────┘                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## SageMaker Studio

Integrated IDE for ML development.

### Components
- JupyterLab notebooks
- Code editor (VS Code-like)
- Data Wrangler
- Feature Store
- Experiments
- Pipelines
- Model Registry
- Debugger

### Studio vs Notebook Instances

| Feature | Studio | Notebook Instance |
|---------|--------|-------------------|
| Interface | Full IDE | JupyterLab only |
| Collaboration | Yes | Limited |
| Git integration | Built-in | Manual |
| Cost | Per-user + compute | Per-instance |
| Persistence | EFS-backed | EBS-backed |

---

## Training

### Built-in Algorithms
| Algorithm | Type | Use Case |
|-----------|------|----------|
| XGBoost | Classification/Regression | Structured data |
| Linear Learner | Classification/Regression | Linear models |
| K-Means | Clustering | Grouping |
| Random Cut Forest | Anomaly Detection | Outliers |
| DeepAR | Forecasting | Time series |
| BlazingText | NLP | Word embeddings, text classification |
| Image Classification | CV | Image labels |
| Object Detection | CV | Detect objects |
| Semantic Segmentation | CV | Pixel-level classification |

### Training Job

```python
import sagemaker
from sagemaker.estimator import Estimator

estimator = Estimator(
    image_uri='123456789.dkr.ecr.us-east-1.amazonaws.com/my-algorithm',
    role='arn:aws:iam::xxx:role/SageMakerRole',
    instance_count=1,
    instance_type='ml.m5.xlarge',
    output_path='s3://my-bucket/output',
    hyperparameters={
        'epochs': 10,
        'learning_rate': 0.01
    }
)

estimator.fit({'training': 's3://my-bucket/train', 'validation': 's3://my-bucket/val'})
```

### Input Modes
| Mode | Description | Use Case |
|------|-------------|----------|
| File | Download to instance | Small datasets |
| Pipe | Stream from S3 | Large datasets |
| FastFile | Stream with caching | Repeated access |

### Spot Training
- Up to 90% cost savings
- Use checkpointing for interruptions

```python
estimator = Estimator(
    ...
    use_spot_instances=True,
    max_wait=3600,  # Max wait time
    checkpoint_s3_uri='s3://bucket/checkpoints'
)
```

---

## Hyperparameter Tuning

Automatic hyperparameter optimization.

```python
from sagemaker.tuner import HyperparameterTuner, ContinuousParameter

tuner = HyperparameterTuner(
    estimator=estimator,
    objective_metric_name='validation:accuracy',
    objective_type='Maximize',
    hyperparameter_ranges={
        'learning_rate': ContinuousParameter(0.001, 0.1),
        'epochs': IntegerParameter(10, 100)
    },
    max_jobs=20,
    max_parallel_jobs=3
)

tuner.fit({'training': 's3://bucket/train'})
```

### Tuning Strategies
- Random Search
- Bayesian Optimization
- Hyperband
- Grid Search

---

## Deployment

### Real-time Inference

```
Client ──▶ Endpoint ──▶ Model Container ──▶ Response
              │
         Auto Scaling
         (1-N instances)
```

```python
predictor = estimator.deploy(
    initial_instance_count=1,
    instance_type='ml.t2.medium',
    endpoint_name='my-endpoint'
)

result = predictor.predict(data)
```

### Deployment Options

| Type | Latency | Use Case |
|------|---------|----------|
| Real-time | Milliseconds | Interactive apps |
| Serverless | Variable | Sporadic traffic |
| Async | Seconds-minutes | Large payloads |
| Batch | Minutes-hours | Bulk processing |

### Multi-Model Endpoints
Host multiple models on single endpoint.

```
Endpoint
├── Model A (30% traffic)
├── Model B (50% traffic)
└── Model C (20% traffic)
```

### Serverless Inference

```python
from sagemaker.serverless import ServerlessInferenceConfig

serverless_config = ServerlessInferenceConfig(
    memory_size_in_mb=2048,
    max_concurrency=10
)

predictor = model.deploy(serverless_inference_config=serverless_config)
```

---

## Feature Store

Centralized repository for ML features.

```
┌─────────────────────────────────────────────────────────────────┐
│                       Feature Store                             │
│                                                                 │
│   Feature Group: customer_features                              │
│   ├── customer_id (identifier)                                  │
│   ├── total_purchases                                           │
│   ├── average_order_value                                       │
│   ├── days_since_last_purchase                                  │
│   └── customer_segment                                          │
│                                                                 │
│   ┌───────────────┐         ┌───────────────┐                  │
│   │ Online Store  │         │ Offline Store │                  │
│   │ (low latency) │         │ (S3/Athena)   │                  │
│   │               │         │               │                  │
│   │ Real-time     │         │ Training      │                  │
│   │ inference     │         │ data          │                  │
│   └───────────────┘         └───────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Benefits:
- Consistency between training and inference
- Feature reuse across teams
- Point-in-time queries
- Automatic feature engineering

---

## Pipelines

Automated ML workflows.

```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import TrainingStep, ProcessingStep

# Define steps
processing_step = ProcessingStep(...)
training_step = TrainingStep(...)
evaluation_step = ProcessingStep(...)

# Create pipeline
pipeline = Pipeline(
    name='my-pipeline',
    steps=[processing_step, training_step, evaluation_step],
    parameters=[...]
)

pipeline.upsert(role_arn=role)
pipeline.start()
```

---

## Model Monitor

Detect model and data drift.

```
Production Endpoint
        │
        ▼
┌───────────────┐
│ Model Monitor │
│               │
│ ├── Data Quality (input drift)
│ ├── Model Quality (performance)
│ ├── Bias Drift (fairness)
│ └── Feature Attribution (explainability)
│               │
└───────────────┘
        │
        ▼
CloudWatch Alerts
```

---

## Ground Truth

Data labeling service.

### Labeling Types
- Image classification
- Object detection
- Semantic segmentation
- Text classification
- Named entity recognition

### Workforce Options
- Amazon Mechanical Turk
- Private workforce
- Vendor managed

### Auto Labeling
Active learning to reduce manual labeling.

---

## Data Wrangler

Visual data preparation tool.

```
Import → Analyze → Transform → Export
  │         │          │         │
S3/Redshift  EDA    300+ built-in  Feature Store
Athena              transforms     S3
                                   Pipeline
```

---

## JumpStart

Pre-trained models and solutions.

### Model Hub
- Foundation models (Llama, Falcon, etc.)
- Pre-trained CV models
- NLP models
- Tabular models

### Solution Templates
- Fraud detection
- Demand forecasting
- Predictive maintenance
- Recommendation systems

---

## Canvas

No-code ML for business users.

```
Upload Data → Select Target → AutoML → Deploy
                                │
                          Automatic:
                          - Feature engineering
                          - Algorithm selection
                          - Hyperparameter tuning
```

---

## CLI Quick Reference

```bash
# List training jobs
aws sagemaker list-training-jobs

# Describe training job
aws sagemaker describe-training-job --training-job-name my-job

# List endpoints
aws sagemaker list-endpoints

# Create endpoint
aws sagemaker create-endpoint \
  --endpoint-name my-endpoint \
  --endpoint-config-name my-config

# Invoke endpoint
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name my-endpoint \
  --body '{"data": [...]}' \
  --content-type application/json \
  output.json

# Delete endpoint
aws sagemaker delete-endpoint --endpoint-name my-endpoint

# List models
aws sagemaker list-models

# List pipelines
aws sagemaker list-pipelines
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Studio | Free (pay for compute) |
| Notebook Instance | Per hour (ml.t3.medium: $0.05/hr) |
| Training | Per instance-hour |
| Inference | Per instance-hour |
| Serverless Inference | Per request + duration |
| Feature Store | Storage + requests |
| Ground Truth | Per labeled object |

### Savings
- Spot training: up to 90% off
- Savings Plans: up to 64% off
- Reserved capacity: committed use

---

## Security

### Network
- VPC deployment
- PrivateLink endpoints
- No internet access option

### Encryption
- At rest: KMS
- In transit: TLS

### Access Control
- IAM roles for jobs
- Execution roles
- Resource policies

---

## Best Practices

1. **Use Spot training** for cost savings
2. **Version everything** in Model Registry
3. **Automate with Pipelines**
4. **Monitor with Model Monitor**
5. **Use Feature Store** for consistency
6. **Start with JumpStart** models
7. **Enable experiments tracking**
8. **Use appropriate instance types**
9. **Implement checkpointing**
10. **Set up alerts** for drift

---

## Exam Tips

1. **Built-in algorithms** - know when to use each
2. **Training modes** - File, Pipe, FastFile
3. **Spot training** - checkpointing required
4. **Endpoint types** - real-time, serverless, async, batch
5. **Feature Store** - online (low latency) + offline (training)
6. **Model Monitor** - data quality, model quality, bias, explainability
7. **Ground Truth** - labeling with active learning
8. **Pipelines** - MLOps automation
9. **JumpStart** - pre-trained models and solutions
10. **Canvas** - no-code ML
11. **Hyperparameter tuning** - Bayesian optimization
12. **Multi-model endpoints** - cost optimization
