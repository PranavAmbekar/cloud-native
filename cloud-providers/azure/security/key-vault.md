# Azure Key Vault

> Securely store and manage secrets, keys, and certificates for cloud applications and services.

## Overview

Azure Key Vault is a cloud service for securely storing and accessing secrets. A secret is anything you want to tightly control access to, such as API keys, passwords, certificates, or cryptographic keys.

## Key Concepts

| Term | Definition |
|------|------------|
| Vault | Container for secrets, keys, and certificates |
| Secret | Any sensitive data (passwords, connection strings) |
| Key | Cryptographic key for encryption/signing |
| Certificate | X.509 certificate with optional private key |
| Soft Delete | Recoverable deletion (retention period) |
| Purge Protection | Prevents permanent deletion during retention |

## Object Types

### Secrets

Store any sensitive string value.

```bash
# Store secret
az keyvault secret set \
  --vault-name myVault \
  --name "DatabasePassword" \
  --value "MyS3cr3tP@ssw0rd!"

# Retrieve secret
az keyvault secret show \
  --vault-name myVault \
  --name "DatabasePassword" \
  --query value

# List secrets
az keyvault secret list --vault-name myVault
```

### Keys

Cryptographic keys for encryption, signing, wrapping.

| Key Type | Sizes | Use Case |
|----------|-------|----------|
| RSA | 2048, 3072, 4096 | Encryption, signing |
| EC | P-256, P-384, P-521 | Signing, ECDH |
| Symmetric (oct) | 128, 192, 256 | AES operations (managed HSM) |

```bash
# Create RSA key
az keyvault key create \
  --vault-name myVault \
  --name "MyEncryptionKey" \
  --kty RSA \
  --size 2048

# Create EC key
az keyvault key create \
  --vault-name myVault \
  --name "MySigningKey" \
  --kty EC \
  --curve P-256
```

### Certificates

Manage SSL/TLS certificates with automatic renewal.

```bash
# Create self-signed certificate
az keyvault certificate create \
  --vault-name myVault \
  --name "MyCert" \
  --policy "$(az keyvault certificate get-default-policy)"

# Import existing certificate
az keyvault certificate import \
  --vault-name myVault \
  --name "MyCert" \
  --file mycert.pfx \
  --password "pfxpassword"
```

## SKU Tiers

| Feature | Standard | Premium |
|---------|----------|---------|
| Secrets | Yes | Yes |
| Keys (software-protected) | Yes | Yes |
| Keys (HSM-protected) | No | Yes |
| Price | Lower | Higher |
| FIPS 140-2 | Level 1 | Level 2 |

## Architecture

```
+-------------------------------------------------------------------+
|                        Azure Key Vault                            |
|                                                                   |
|  +-----------------+  +-----------------+  +-----------------+    |
|  |     Secrets     |  |      Keys       |  |  Certificates   |    |
|  |                 |  |                 |  |                 |    |
|  | - Passwords     |  | - RSA           |  | - SSL/TLS       |    |
|  | - Conn strings  |  | - EC            |  | - Code signing  |    |
|  | - API keys      |  | - HSM-backed    |  | - Auto-renewal  |    |
|  +-----------------+  +-----------------+  +-----------------+    |
|                              |                                    |
|                     Access Control                                |
|  +-------------------------------------------------------------+  |
|  |  RBAC              |   Access Policies    |   Network      |  |
|  |  (Recommended)     |   (Legacy)           |   Rules        |  |
|  +-------------------------------------------------------------+  |
+-------------------------------------------------------------------+
```

## Access Control

### RBAC Roles (Recommended)

| Role | Permissions |
|------|-------------|
| Key Vault Administrator | Full access to secrets, keys, certificates |
| Key Vault Secrets Officer | Manage secrets |
| Key Vault Secrets User | Read secrets |
| Key Vault Certificates Officer | Manage certificates |
| Key Vault Crypto Officer | Manage keys |
| Key Vault Crypto User | Use keys for crypto operations |
| Key Vault Reader | Read metadata only |

```bash
# Assign role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee user@example.com \
  --scope /subscriptions/.../vaults/myVault
```

### Access Policies (Legacy)

```bash
az keyvault set-policy \
  --name myVault \
  --upn user@example.com \
  --secret-permissions get list \
  --key-permissions get list \
  --certificate-permissions get list
```

## Network Security

### Private Endpoint

```bash
# Create private endpoint
az network private-endpoint create \
  --name myKeyVaultPE \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id /subscriptions/.../vaults/myVault \
  --group-id vault \
  --connection-name myConnection
```

### Firewall Rules

```bash
# Allow specific IP
az keyvault network-rule add \
  --name myVault \
  --ip-address 1.2.3.4

# Allow VNet subnet
az keyvault network-rule add \
  --name myVault \
  --vnet-name myVNet \
  --subnet mySubnet

# Set default action
az keyvault update \
  --name myVault \
  --default-action Deny
```

## Soft Delete & Purge Protection

```
Delete -> Soft Deleted State -> Purge (Permanent Delete)
              |                        ^
              |    Retention Period    |
              |      (7-90 days)       |
              |                        |
              +---- Can Recover -------+

Purge Protection: Cannot purge during retention (even owner)
```

```bash
# Enable soft delete and purge protection
az keyvault update \
  --name myVault \
  --enable-soft-delete true \
  --enable-purge-protection true

# Recover deleted secret
az keyvault secret recover --vault-name myVault --name MySecret

# Purge (if allowed)
az keyvault secret purge --vault-name myVault --name MySecret
```

## Managed Identity Integration

### System-Assigned Identity

```bash
# App Service accessing Key Vault
az webapp identity assign --name myApp --resource-group myRG

# Grant access
az keyvault set-policy \
  --name myVault \
  --object-id <identity-object-id> \
  --secret-permissions get list
```

### In Application Code

```csharp
// .NET - Automatically uses managed identity
var client = new SecretClient(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential()
);

KeyVaultSecret secret = await client.GetSecretAsync("MySecret");
string value = secret.Value;
```

```python
# Python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://myvault.vault.azure.net/", credential=credential)

secret = client.get_secret("MySecret")
print(secret.value)
```

## Key Vault References

### App Service / Functions

```
# In App Settings, reference Key Vault secret:
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/MySecret/)

# Or with specific version:
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/MySecret/abc123)

# Or simplified:
@Microsoft.KeyVault(VaultName=myvault;SecretName=MySecret)
```

### ARM Templates

```json
{
  "type": "Microsoft.Web/sites/config",
  "properties": {
    "appSettings": [
      {
        "name": "DatabasePassword",
        "value": "[reference(resourceId('Microsoft.KeyVault/vaults/secrets', 'myVault', 'DbPassword')).secretUriWithVersion]"
      }
    ]
  }
}
```

## Certificate Management

### Auto-Renewal

```
Certificate Policy:
|-- Issuer: Self-signed / DigiCert / GlobalSign
|-- Subject: CN=myapp.example.com
|-- Validity: 12 months
|-- Key Type: RSA 2048
+-- Lifetime Actions:
    +-- Auto-renew at 80% of lifetime
```

### Integration with Services

```bash
# Use Key Vault certificate with App Service
az webapp config ssl import \
  --name myApp \
  --resource-group myRG \
  --key-vault myVault \
  --key-vault-certificate-name MyCert
```

## Key Operations

### Encryption/Decryption

```bash
# Encrypt data
az keyvault key encrypt \
  --name MyKey \
  --vault-name myVault \
  --algorithm RSA-OAEP \
  --value "SGVsbG8gV29ybGQ="

# Decrypt data
az keyvault key decrypt \
  --name MyKey \
  --vault-name myVault \
  --algorithm RSA-OAEP \
  --value "<encrypted-value>"
```

### Sign/Verify

```bash
# Sign data
az keyvault key sign \
  --name MyKey \
  --vault-name myVault \
  --algorithm RS256 \
  --digest "<base64-hash>"

# Verify signature
az keyvault key verify \
  --name MyKey \
  --vault-name myVault \
  --algorithm RS256 \
  --digest "<base64-hash>" \
  --signature "<signature>"
```

## Backup and Restore

```bash
# Backup secret
az keyvault secret backup \
  --vault-name myVault \
  --name MySecret \
  --file secret-backup.blob

# Restore secret
az keyvault secret restore \
  --vault-name myVault \
  --file secret-backup.blob

# Backup entire vault (requires Azure Backup)
# Use managed backup for full vault recovery
```

## CLI Quick Reference

```bash
# Create vault
az keyvault create \
  --name myVault \
  --resource-group myRG \
  --location eastus \
  --sku standard \
  --enable-rbac-authorization true

# Set secret
az keyvault secret set --vault-name myVault --name "MySecret" --value "secret123"

# Get secret
az keyvault secret show --vault-name myVault --name "MySecret" --query value -o tsv

# Create key
az keyvault key create --vault-name myVault --name "MyKey" --kty RSA --size 2048

# Import certificate
az keyvault certificate import --vault-name myVault --name "MyCert" --file cert.pfx

# Enable logging
az monitor diagnostic-settings create \
  --name "KeyVaultLogs" \
  --resource /subscriptions/.../vaults/myVault \
  --workspace /subscriptions/.../workspaces/myLA \
  --logs '[{"category":"AuditEvent","enabled":true}]'

# List deleted items
az keyvault secret list-deleted --vault-name myVault
```

## Exam Tips (AZ-104, AZ-204, AZ-305)

1. **RBAC vs Access Policies**: RBAC is recommended, more granular
2. **HSM-backed keys**: Requires Premium SKU
3. **Soft delete**: Enabled by default, 7-90 day retention
4. **Purge protection**: Cannot be disabled once enabled
5. **Key Vault references**: App Service can reference secrets directly
6. **Managed identity**: Best practice for accessing Key Vault
7. **Private endpoint**: Secure access without public exposure
8. **Certificate auto-renewal**: Configure lifetime actions
9. **Secrets versioning**: Each update creates new version
10. **Backup/restore**: Cannot restore to different tenant

## Gotchas

- Vault names are globally unique
- Soft delete is enabled by default (new vaults)
- Purge protection cannot be disabled once enabled
- Backup/restore only works within same Azure subscription
- Access policies limited to 1024 per vault
- Secret values limited to 25 KB
- Key operations have rate limits
- Private endpoint requires DNS configuration
- RBAC takes up to 10 minutes to propagate
- Cannot move vault between tenants with HSM keys

## Limits

| Resource | Limit |
|----------|-------|
| Vaults per subscription per region | 500 |
| Keys per vault | 500 |
| Secrets per vault | 500 |
| Certificates per vault | 500 |
| Secret/key versions | 500 per object |
| Secret size | 25 KB |
| Transactions (Standard) | 2,000/10s per vault |
| Transactions (Premium) | 6,000/10s per vault |
| RSA key operations | 1,000/10s (2048-bit) |
| EC key operations | 2,000/10s (P-256) |
