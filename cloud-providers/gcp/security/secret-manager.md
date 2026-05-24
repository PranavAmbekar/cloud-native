# Google Secret Manager

> Secure storage, management, and access to application secrets.

## Overview

Secret Manager is a secure and convenient storage system for API keys, passwords, certificates, and other sensitive data. It provides a central place to manage secrets with versioning, audit logging, and fine-grained IAM access control.

## Key Concepts

| Term | Definition |
|------|------------|
| Secret | Container for sensitive data |
| Version | Specific instance of secret data |
| Accessor | Principal that can read secrets |
| Replication | How secret data is distributed |
| Rotation | Updating secret values periodically |

## Architecture

```
+---------------------------------------------------------------+
|                       Secret Manager                          |
|                                                               |
|  +---------------------------------------------------------+  |
|  |                        Secret                           |  |
|  |   Name: my-api-key                                      |  |
|  |   Labels: env=production                                |  |
|  |                                                         |  |
|  |   +---------------------------------------------------+ |  |
|  |   |                    Versions                       | |  |
|  |   |   +---------+ +---------+ +---------+             | |  |
|  |   |   |Version 1| |Version 2| |Version 3|             | |  |
|  |   |   |(disabled)| |(disabled)| | (latest)|             | |  |
|  |   |   +---------+ +---------+ +---------+             | |  |
|  |   +---------------------------------------------------+ |  |
|  |                                                         |  |
|  |   Replication: Automatic (all regions)                  |  |
|  |   IAM: roles/secretmanager.secretAccessor               |  |
|  +---------------------------------------------------------+  |
|                                                               |
+---------------------------------------------------------------+
```

## Create and Manage Secrets

### Create Secret

```bash
# Create secret
gcloud secrets create my-secret \
  --replication-policy="automatic"

# Create and add version in one step
echo -n "my-secret-value" | gcloud secrets create my-secret \
  --replication-policy="automatic" \
  --data-file=-

# Create with user-managed replication
gcloud secrets create my-secret \
  --replication-policy="user-managed" \
  --locations=us-central1,us-east1
```

### Add Secret Version

```bash
# Add new version from stdin
echo -n "new-secret-value" | gcloud secrets versions add my-secret --data-file=-

# Add from file
gcloud secrets versions add my-secret --data-file=./secret.txt
```

### Access Secret

```bash
# Access latest version
gcloud secrets versions access latest --secret=my-secret

# Access specific version
gcloud secrets versions access 3 --secret=my-secret

# Output to file
gcloud secrets versions access latest --secret=my-secret --out-file=./output.txt
```

### Manage Versions

```bash
# List versions
gcloud secrets versions list my-secret

# Disable version
gcloud secrets versions disable 2 --secret=my-secret

# Enable version
gcloud secrets versions enable 2 --secret=my-secret

# Destroy version (permanent)
gcloud secrets versions destroy 1 --secret=my-secret
```

## Replication Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| **Automatic** | GCP manages replication | Most use cases |
| **User-managed** | You specify regions | Data residency, compliance |

### User-Managed Replication

```bash
gcloud secrets create my-secret \
  --replication-policy="user-managed" \
  --locations=us-central1,europe-west1
```

## IAM Roles

| Role | Description |
|------|-------------|
| `roles/secretmanager.admin` | Full access to secrets |
| `roles/secretmanager.secretAccessor` | Access secret data |
| `roles/secretmanager.secretVersionAdder` | Add new versions |
| `roles/secretmanager.secretVersionManager` | Manage versions |
| `roles/secretmanager.viewer` | View metadata only |

### Grant Access

```bash
# Grant accessor role
gcloud secrets add-iam-policy-binding my-secret \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Grant to compute default service account
gcloud secrets add-iam-policy-binding my-secret \
  --member="serviceAccount:123456789-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## SDK Usage

### Python

```python
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()

# Access secret
def access_secret(project_id: str, secret_id: str, version: str = "latest") -> str:
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# Create secret
def create_secret(project_id: str, secret_id: str):
    parent = f"projects/{project_id}"
    secret = client.create_secret(
        request={
            "parent": parent,
            "secret_id": secret_id,
            "secret": {"replication": {"automatic": {}}}
        }
    )
    return secret

# Add secret version
def add_secret_version(project_id: str, secret_id: str, payload: str):
    parent = f"projects/{project_id}/secrets/{secret_id}"
    response = client.add_secret_version(
        request={
            "parent": parent,
            "payload": {"data": payload.encode("UTF-8")}
        }
    )
    return response
```

### Go

```go
import (
    secretmanager "cloud.google.com/go/secretmanager/apiv1"
    "context"
    secretmanagerpb "google.golang.org/genproto/googleapis/cloud/secretmanager/v1"
)

func accessSecret(projectID, secretID, version string) (string, error) {
    ctx := context.Background()
    client, err := secretmanager.NewClient(ctx)
    if err != nil {
        return "", err
    }
    defer client.Close()

    req := &secretmanagerpb.AccessSecretVersionRequest{
        Name: fmt.Sprintf("projects/%s/secrets/%s/versions/%s", projectID, secretID, version),
    }

    result, err := client.AccessSecretVersion(ctx, req)
    if err != nil {
        return "", err
    }

    return string(result.Payload.Data), nil
}
```

## Integration with GCP Services

### Cloud Run

```yaml
# service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/my-project/my-app
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: latest
```

```bash
# Deploy with secret
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-app \
  --set-secrets=API_KEY=my-secret:latest
```

### Cloud Functions

```bash
gcloud functions deploy my-function \
  --runtime=python39 \
  --trigger-http \
  --set-secrets='API_KEY=my-secret:latest'
```

### GKE (External Secrets Operator)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-external-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: SecretStore
  target:
    name: my-k8s-secret
  data:
  - secretKey: api-key
    remoteRef:
      key: my-secret
      version: latest
```

## Secret Rotation

### Rotation Schedule

```bash
# Set rotation schedule
gcloud secrets update my-secret \
  --next-rotation-time="2024-06-01T00:00:00Z" \
  --rotation-period="30d" \
  --topics="projects/my-project/topics/secret-rotation"
```

### Rotation Notification

```python
# Cloud Function triggered by Pub/Sub
def rotate_secret(event, context):
    import base64
    import json

    pubsub_message = base64.b64decode(event['data']).decode('utf-8')
    message = json.loads(pubsub_message)

    secret_name = message['name']
    # Generate new secret value
    new_value = generate_new_secret()

    # Add new version
    add_secret_version(project_id, secret_name, new_value)
```

## Labels and Filtering

```bash
# Create secret with labels
gcloud secrets create my-secret \
  --labels=env=production,team=backend

# Update labels
gcloud secrets update my-secret \
  --update-labels=env=staging

# Filter by labels
gcloud secrets list --filter="labels.env=production"
```

## Audit Logging

Secret Manager automatically logs:
- Secret creation/deletion
- Version additions
- Access events
- IAM policy changes

```bash
# View audit logs
gcloud logging read "resource.type=secretmanager.googleapis.com/Secret"
```

## CLI Quick Reference

```bash
# Create secret
gcloud secrets create my-secret --replication-policy="automatic"

# Add version
echo -n "value" | gcloud secrets versions add my-secret --data-file=-

# Access secret
gcloud secrets versions access latest --secret=my-secret

# List secrets
gcloud secrets list

# List versions
gcloud secrets versions list my-secret

# Describe secret
gcloud secrets describe my-secret

# Delete secret
gcloud secrets delete my-secret

# Disable version
gcloud secrets versions disable 2 --secret=my-secret

# Destroy version
gcloud secrets versions destroy 1 --secret=my-secret

# Grant access
gcloud secrets add-iam-policy-binding my-secret \
  --member="user:alice@example.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Pricing

| Component | Cost |
|-----------|------|
| Active secret versions | $0.06 per version/month |
| Access operations | $0.03 per 10,000 |
| Rotation operations | Included |
| Destroy operations | Free |

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Versions**: Immutable, can disable but not edit
2. **Replication**: Automatic or user-managed
3. **IAM**: secretAccessor for reading values
4. **Latest**: Alias for most recent enabled version
5. **Audit logs**: Automatic, track all access
6. **Rotation**: Use Pub/Sub for notifications
7. **Destroy**: Permanent, cannot recover
8. **Labels**: For organizing and filtering
9. **Integration**: Native with Cloud Run, Functions, GKE
10. **CMEK**: Optional customer-managed encryption

## Gotchas

- Cannot edit secret versions (create new version)
- Destroyed versions are permanently gone
- Secret names cannot be reused after deletion
- Access operations are metered
- Automatic replication is to all GCP regions
- IAM changes may take time to propagate
- Maximum secret value size is 64 KB
- Version "latest" is newest enabled version
- Labels don't encrypt (don't put sensitive data)
- Regional secrets need service in same region

## Limits

| Resource | Limit |
|----------|-------|
| Secrets per project | 10,000 |
| Versions per secret | 10,000 |
| Secret value size | 64 KB |
| Secret name length | 255 characters |
| Labels per secret | 64 |
| Label key length | 63 characters |
| Label value length | 63 characters |
| Access operations | 3,000 per minute |
