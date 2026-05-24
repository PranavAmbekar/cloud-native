# Azure Machine Learning

> End-to-end platform for building, training, deploying, and managing machine learning models at scale.

## Overview

Azure Machine Learning is a cloud-based environment for training, deploying, automating, managing, and tracking ML models. It supports the complete machine learning lifecycle from data preparation to model deployment and monitoring.

## Key Concepts

| Term | Definition |
|------|------------|
| Workspace | Top-level resource, collaboration boundary |
| Compute | Training and inference infrastructure |
| Datastore | Connection to data storage |
| Dataset | Versioned reference to data |
| Experiment | Collection of training runs |
| Model | Trained ML artifact |
| Endpoint | Deployed model serving predictions |
| Pipeline | Automated ML workflow |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure ML Workspace                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                       Studio UI                            │  │
│  │  • Designer (drag-and-drop)  • Notebooks  • AutoML        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Compute   │  │ Datastores  │  │   Models    │             │
│  │             │  │             │  │             │             │
│  │ • Instances │  │ • Blob      │  │ • Registry  │             │
│  │ • Clusters  │  │ • ADLS      │  │ • Versions  │             │
│  │ • Kubernetes│  │ • SQL       │  │ • Artifacts │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                              │                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Experiments │  │  Pipelines  │  │  Endpoints  │             │
│  │             │  │             │  │             │             │
│  │ • Runs      │  │ • Jobs      │  │ • Online    │             │
│  │ • Metrics   │  │ • Components│  │ • Batch     │             │
│  │ • Artifacts │  │ • Schedules │  │             │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   Storage Account      Key Vault      Container Registry
    (Artifacts)         (Secrets)         (Images)
```

## Compute Types

| Type | Use Case | Scaling |
|------|----------|---------|
| **Compute Instance** | Development, notebooks | Single VM |
| **Compute Cluster** | Training jobs | Auto-scale 0-N nodes |
| **Serverless Compute** | On-demand training | Managed |
| **Kubernetes** | Training and inference | AKS or Arc |
| **Managed Endpoints** | Real-time inference | Auto-scale |
| **Batch Endpoints** | Batch inference | As needed |

### Create Compute

```bash
# Create compute instance
az ml compute create \
  --name dev-instance \
  --type ComputeInstance \
  --size Standard_DS3_v2 \
  --workspace-name myworkspace \
  --resource-group myRG

# Create compute cluster
az ml compute create \
  --name train-cluster \
  --type AmlCompute \
  --size Standard_NC6 \
  --min-instances 0 \
  --max-instances 4 \
  --workspace-name myworkspace \
  --resource-group myRG
```

## SDK v2 Basics

```python
from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential

# Connect to workspace
ml_client = MLClient(
    DefaultAzureCredential(),
    subscription_id="subscription-id",
    resource_group_name="myRG",
    workspace_name="myworkspace"
)
```

## Training Jobs

### Command Job

```python
from azure.ai.ml import command, Input

job = command(
    code="./src",
    command="python train.py --data ${{inputs.training_data}} --epochs 10",
    inputs={
        "training_data": Input(
            type="uri_folder",
            path="azureml://datastores/workspaceblobstore/paths/data"
        )
    },
    environment="AzureML-sklearn-1.0@latest",
    compute="train-cluster",
    experiment_name="my-experiment"
)

# Submit job
returned_job = ml_client.jobs.create_or_update(job)
```

### YAML Configuration

```yaml
# job.yml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
type: command

code: ./src
command: python train.py --data ${{inputs.training_data}}

inputs:
  training_data:
    type: uri_folder
    path: azureml://datastores/workspaceblobstore/paths/data

environment: azureml:AzureML-sklearn-1.0@latest
compute: azureml:train-cluster
experiment_name: my-experiment
```

```bash
az ml job create --file job.yml --workspace-name myworkspace --resource-group myRG
```

## Environments

### Curated Environments

```python
# List curated environments
envs = ml_client.environments.list()
for env in envs:
    print(env.name)

# Use curated environment
environment = "AzureML-sklearn-1.0@latest"
```

### Custom Environment

```yaml
# environment.yml
$schema: https://azuremlschemas.azureedge.net/latest/environment.schema.json
name: my-custom-env
version: 1
image: mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04
conda_file: conda.yml
```

```yaml
# conda.yml
dependencies:
  - python=3.9
  - pip:
    - scikit-learn==1.0.2
    - pandas==1.4.2
    - mlflow==2.0.0
```

## Data Assets

### Register Data

```python
from azure.ai.ml.entities import Data
from azure.ai.ml.constants import AssetTypes

my_data = Data(
    name="my-dataset",
    version="1",
    description="Training data",
    path="azureml://datastores/workspaceblobstore/paths/data/train.csv",
    type=AssetTypes.URI_FILE
)

ml_client.data.create_or_update(my_data)
```

### Data Types

| Type | Description |
|------|-------------|
| uri_file | Single file |
| uri_folder | Directory |
| mltable | Tabular data with schema |

## Pipelines

### Define Pipeline

```python
from azure.ai.ml import dsl, Input, Output
from azure.ai.ml.entities import Pipeline

@dsl.pipeline(
    compute="cpu-cluster",
    description="Training pipeline"
)
def training_pipeline(input_data):
    # Step 1: Prepare data
    prep_step = prep_component(input_data=input_data)

    # Step 2: Train model
    train_step = train_component(
        training_data=prep_step.outputs.output_data
    )

    return {"model": train_step.outputs.model_output}

# Create pipeline
pipeline = training_pipeline(
    input_data=Input(type="uri_folder", path="azureml://datastores/...")
)

# Submit
pipeline_job = ml_client.jobs.create_or_update(pipeline)
```

### Components

```yaml
# component.yml
$schema: https://azuremlschemas.azureedge.net/latest/commandComponent.schema.json
name: train_model
version: 1
type: command

inputs:
  training_data:
    type: uri_folder
  learning_rate:
    type: number
    default: 0.01

outputs:
  model_output:
    type: uri_folder

code: ./src
environment: azureml:AzureML-sklearn-1.0@latest
command: >-
  python train.py
  --data ${{inputs.training_data}}
  --lr ${{inputs.learning_rate}}
  --model ${{outputs.model_output}}
```

## AutoML

### Code-First AutoML

```python
from azure.ai.ml import automl, Input

classification_job = automl.classification(
    compute="cpu-cluster",
    experiment_name="automl-classification",
    training_data=Input(type="mltable", path="./train_data"),
    target_column_name="label",
    primary_metric="accuracy",
    n_cross_validations=5,
    enable_model_explainability=True
)

# Configuration
classification_job.set_limits(
    max_trials=20,
    max_concurrent_trials=4,
    timeout_minutes=60
)

# Submit
returned_job = ml_client.jobs.create_or_update(classification_job)
```

### AutoML Task Types

| Task | Primary Metrics |
|------|-----------------|
| Classification | accuracy, AUC_weighted, precision |
| Regression | r2_score, RMSE, MAE |
| Time Series | MAPE, RMSE, R2 |
| Image Classification | accuracy |
| Object Detection | mAP |
| NLP Text Classification | accuracy |

## Model Registration

```python
from azure.ai.ml.entities import Model
from azure.ai.ml.constants import AssetTypes

# Register model from job output
model = Model(
    path=f"azureml://jobs/{job.name}/outputs/model",
    name="my-model",
    type=AssetTypes.MLFLOW_MODEL,
    description="Trained classification model"
)

ml_client.models.create_or_update(model)
```

## Endpoints

### Online Endpoint (Real-time)

```yaml
# endpoint.yml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineEndpoint.schema.json
name: my-endpoint
auth_mode: key
```

```yaml
# deployment.yml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json
name: blue
endpoint_name: my-endpoint
model: azureml:my-model@latest
instance_type: Standard_DS3_v2
instance_count: 1
```

```bash
# Create endpoint
az ml online-endpoint create --file endpoint.yml

# Create deployment
az ml online-deployment create --file deployment.yml --all-traffic
```

### Invoke Endpoint

```python
import json

# Get scoring URI
endpoint = ml_client.online_endpoints.get("my-endpoint")
scoring_uri = endpoint.scoring_uri

# Invoke
import urllib.request

data = {"data": [[1, 2, 3, 4, 5]]}
body = json.dumps(data).encode('utf-8')

req = urllib.request.Request(scoring_uri, body, headers={'Content-Type': 'application/json'})
req.add_header('Authorization', f'Bearer {api_key}')

response = urllib.request.urlopen(req)
result = response.read()
```

### Batch Endpoint

```yaml
# batch-endpoint.yml
$schema: https://azuremlschemas.azureedge.net/latest/batchEndpoint.schema.json
name: my-batch-endpoint
description: Batch scoring endpoint
```

```bash
# Create batch endpoint
az ml batch-endpoint create --file batch-endpoint.yml

# Invoke with data
az ml batch-endpoint invoke \
  --name my-batch-endpoint \
  --input azureml://datastores/workspaceblobstore/paths/batch-data
```

## MLflow Integration

```python
import mlflow
from azureml.core import Workspace

# Set tracking URI
ws = Workspace.from_config()
mlflow.set_tracking_uri(ws.get_mlflow_tracking_uri())

# Log experiment
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_metric("accuracy", 0.95)
    mlflow.sklearn.log_model(model, "model")
```

## Responsible AI

### Model Explainability

```python
from azure.ai.ml import automl

job = automl.classification(
    ...
    enable_model_explainability=True  # Generate explanations
)
```

### Fairness Assessment

```python
from raiwidgets import ResponsibleAIDashboard
from responsibleai import RAIInsights

# Create RAI insights
rai_insights = RAIInsights(model, train_data, test_data, "label", task_type="classification")
rai_insights.explainer.add()
rai_insights.error_analysis.add()
rai_insights.compute()

# View dashboard
ResponsibleAIDashboard(rai_insights)
```

## CLI Quick Reference

```bash
# Create workspace
az ml workspace create --name myworkspace --resource-group myRG

# List compute
az ml compute list --workspace-name myworkspace --resource-group myRG

# Submit job
az ml job create --file job.yml --workspace-name myworkspace --resource-group myRG

# Stream logs
az ml job stream --name <job-name> --workspace-name myworkspace --resource-group myRG

# List models
az ml model list --workspace-name myworkspace --resource-group myRG

# Create endpoint
az ml online-endpoint create --file endpoint.yml --workspace-name myworkspace --resource-group myRG

# Get endpoint keys
az ml online-endpoint get-credentials --name my-endpoint --workspace-name myworkspace --resource-group myRG
```

## Exam Tips (AZ-102, AZ-204, AZ-305)

1. **Workspace**: Central resource for all ML assets
2. **Compute cluster**: Auto-scales, use for training
3. **Compute instance**: Development, single VM
4. **Managed endpoints**: Easier than AKS for inference
5. **AutoML**: Automated model selection and tuning
6. **Pipelines**: Reusable, automated workflows
7. **MLflow**: Experiment tracking and model management
8. **Environments**: Reproducible Python environments
9. **Data assets**: Versioned data references
10. **Components**: Reusable pipeline building blocks

## Gotchas

- Workspace requires linked storage account, key vault, and ACR
- Compute clusters scale to 0 by default (cold start)
- Model versions are immutable
- Endpoints have different auth modes (key, AAD)
- MLflow models have specific directory structure
- AutoML has time and trial limits
- Pipeline components must have explicit inputs/outputs
- Batch endpoints process entire datasets
- Managed endpoints bill while running
- Custom environments take time to build

## Limits

| Resource | Limit |
|----------|-------|
| Workspaces per subscription | 800 |
| Compute instances per workspace | 1000 |
| Nodes per compute cluster | 100 |
| Experiments per workspace | Unlimited |
| Runs per experiment | Unlimited |
| Models per workspace | Unlimited |
| Endpoints per workspace | 500 |
| Pipeline components | Unlimited |
| AutoML concurrent trials | 100 |
