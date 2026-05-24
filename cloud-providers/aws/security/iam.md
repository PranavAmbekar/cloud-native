# AWS IAM (Identity and Access Management)

> Manage access to AWS services and resources securely.

## Overview

IAM enables you to manage access to AWS services. You create users, groups, roles, and policies that define who can do what on which resources.

## Key Concepts

| Term | Definition |
|------|------------|
| Principal | Entity that can make requests (user, role, service) |
| User | Person or application with long-term credentials |
| Group | Collection of users (attach policies to groups) |
| Role | Identity with temporary credentials (assumed by users/services) |
| Policy | JSON document defining permissions |
| Permission Boundary | Maximum permissions an entity can have |

## IAM Hierarchy

```
AWS Account (Root User)
    |
    +-- IAM Users
    |   +-- Attached Policies (direct or via groups)
    |
    +-- IAM Groups
    |   +-- Attached Policies (inherited by members)
    |
    +-- IAM Roles
    |   +-- Trust Policy + Permission Policies
    |
    +-- IAM Policies
        +-- AWS Managed Policies
        +-- Customer Managed Policies
        +-- Inline Policies
```

## Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2Describe",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### Policy Elements

| Element | Required | Description |
|---------|----------|-------------|
| Version | Yes | Always use "2012-10-17" |
| Statement | Yes | Array of permission statements |
| Sid | No | Statement identifier |
| Effect | Yes | "Allow" or "Deny" |
| Action | Yes | API actions (e.g., "s3:GetObject") |
| Resource | Yes | ARN of resources |
| Condition | No | When policy applies |

### Common Condition Keys

```json
// IP-based access
"Condition": {
  "IpAddress": {"aws:SourceIp": "203.0.113.0/24"}
}

// MFA required
"Condition": {
  "Bool": {"aws:MultiFactorAuthPresent": "true"}
}

// Tag-based access
"Condition": {
  "StringEquals": {"aws:ResourceTag/Environment": "Production"}
}

// Time-based access
"Condition": {
  "DateGreaterThan": {"aws:CurrentTime": "2024-01-01T00:00:00Z"}
}

// Specific service
"Condition": {
  "StringEquals": {"aws:SourceService": "cloudtrail.amazonaws.com"}
}
```

## Policy Types

| Type | Scope | Use Case |
|------|-------|----------|
| **AWS Managed** | Global | Common use cases (ReadOnlyAccess, PowerUser) |
| **Customer Managed** | Account | Custom policies, reusable |
| **Inline** | Single entity | One-off, tightly coupled permissions |

## IAM Roles

### Use Cases
- EC2 instance accessing S3
- Lambda function accessing DynamoDB
- Cross-account access
- Federated users (SAML, OIDC)

### Trust Policy (Who can assume)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "ec2.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

### Cross-Account Role
```json
// Trust Policy in Account B
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT-A-ID:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id"
      }
    }
  }]
}
```

## Permission Evaluation Logic

```
+-------------------------------------------------+
|                  Is there an                    |
|                EXPLICIT DENY?                   |
|                      |                          |
|          Yes --------+--------- No              |
|           |          |          |               |
|           v          |          v               |
|         DENY         |    Is there an           |
|                      |    EXPLICIT ALLOW?       |
|                      |          |               |
|                      |   Yes ---+----- No       |
|                      |    |     |      |        |
|                      |    v     |      v        |
|                      |  ALLOW   |   IMPLICIT    |
|                      |          |     DENY      |
+-------------------------------------------------+

Order: Explicit Deny > Explicit Allow > Implicit Deny
```

### Permission Boundaries

```
Effective Permissions = Policy Permissions ∩ Permission Boundary

Example:
- Policy allows: s3:*, ec2:*, lambda:*
- Boundary allows: s3:*, ec2:*
- Effective: s3:*, ec2:* (lambda denied by boundary)
```

## Best Practices

1. **Never use root account** for daily tasks
2. **Enable MFA** on root and all users
3. **Use groups** to assign permissions
4. **Principle of least privilege** - minimum required permissions
5. **Use roles** for applications and services (not access keys)
6. **Rotate credentials** regularly
7. **Use IAM Access Analyzer** to identify unused access
8. **Use permission boundaries** for delegated administration

## IAM Identity Center (SSO)

- Centralized access management
- Single sign-on to AWS accounts and applications
- Integrates with external IdPs (Okta, Azure AD, etc.)
- Permission Sets define access levels

```
Identity Source (Azure AD, Okta)
         |
         v
IAM Identity Center
         |
    +----+----+
    |         |
Account A  Account B
 (Admin)   (ReadOnly)
```

## Security Token Service (STS)

Provides temporary credentials:

| API | Use Case |
|-----|----------|
| AssumeRole | Cross-account, EC2 roles |
| AssumeRoleWithSAML | SAML federation |
| AssumeRoleWithWebIdentity | Web identity (Cognito, OIDC) |
| GetSessionToken | MFA-protected API calls |
| GetFederationToken | Federated users |

```bash
# Assume role
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name MySession
```

## Resource-Based Policies

Some services support resource-based policies (attached to resource, not identity):

- S3 bucket policies
- SQS queue policies
- SNS topic policies
- Lambda function policies
- KMS key policies

```json
// S3 Bucket Policy - allows cross-account access
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::OTHER-ACCOUNT:root"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

## CLI Quick Reference

```bash
# Create user
aws iam create-user --user-name newuser

# Add user to group
aws iam add-user-to-group --user-name newuser --group-name Developers

# Attach policy to user
aws iam attach-user-policy \
  --user-name newuser \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create role
aws iam create-role --role-name MyRole \
  --assume-role-policy-document file://trust-policy.json

# List policies attached to user
aws iam list-attached-user-policies --user-name newuser

# Simulate policy (test permissions)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/testuser \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/file.txt
```

## Exam Tips

1. **Explicit Deny always wins** - overrides any Allow
2. **Roles > Access Keys** - for EC2, Lambda, etc.
3. **Cross-account**: Role in target account + assume permission in source
4. **External ID**: Prevents confused deputy problem in cross-account
5. **Policy Variables**: `${aws:username}`, `${aws:userid}` for dynamic policies
6. **Service-linked roles**: Created by service, can't modify
7. **PassRole permission**: Required to assign roles to services
8. **Maximum session duration**: Roles up to 12 hours (default 1 hour)
9. **Instance profiles**: Container for EC2 role (console creates automatically)

## Common Policies

```json
// Force MFA
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyAllExceptMFA",
    "Effect": "Deny",
    "NotAction": [
      "iam:CreateVirtualMFADevice",
      "iam:EnableMFADevice",
      "iam:GetUser",
      "iam:ListMFADevices"
    ],
    "Resource": "*",
    "Condition": {
      "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
    }
  }]
}

// S3 folder per user
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::bucket/${aws:username}/*"
    ]
  }]
}
```

## Gotchas

- Groups cannot be nested (no groups within groups)
- Users can belong to max 10 groups
- Policy size limits: managed (6KB), inline (2KB user, 10KB role)
- IAM is global (not regional)
- Changes can take a few seconds to propagate
- Root account cannot be restricted by IAM policies

## Limits

| Resource | Default Limit |
|----------|---------------|
| Users per account | 5,000 |
| Groups per account | 300 |
| Roles per account | 1,000 |
| Managed policies per account | 1,500 |
| Groups per user | 10 |
| Policies attached to entity | 10 managed |
| Policy versions | 5 (per managed policy) |
| Access keys per user | 2 |
