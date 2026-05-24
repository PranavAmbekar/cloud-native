# Google Cloud IAM (Identity and Access Management)

> Fine-grained access control for managing who can do what on which resources.

## Overview

Cloud IAM lets you grant granular access to specific Google Cloud resources and helps prevent access to other resources. IAM lets you adopt the security principle of least privilege.

## Key Concepts

| Term | Definition |
|------|------------|
| Principal | Who is requesting access (user, group, service account) |
| Role | Collection of permissions |
| Permission | Allows specific action on resource |
| Policy | Binds principals to roles |
| Resource | GCP resource being accessed |
| Condition | Contextual access rules |

## IAM Model

```
WHO (Principal)     +     WHAT (Role)     +     WHERE (Resource)
-----------------         --------------         -----------------
* User                    * Viewer               * Project
* Group                   * Editor               * Folder
* Service Account         * Owner                * Organization
* Domain                  * Custom roles         * Resource
* allUsers                * Predefined roles
* allAuthenticatedUsers
```

## Principal Types

| Type | Format | Description |
|------|--------|-------------|
| Google Account | user:alice@example.com | Individual user |
| Service Account | serviceAccount:sa@project.iam.gserviceaccount.com | Application identity |
| Google Group | group:admins@example.com | Collection of users |
| Google Workspace Domain | domain:example.com | All users in domain |
| Cloud Identity Domain | domain:example.com | All users in domain |
| allUsers | allUsers | Anyone (public) |
| allAuthenticatedUsers | allAuthenticatedUsers | Any authenticated user |

## Role Types

### Basic Roles (Primitive)

| Role | Description |
|------|-------------|
| roles/viewer | Read-only access |
| roles/editor | Read + write access |
| roles/owner | Full access + IAM management |

**Avoid using basic roles in production** - too broad.

### Predefined Roles

Fine-grained roles for specific services.

```
roles/compute.instanceAdmin.v1
|           |            |
|           |            +-- Version
|           +--------------- Resource/Action
+--------------------------- Service

Examples:
- roles/storage.objectViewer
- roles/compute.networkAdmin
- roles/cloudsql.admin
- roles/container.developer
```

### Custom Roles

```bash
# Create custom role
gcloud iam roles create myCustomRole \
  --project=my-project \
  --title="My Custom Role" \
  --description="Custom role for specific access" \
  --permissions=compute.instances.get,compute.instances.list

# Update custom role
gcloud iam roles update myCustomRole \
  --project=my-project \
  --add-permissions=compute.instances.start
```

## Service Accounts

### Types

| Type | Description |
|------|-------------|
| User-managed | Created by you |
| Default | Auto-created for services |
| Google-managed | Used by GCP services internally |

### Create Service Account

```bash
# Create service account
gcloud iam service-accounts create my-sa \
  --display-name="My Service Account" \
  --description="Used for app authentication"

# Grant roles
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Create key (avoid if possible)
gcloud iam service-accounts keys create key.json \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com
```

### Service Account Impersonation

```bash
# Grant impersonation permission
gcloud iam service-accounts add-iam-policy-binding \
  target-sa@my-project.iam.gserviceaccount.com \
  --member="user:alice@example.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# Use impersonation
gcloud storage ls gs://my-bucket \
  --impersonate-service-account=target-sa@my-project.iam.gserviceaccount.com
```

## Policy Bindings

### Grant Access

```bash
# Grant project-level role
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/compute.viewer"

# Grant resource-level role
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Grant folder-level role
gcloud resource-manager folders add-iam-policy-binding 123456789 \
  --member="group:admins@example.com" \
  --role="roles/resourcemanager.folderViewer"
```

### View Policy

```bash
# Get project IAM policy
gcloud projects get-iam-policy my-project

# Get bucket IAM policy
gcloud storage buckets get-iam-policy gs://my-bucket
```

## Resource Hierarchy

```
Organization (org-level policies)
    |
    +-- Folder (department policies)
    |   |
    |   +-- Project A (project policies)
    |   |   +-- Resource 1 (resource policies)
    |   |   +-- Resource 2
    |   |
    |   +-- Project B
    |       +-- Resource 3
    |
    +-- Project C

Policy Inheritance: Org -> Folder -> Project -> Resource
(More permissive wins - additive)
```

## IAM Conditions

Contextual access control.

```bash
# Time-based condition
gcloud projects add-iam-policy-binding my-project \
  --member="user:contractor@example.com" \
  --role="roles/compute.viewer" \
  --condition='
    title=WorkHoursOnly,
    description=Access only during work hours,
    expression=request.time.getHours("America/Los_Angeles") >= 9 &&
               request.time.getHours("America/Los_Angeles") <= 17
  '

# Resource-based condition
gcloud projects add-iam-policy-binding my-project \
  --member="user:dev@example.com" \
  --role="roles/storage.objectViewer" \
  --condition='
    title=DevBucketsOnly,
    description=Access only dev buckets,
    expression=resource.name.startsWith("projects/_/buckets/dev-")
  '
```

### Condition Attributes

| Attribute | Description |
|-----------|-------------|
| request.time | Request timestamp |
| resource.name | Resource name |
| resource.type | Resource type |
| resource.service | Service name |
| request.path | Request URL path |

## Workload Identity Federation

Access GCP without service account keys.

```
External Identity Provider           GCP
+-------------------------+         +-----------------+
|  AWS / Azure / GitHub   | ------> | Workload        |
|  OIDC Provider          | token   | Identity Pool   |
+-------------------------+         +--------+--------+
                                             |
                                    +--------v--------+
                                    | Service Account |
                                    | (impersonated)  |
                                    +-----------------+
```

```bash
# Create workload identity pool
gcloud iam workload-identity-pools create my-pool \
  --location="global" \
  --display-name="My Pool"

# Create provider
gcloud iam workload-identity-pools providers create-oidc my-provider \
  --location="global" \
  --workload-identity-pool=my-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

# Allow impersonation
gcloud iam service-accounts add-iam-policy-binding my-sa@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/my-pool/attribute.repository/my-org/my-repo"
```

## Policy Troubleshooter

```bash
# Check if principal has permission
gcloud policy-troubleshoot iam my-project \
  --permission=compute.instances.start \
  --principal=user:alice@example.com
```

## Best Practices

```
1. Least Privilege
   +-- Grant minimum necessary permissions

2. Use Groups
   +-- Manage access via groups, not individuals

3. Service Accounts
   +-- One per application
   +-- Avoid sharing
   +-- Use Workload Identity

4. Avoid Basic Roles
   +-- Use predefined or custom roles

5. Conditions
   +-- Add time/resource restrictions

6. Regular Audits
   +-- Review IAM policies periodically
```

## CLI Quick Reference

```bash
# List roles
gcloud iam roles list

# Describe role
gcloud iam roles describe roles/compute.admin

# List permissions for role
gcloud iam roles describe roles/compute.admin --format="value(includedPermissions)"

# List service accounts
gcloud iam service-accounts list

# Create service account
gcloud iam service-accounts create my-sa

# Grant role
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# Remove role
gcloud projects remove-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# Test permissions
gcloud projects test-iam-permissions my-project \
  --permissions=compute.instances.create,compute.instances.delete
```

## Exam Tips (Associate Cloud Engineer, Professional Cloud Architect)

1. **Inheritance**: Policies inherit down the hierarchy
2. **Additive**: Can't deny at lower level
3. **Service accounts**: Preferred for applications
4. **Basic roles**: Avoid in production
5. **Custom roles**: For specific permission sets
6. **Conditions**: Time/resource-based access
7. **Groups**: Easier management than individual users
8. **Workload Identity**: Keyless authentication
9. **Policy Troubleshooter**: Debug access issues
10. **Audit logs**: Track IAM changes

## Gotchas

- IAM policies are additive (can't explicitly deny)
- Basic roles are too permissive for production
- Service account keys are security risks
- Role changes can take time to propagate
- Custom roles require maintenance
- Conditions have performance impact
- allUsers includes unauthenticated users
- Organization policies can restrict IAM
- Some permissions can't be conditioned
- Service account impersonation needs explicit grants

## Limits

| Resource | Limit |
|----------|-------|
| Members per policy | 1,500 |
| Bindings per policy | 1,500 |
| Custom roles per project | 300 |
| Custom roles per org | 300 |
| Permissions per custom role | 3,000 |
| Service accounts per project | 100 |
| Keys per service account | 10 |
| Conditions per binding | 1 |
| Policy size | 64 KB |
| Workload identity pools | 100 per project |
