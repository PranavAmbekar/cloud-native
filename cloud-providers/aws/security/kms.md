# AWS KMS (Key Management Service)

> Managed service to create and control encryption keys.

---

## Key Concepts

| Term | Definition |
|------|------------|
| CMK | Customer Master Key (now called KMS Key) |
| Data Key | Key used to encrypt actual data |
| Envelope Encryption | Encrypt data key with CMK |
| Key Policy | Resource policy for key access |
| Grant | Temporary permission delegation |
| Alias | Friendly name for key |

---

## Key Types

| Type | Management | Use Case |
|------|------------|----------|
| AWS Owned | AWS manages, shared | Default S3, default EBS |
| AWS Managed | AWS manages, per-service | aws/s3, aws/ebs, aws/rds |
| Customer Managed | You manage | Full control, audit |
| Imported | You provide material | BYOK (Bring Your Own Key) |

### Key Hierarchy
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   AWS Owned    │   AWS Managed    │   Customer Managed         │
│                │                  │                             │
│   No visibility│   View in KMS    │   Full control             │
│   No control   │   Limited control│   Rotation, policies       │
│   Free         │   Free*          │   $1/month + usage         │
│                │                  │                             │
│   * charged when used by some services                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Envelope Encryption

```
┌─────────────────────────────────────────────────────────────────┐
│                    Envelope Encryption                          │
│                                                                 │
│   1. GenerateDataKey                                           │
│      KMS Key ──▶ Plaintext Data Key + Encrypted Data Key       │
│                                                                 │
│   2. Encrypt Data                                              │
│      Plaintext Data Key + Data ──▶ Encrypted Data              │
│                                                                 │
│   3. Store                                                     │
│      Encrypted Data Key + Encrypted Data (delete plaintext key)│
│                                                                 │
│   4. Decrypt (later)                                           │
│      Encrypted Data Key ──▶ KMS ──▶ Plaintext Data Key        │
│      Plaintext Data Key + Encrypted Data ──▶ Data             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Why envelope encryption?
- Avoids sending large data to KMS
- KMS has 4KB limit
- Data key can encrypt unlimited data

---

## Key Policies

Every KMS key must have a key policy.

### Default Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "Enable IAM policies",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT:root"},
    "Action": "kms:*",
    "Resource": "*"
  }]
}
```

### Custom Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow admin",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ACCOUNT:role/Admin"},
      "Action": [
        "kms:Create*",
        "kms:Describe*",
        "kms:Enable*",
        "kms:List*",
        "kms:Put*",
        "kms:Update*",
        "kms:Revoke*",
        "kms:Disable*",
        "kms:Get*",
        "kms:Delete*",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow use",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ACCOUNT:role/App"},
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Key Access

Access requires BOTH:
1. Key policy allows it
2. IAM policy allows it (if principal is in same account)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Same Account:                                                 │
│   Key Policy (root access) + IAM Policy = Access               │
│                                                                 │
│   Cross Account:                                                │
│   Key Policy (explicit principal) + IAM Policy = Access        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Rotation

### Automatic Rotation
- AWS managed keys: 1 year (automatic)
- Customer managed keys: Optional, 1 year
- Key ID stays same, backing key changes

### Manual Rotation
- Create new key
- Update applications to use new key
- Keep old key for decryption

```bash
# Enable automatic rotation
aws kms enable-key-rotation --key-id xxx
```

---

## Grants

Temporary, programmatic permissions.

```python
import boto3

kms = boto3.client('kms')

# Create grant
response = kms.create_grant(
    KeyId='arn:aws:kms:us-east-1:xxx:key/xxx',
    GranteePrincipal='arn:aws:iam::xxx:role/MyRole',
    Operations=['Encrypt', 'Decrypt'],
    Constraints={
        'EncryptionContextSubset': {
            'Department': 'Finance'
        }
    }
)
```

Use cases:
- Lambda function needs temporary access
- Cross-account temporary access
- Delegate permissions without policy change

---

## Multi-Region Keys

Replicate keys across regions.

```
┌──────────────────┐         ┌──────────────────┐
│   us-east-1      │         │   eu-west-1      │
│                  │         │                  │
│  Primary Key     │ ───────▶│  Replica Key    │
│  (mrk-xxx)       │  sync   │  (mrk-xxx)       │
│                  │         │                  │
└──────────────────┘         └──────────────────┘

Same key ID, interoperable
Encrypt in us-east-1, decrypt in eu-west-1
```

Use cases:
- Global applications
- Disaster recovery
- Global DynamoDB tables

---

## Encryption Context

Additional authenticated data (not encrypted, but authenticated).

```python
# Encrypt with context
response = kms.encrypt(
    KeyId='xxx',
    Plaintext=b'secret data',
    EncryptionContext={
        'department': 'finance',
        'purpose': 'payroll'
    }
)

# Must provide same context to decrypt
response = kms.decrypt(
    CiphertextBlob=encrypted_data,
    EncryptionContext={
        'department': 'finance',
        'purpose': 'payroll'
    }
)
```

Benefits:
- Additional authorization layer
- Appears in CloudTrail logs
- Protects against ciphertext swapping

---

## AWS Services Integration

| Service | Encryption |
|---------|------------|
| S3 | SSE-KMS |
| EBS | KMS encrypted volumes |
| RDS | KMS encrypted storage |
| DynamoDB | KMS encryption at rest |
| Lambda | Environment variable encryption |
| Secrets Manager | Secret encryption |
| SSM Parameter Store | SecureString parameters |
| SQS | Message encryption |
| SNS | Message encryption |

---

## KMS API Operations

| Operation | Purpose |
|-----------|---------|
| CreateKey | Create new KMS key |
| Encrypt | Encrypt data (up to 4KB) |
| Decrypt | Decrypt data |
| GenerateDataKey | Get plaintext + encrypted data key |
| GenerateDataKeyWithoutPlaintext | Get encrypted data key only |
| ReEncrypt | Re-encrypt under different key |
| ScheduleKeyDeletion | Delete key (7-30 day wait) |

---

## CLI Quick Reference

```bash
# Create key
aws kms create-key --description "My key"

# Create alias
aws kms create-alias \
  --alias-name alias/my-key \
  --target-key-id xxx

# Encrypt data
aws kms encrypt \
  --key-id alias/my-key \
  --plaintext fileb://data.txt \
  --output text \
  --query CiphertextBlob | base64 --decode > encrypted.bin

# Decrypt data
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --output text \
  --query Plaintext | base64 --decode > decrypted.txt

# Generate data key
aws kms generate-data-key \
  --key-id alias/my-key \
  --key-spec AES_256

# Enable rotation
aws kms enable-key-rotation --key-id xxx

# List keys
aws kms list-keys

# Describe key
aws kms describe-key --key-id xxx

# Schedule deletion
aws kms schedule-key-deletion \
  --key-id xxx \
  --pending-window-in-days 7
```

---

## Pricing

| Component | Cost |
|-----------|------|
| Customer managed keys | $1/month |
| AWS managed keys | Free (usage charges may apply) |
| API requests | $0.03/10,000 requests |
| Asymmetric operations | $0.10-$0.15/10,000 |

---

## Key Types (Cryptographic)

| Type | Use Case |
|------|----------|
| Symmetric (AES-256-GCM) | Encrypt/decrypt, envelope encryption |
| Asymmetric (RSA) | Sign/verify, encrypt/decrypt |
| Asymmetric (ECC) | Sign/verify only |
| HMAC | Message authentication codes |

---

## Best Practices

1. **Use aliases** - easier to manage and rotate
2. **Enable key rotation** - automatic yearly rotation
3. **Use encryption context** - additional auth data
4. **Least privilege policies** - separate admin and usage
5. **Use grants** for temporary access
6. **Multi-region keys** for global apps
7. **Audit with CloudTrail** - all KMS calls logged
8. **Avoid direct encryption** - use envelope encryption
9. **Separate keys per environment** - dev, staging, prod
10. **Delete keys carefully** - 7-30 day waiting period

---

## Exam Tips

1. **Envelope encryption** - data key encrypts data, KMS key encrypts data key
2. **4KB limit** - direct KMS encryption limited to 4KB
3. **GenerateDataKey** - returns plaintext AND encrypted data key
4. **Key policy + IAM policy** - both needed for same-account access
5. **Key policy only** - sufficient for cross-account (explicit principal)
6. **Rotation** - key ID stays same, backing key changes
7. **Multi-region keys** - same key ID across regions
8. **Encryption context** - authenticated but not encrypted
9. **Grants** - temporary, programmatic permissions
10. **Deletion waiting period** - 7-30 days, reversible
11. **AWS managed keys** - prefixed aws/service (aws/s3)
12. **CloudTrail** - all KMS operations logged
