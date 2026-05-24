# Google Vertex AI

> Unified ML platform for building, deploying, and scaling ML models with pre-trained and custom models.

## Overview

Vertex AI is Google Cloud's unified machine learning platform that brings together Google Cloud services for building ML under one unified API, client library, and UI. It includes AutoML, custom training, model deployment, and generative AI capabilities.

## Key Concepts

| Term | Definition |
|------|------------|
| Dataset | Managed data for training |
| Training Pipeline | ML model training workflow |
| Model | Trained ML artifact |
| Endpoint | Deployed model for predictions |
| Prediction | Inference from deployed model |
| Feature Store | Centralized ML feature management |
| Model Registry | Version and manage models |

## Architecture

```
+---------------------------------------------------------------+
|                          Vertex AI                            |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                    Data Preparation                     |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  |  | Datasets | | Feature  | |   Data   | | Labeling |    |  |
|  |  |          | |  Store   | | Labeling | | Service  |    |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                              v                                |
|  +---------------------------------------------------------+  |
|  |                       Training                          |  |
|  |  +----------+ +----------+ +-----------+ +-----------+  |  |
|  |  |  AutoML  | |  Custom  | |Hyperparams| |Distributed|  |  |
|  |  | Training | | Training | |  Tuning   | | Training  |  |  |
|  |  +----------+ +----------+ +-----------+ +-----------+  |  |
|  +---------------------------------------------------------+  |
|                              |                                |
|                              v                                |
|  +---------------------------------------------------------+  |
|  |                      Deployment                         |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  |  | Endpoints| |  Batch   | |  Model   | |  Model   |    |  |
|  |  | (Online) | |Prediction| | Registry | |Monitoring|    |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  +---------------------------------------------------------+  |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                     Generative AI                       |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  |  |  Gemini  | |  PaLM 2  | |  Imagen  | |  Codey   |    |  |
|  |  |  (LLM)   | | (Legacy) | | (Images) | |  (Code)  |    |  |
|  |  +----------+ +----------+ +----------+ +----------+    |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

## Generative AI (Gemini)

### Available Models

| Model | Description |
|-------|-------------|
| gemini-1.5-pro | Most capable, 1M token context |
| gemini-1.5-flash | Fast, efficient |
| gemini-1.0-pro | Previous generation |
| text-bison | Text generation (legacy) |
| code-bison | Code generation |
| chat-bison | Conversational |
| imagetext | Image captioning |

### Text Generation

```python
import vertexai
from vertexai.generative_models import GenerativeModel

vertexai.init(project="my-project", location="us-central1")

model = GenerativeModel("gemini-1.5-pro")

response = model.generate_content("Write a poem about cloud computing")
print(response.text)
```

### Chat

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-1.5-pro")
chat = model.start_chat()

response = chat.send_message("Hello, what can you help me with?")
print(response.text)

response = chat.send_message("Tell me about Vertex AI")
print(response.text)
```

### Multimodal (Image + Text)

```python
from vertexai.generative_models import GenerativeModel, Part

model = GenerativeModel("gemini-1.5-pro")

image = Part.from_uri("gs://my-bucket/image.jpg", mime_type="image/jpeg")
prompt = "Describe this image in detail"

response = model.generate_content([image, prompt])
print(response.text)
```

### Embeddings

```python
from vertexai.language_models import TextEmbeddingModel

model = TextEmbeddingModel.from_pretrained("text-embedding-004")
embeddings = model.get_embeddings(["Hello, World!"])

for embedding in embeddings:
    print(f"Embedding dimension: {len(embedding.values)}")
```

## AutoML

### Create Dataset

```bash
# Create image classification dataset
gcloud ai datasets create \
  --region=us-central1 \
  --display-name="my-image-dataset" \
  --metadata-schema-uri="gs://google-cloud-aiplatform/schema/dataset/metadata/image_1.0.0.yaml"

# Import data
gcloud ai datasets import my-dataset \
  --region=us-central1 \
  --source="gs://my-bucket/data.csv"
```

### Train AutoML Model

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1")

# Image classification
job = aiplatform.AutoMLImageTrainingJob(
    display_name="my-automl-job",
    prediction_type="classification",
    multi_label=False
)

model = job.run(
    dataset=dataset,
    model_display_name="my-model",
    training_fraction_split=0.8,
    validation_fraction_split=0.1,
    test_fraction_split=0.1,
    budget_milli_node_hours=8000
)
```

### AutoML Types

| Type | Task |
|------|------|
| Image | Classification, object detection, segmentation |
| Video | Classification, object tracking |
| Text | Classification, extraction, sentiment |
| Tabular | Classification, regression, forecasting |

## Custom Training

### Training Script

```python
# trainer/train.py
import argparse
from google.cloud import aiplatform

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--epochs', type=int, default=10)
    parser.add_argument('--model-dir', type=str, default='/gcs/model')
    args = parser.parse_args()

    # Training logic
    model = train_model(args.epochs)

    # Save model
    model.save(args.model_dir)

if __name__ == '__main__':
    main()
```

### Submit Training Job

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1")

job = aiplatform.CustomTrainingJob(
    display_name="my-training-job",
    script_path="trainer/train.py",
    container_uri="us-docker.pkg.dev/vertex-ai/training/tf-cpu.2-12:latest",
    requirements=["pandas", "scikit-learn"],
    model_serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/tf2-cpu.2-12:latest"
)

model = job.run(
    replica_count=1,
    machine_type="n1-standard-4",
    accelerator_type="NVIDIA_TESLA_T4",
    accelerator_count=1,
    args=["--epochs=20"]
)
```

### Pre-built Containers

| Framework | Container |
|-----------|-----------|
| TensorFlow | us-docker.pkg.dev/vertex-ai/training/tf-cpu.2-12 |
| PyTorch | us-docker.pkg.dev/vertex-ai/training/pytorch-gpu.2-0 |
| scikit-learn | us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-0 |
| XGBoost | us-docker.pkg.dev/vertex-ai/training/xgboost-cpu.1-1 |

## Model Deployment

### Deploy to Endpoint

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1")

# Create endpoint
endpoint = aiplatform.Endpoint.create(display_name="my-endpoint")

# Deploy model
model = aiplatform.Model("projects/my-project/locations/us-central1/models/my-model")

endpoint.deploy(
    model=model,
    deployed_model_display_name="my-deployed-model",
    machine_type="n1-standard-4",
    min_replica_count=1,
    max_replica_count=5,
    traffic_percentage=100
)
```

### Online Prediction

```python
from google.cloud import aiplatform

endpoint = aiplatform.Endpoint("projects/my-project/locations/us-central1/endpoints/my-endpoint")

instances = [
    {"feature1": 1.0, "feature2": "value"}
]

predictions = endpoint.predict(instances=instances)
print(predictions)
```

### Batch Prediction

```python
batch_job = model.batch_predict(
    job_display_name="my-batch-job",
    gcs_source="gs://my-bucket/input/*.jsonl",
    gcs_destination_prefix="gs://my-bucket/output/",
    machine_type="n1-standard-4",
    starting_replica_count=1,
    max_replica_count=10
)
```

## Feature Store

```python
from google.cloud import aiplatform

# Create feature store
featurestore = aiplatform.Featurestore.create(
    featurestore_id="my-featurestore",
    online_store_fixed_node_count=1
)

# Create entity type
entity_type = featurestore.create_entity_type(
    entity_type_id="users",
    description="User features"
)

# Create feature
feature = entity_type.create_feature(
    feature_id="age",
    value_type="INT64",
    description="User age"
)

# Ingest features
entity_type.ingest_from_gcs(
    feature_ids=["age", "country"],
    feature_time="timestamp",
    gcs_source_uris="gs://my-bucket/features.csv"
)

# Online serving
entity_type.read(entity_ids=["user123"])
```

## Pipelines

```python
from kfp import dsl
from google.cloud import aiplatform

@dsl.component
def train_component(data_path: str) -> str:
    # Training logic
    return model_path

@dsl.component
def deploy_component(model_path: str) -> str:
    # Deployment logic
    return endpoint_path

@dsl.pipeline(name="my-pipeline")
def my_pipeline(data_path: str):
    train_task = train_component(data_path=data_path)
    deploy_task = deploy_component(model_path=train_task.output)

# Compile and run
from kfp import compiler
compiler.Compiler().compile(my_pipeline, 'pipeline.json')

job = aiplatform.PipelineJob(
    display_name="my-pipeline-job",
    template_path="pipeline.json",
    parameter_values={"data_path": "gs://my-bucket/data"}
)
job.run()
```

## CLI Quick Reference

```bash
# Initialize
gcloud ai init

# List models
gcloud ai models list --region=us-central1

# Create endpoint
gcloud ai endpoints create --region=us-central1 --display-name="my-endpoint"

# Deploy model
gcloud ai endpoints deploy-model ENDPOINT_ID \
  --region=us-central1 \
  --model=MODEL_ID \
  --display-name="deployed-model" \
  --machine-type=n1-standard-4 \
  --min-replica-count=1 \
  --max-replica-count=5

# Online prediction
gcloud ai endpoints predict ENDPOINT_ID \
  --region=us-central1 \
  --json-request=request.json

# Create training job
gcloud ai custom-jobs create \
  --region=us-central1 \
  --display-name="my-job" \
  --worker-pool-spec=machine-type=n1-standard-4,replica-count=1,container-image-uri=gcr.io/my-project/trainer

# List endpoints
gcloud ai endpoints list --region=us-central1
```

## Pricing

| Component | Cost |
|-----------|------|
| Training | Per node-hour (varies by machine) |
| Prediction (online) | Per node-hour + per prediction |
| Prediction (batch) | Per node-hour |
| Gemini Pro | Per 1M characters |
| AutoML | Per node-hour |
| Feature Store | Per GB stored + operations |

## Exam Tips (Professional ML Engineer, Cloud Architect)

1. **AutoML**: No-code ML for common tasks
2. **Custom training**: Full control with containers
3. **Endpoints**: Managed model serving
4. **Feature Store**: Centralized feature management
5. **Pipelines**: ML workflow orchestration
6. **Model Registry**: Version and manage models
7. **Gemini**: Latest generative AI models
8. **Batch vs Online**: Choose based on latency needs
9. **Pre-built containers**: Faster than custom
10. **Explainability**: Built-in model explanations

## Gotchas

- AutoML training can be expensive
- Custom containers need specific structure
- Endpoints have cold start latency
- Feature Store has online/offline stores
- Pipelines require KFP SDK knowledge
- Model deployment needs serving container
- Batch prediction requires specific input format
- Gemini has rate limits
- Regional resources (not global)
- Training timeouts vary by job type

## Limits

| Resource | Limit |
|----------|-------|
| Models per project | 10,000 |
| Endpoints per project | 10,000 |
| Deployed models per endpoint | 4 |
| Concurrent predictions | 10,000 per region |
| Training job duration | 7 days |
| Pipeline runs per day | 1,000 |
| Feature stores per project | 100 |
| Features per entity type | 400 |
| Gemini context window | 1M tokens (Pro) |
