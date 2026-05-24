# Microsoft Entra ID (formerly Azure AD)

> Cloud-based identity and access management service for securing access to applications and resources.

## Overview

Microsoft Entra ID is a cloud-based identity and access management (IAM) service. It enables employees to sign in and access external resources (Microsoft 365, Azure portal, SaaS applications) and internal resources (apps on corporate network, cloud apps).

## Key Concepts

| Term | Definition |
|------|------------|
| Tenant | Dedicated instance of Entra ID for an organization |
| Directory | Container for users, groups, and applications |
| User | Identity for a person or service |
| Group | Collection of users for access management |
| Application | App registered for authentication |
| Service Principal | Identity for application/service |
| Managed Identity | Azure-managed service identity |

## License Tiers

| Feature | Free | P1 | P2 |
|---------|------|----|----|
| Users and groups | Yes | Yes | Yes |
| SSO (unlimited apps) | Yes | Yes | Yes |
| MFA | Yes | Yes | Yes |
| Conditional Access | No | Yes | Yes |
| Self-service password reset | Cloud only | Yes | Yes |
| Dynamic groups | No | Yes | Yes |
| Identity Protection | No | No | Yes |
| Privileged Identity Management | No | No | Yes |
| Access Reviews | No | No | Yes |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Microsoft Entra ID Tenant                     │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                     Directory                               │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────────┐ │ │
│  │  │  Users  │  │ Groups  │  │  Apps   │  │   Devices    │ │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └──────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌───────────────────────────▼───────────────────────────────┐ │
│  │                     Features                                │ │
│  │  • Authentication (SSO, MFA)                               │ │
│  │  • Conditional Access                                       │ │
│  │  • Identity Protection                                      │ │
│  │  • Privileged Identity Management                          │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │  Azure   │        │ Microsoft│        │  Custom  │
    │ Portal   │        │   365    │        │   Apps   │
    └──────────┘        └──────────┘        └──────────┘
```

## User Types

| Type | Description | Source |
|------|-------------|--------|
| **Member** | Employee/internal user | Cloud or synced |
| **Guest** | External user (B2B) | Invited via email |
| **Service Account** | Application identity | Cloud-created |

### Create User

```bash
# Create cloud user
az ad user create \
  --display-name "John Doe" \
  --user-principal-name john@contoso.onmicrosoft.com \
  --password "SecureP@ss123!" \
  --force-change-password-next-sign-in

# Invite guest user
az ad user invite \
  --user-email-address guest@external.com \
  --redirect-url https://myapp.com
```

## Groups

### Types

| Type | Membership | Use Case |
|------|------------|----------|
| **Security** | Assigned or Dynamic | Access control |
| **Microsoft 365** | Assigned or Dynamic | Collaboration |
| **Assigned** | Manual membership | Static groups |
| **Dynamic** | Rule-based auto-membership | Auto-managed |

### Dynamic Group Rules

```
# All users in Marketing department
(user.department -eq "Marketing")

# All full-time employees
(user.employeeType -eq "FullTime")

# All users with Manager title
(user.jobTitle -contains "Manager")

# Users in specific location
(user.country -eq "United States") -and (user.state -eq "Washington")

# Devices with specific OS
(device.deviceOSType -eq "Windows")
```

## Application Registration

### App Registration vs Enterprise App

| Registration | Enterprise App |
|--------------|----------------|
| Define your app | Instance of app in tenant |
| Set permissions | Assign users/groups |
| Configure auth | Grant consent |
| Developer focus | Admin focus |

### Authentication Flows

```
Authorization Code Flow (Web apps):
User → App → Entra ID → Authorization Code → App → Token

Client Credentials (Service-to-service):
App → Entra ID → Access Token → API

Device Code (CLI/IoT):
App → Device Code → User authenticates → App polls → Token

ROPC (Legacy - avoid):
App + Username/Password → Token (Not recommended)
```

### Register Application

```bash
# Create app registration
az ad app create \
  --display-name "My Application" \
  --sign-in-audience AzureADMyOrg

# Create service principal
az ad sp create --id <app-id>

# Create secret
az ad app credential reset \
  --id <app-id> \
  --append \
  --display-name "Production Secret"
```

## Service Principals

Types of service principals:

| Type | Description |
|------|-------------|
| **Application** | Local representation of app registration |
| **Managed Identity** | Azure-managed, no credentials to manage |
| **Legacy** | Older apps without registration |

## Managed Identities

### System-Assigned

```bash
# Enable on VM
az vm identity assign --name myVM --resource-group myRG

# Enable on App Service
az webapp identity assign --name myApp --resource-group myRG
```

### User-Assigned

```bash
# Create identity
az identity create --name myIdentity --resource-group myRG

# Assign to VM
az vm identity assign \
  --name myVM \
  --resource-group myRG \
  --identities /subscriptions/.../userAssignedIdentities/myIdentity
```

## Conditional Access

### Components

```
IF:                          THEN:
├── Users/Groups             ├── Allow
├── Cloud apps               │   └── Require MFA
├── Conditions               │   └── Require device compliance
│   ├── Sign-in risk         │   └── Require app protection
│   ├── Device platforms     └── Block
│   ├── Locations
│   ├── Client apps
│   └── Device state
```

### Common Policies

| Policy | Condition | Control |
|--------|-----------|---------|
| Require MFA for admins | Admin roles | Require MFA |
| Block legacy auth | Client apps = other | Block |
| Require compliant device | All apps | Require compliant device |
| Block risky sign-ins | High risk | Block |
| Named locations | Trusted IPs | Allow without MFA |

## Multi-Factor Authentication (MFA)

### Methods

| Method | Type |
|--------|------|
| Microsoft Authenticator | Push/Code |
| SMS | Code via text |
| Phone call | Voice verification |
| FIDO2 security key | Hardware key |
| Windows Hello | Biometric |
| OATH tokens | Hardware/software |

### MFA Configuration

```
Security Defaults (Basic):
- MFA required for all users
- Block legacy authentication
- Free, tenant-wide

Per-User MFA (Legacy):
- Enable/enforce per user
- Less flexible

Conditional Access (Recommended):
- Policy-based
- Granular control
- Requires P1 license
```

## Identity Protection (P2)

### Risk Policies

| Policy | Description |
|--------|-------------|
| **User Risk** | Compromised credentials detected |
| **Sign-in Risk** | Suspicious sign-in activity |

### Risk Levels

| Level | Examples |
|-------|----------|
| High | Leaked credentials, malware-linked IP |
| Medium | Unfamiliar sign-in properties |
| Low | Anonymous IP address |

```
Risk-Based Conditional Access:
IF sign-in risk = High
THEN Block access

IF user risk = Medium
THEN Require password change
```

## Privileged Identity Management (PIM) (P2)

Just-in-time privileged access.

```
Standard RBAC:                     PIM:
User ─── Always Admin Role         User ─── Eligible for Role
                                            │
                                            ▼ Activate (justification)
                                            │
                                   User ─── Active Role (time-limited)
                                            │
                                            ▼ Expires
                                            │
                                   User ─── Eligible for Role
```

### PIM Settings

| Setting | Purpose |
|---------|---------|
| Activation duration | How long role is active |
| Approval required | Require manager approval |
| Justification | Require reason for activation |
| MFA | Require MFA to activate |
| Notification | Alert on activation |

## Azure RBAC Integration

```
Microsoft Entra ID                    Azure Resources
┌─────────────────┐                   ┌─────────────────┐
│     Users       │───Authentication──▶│  Subscriptions  │
│     Groups      │                   │  Resource Groups│
│   Applications  │───RBAC Roles──────▶│    Resources    │
└─────────────────┘                   └─────────────────┘
```

### Role Assignment

```bash
# Assign role at subscription scope
az role assignment create \
  --role "Contributor" \
  --assignee user@contoso.com \
  --scope /subscriptions/<subscription-id>

# Assign to group
az role assignment create \
  --role "Reader" \
  --assignee-object-id <group-object-id> \
  --scope /subscriptions/<subscription-id>/resourceGroups/myRG
```

## Hybrid Identity

### Sync Methods

| Method | Description |
|--------|-------------|
| **Cloud-only** | Users created in Entra ID |
| **Entra Connect** | Sync from on-prem AD |
| **Entra Cloud Sync** | Lightweight agent-based sync |

### Password Options

| Option | Description |
|--------|-------------|
| Password Hash Sync | Hash synced to cloud |
| Pass-through Auth | Auth against on-prem AD |
| Federation (ADFS) | On-prem ADFS server |

## CLI Quick Reference

```bash
# List users
az ad user list --output table

# Get user
az ad user show --id user@contoso.com

# List groups
az ad group list --output table

# Create group
az ad group create --display-name "MyGroup" --mail-nickname "mygroup"

# Add user to group
az ad group member add --group "MyGroup" --member-id <user-object-id>

# List applications
az ad app list --output table

# List service principals
az ad sp list --all --output table

# Get tenant info
az account show

# List directory roles
az rest --method GET --url "https://graph.microsoft.com/v1.0/directoryRoles"
```

## Exam Tips (AZ-104, AZ-305)

1. **Tenant**: Single organization, globally unique
2. **Guest users**: B2B collaboration, external access
3. **Dynamic groups**: Auto-membership via rules (P1)
4. **Conditional Access**: Policy-based access control (P1)
5. **MFA**: Security defaults (free) or Conditional Access (P1)
6. **PIM**: Just-in-time access for privileged roles (P2)
7. **Identity Protection**: Risk-based policies (P2)
8. **Managed Identity**: Best for Azure resource auth
9. **Service Principal**: App identity for automation
10. **Entra Connect**: Sync on-prem AD to cloud

## Gotchas

- Tenant name (*.onmicrosoft.com) cannot be changed
- Users can be members of multiple tenants
- Guest users have limited permissions by default
- Dynamic groups only available with P1/P2
- Conditional Access requires P1 at minimum
- PIM requires P2 license
- MFA registration takes effect immediately
- Deleted users retained for 30 days
- Application secrets expire (max 2 years)
- RBAC roles are separate from Entra directory roles

## Limits

| Resource | Limit |
|----------|-------|
| Objects per directory | 500,000 (free), 500,000+ (paid) |
| Groups per user | 500 |
| Users per group | No limit |
| App registrations | Unlimited |
| Service principals | No limit |
| Custom domains | 900 |
| Conditional Access policies | Unlimited |
| Named locations | 195 |
| Directory roles | 70+ built-in |
