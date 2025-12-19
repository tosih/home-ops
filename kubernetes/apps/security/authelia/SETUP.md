# Authelia Setup Guide

## Overview

Authelia is deployed as a Single Sign-On (SSO) authentication portal that integrates with Pocket ID as an OIDC identity provider.

## 1Password Secret Configuration

You need to create an item called `authelia` in 1Password with the following fields:

### Required Secrets

Generate these secrets using the commands below:

```bash
# NOTE: Database credentials are pulled from the 'postgres' 1Password item
# which contains POSTGRES_SUPER_USER and POSTGRES_SUPER_PASSWORD
# You don't need to add these to the authelia item

# Encryption keys (must be 64 hex characters each)
openssl rand -hex 64  # For storage_encryption_key
openssl rand -hex 64  # For session_secret
openssl rand -hex 64  # For oidc_hmac_secret

# Generate RSA private key for OIDC issuer
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:4096
cat private_key.pem  # Copy the entire contents including BEGIN/END lines
```

### 1Password Fields

Create these fields in your `authelia` 1Password item:

| Field Name | Type | Value | Notes |
|------------|------|-------|-------|
| `storage_encryption_key` | password | (64 hex chars) | Used to encrypt data at rest |
| `session_secret` | password | (64 hex chars) | Used to sign session cookies |
| `oidc_hmac_secret` | password | (64 hex chars) | Used for OIDC token signing |
| `oidc_issuer_private_key` | text | (RSA private key) | Multi-line RSA private key |
| `pocket_id_client_id` | text | (from Pocket ID) | OIDC client ID from Pocket ID |
| `pocket_id_client_secret` | password | (from Pocket ID) | OIDC client secret from Pocket ID |

## Setup Steps

### Step 1: Generate Encryption Keys

Run the following commands to generate the required secrets:

```bash
# Generate the three encryption keys
echo "storage_encryption_key: $(openssl rand -hex 64)"
echo "session_secret: $(openssl rand -hex 64)"
echo "oidc_hmac_secret: $(openssl rand -hex 64)"

# Generate OIDC issuer private key
openssl genpkey -algorithm RSA -out /tmp/authelia_private_key.pem -pkeyopt rsa_keygen_bits:4096
echo "oidc_issuer_private_key:"
cat /tmp/authelia_private_key.pem
rm /tmp/authelia_private_key.pem
```

### Step 2: Create Pocket ID OIDC Client (Optional - for future use)

If you want Authelia to authenticate users via Pocket ID instead of using its local user database:

1. Log into Pocket ID: https://pid.tosih.org
2. Go to **Applications** → **Create Application**
3. Configure:
   - **Name**: `Authelia`
   - **Client Type**: `Confidential`
   - **Redirect URIs**: `https://auth.tosih.org/api/oidc/callback`
   - **Grant Types**: `Authorization Code`
   - **Scopes**: `openid`, `profile`, `email`, `groups`
4. Save the Client ID and Client Secret

### Step 3: Create 1Password Item

1. In 1Password, create a new item called `authelia`
2. Add all the fields listed above with their generated values
3. Save the item

### Step 4: Verify Deployment

```bash
# Check pod status
kubectl -n security get pods -l app.kubernetes.io/name=authelia

# Check logs
kubectl -n security logs -l app.kubernetes.io/name=authelia --tail=50

# Access Authelia
# Navigate to: https://auth.tosih.org
```

## Default User Configuration

Authelia is configured to use a file-based authentication backend. To add users, you need to update the users database file.

### Create Initial Admin User

```bash
# Generate a password hash
docker run authelia/authelia:latest authelia crypto hash generate argon2 --password 'YourPasswordHere'

# The output will be a hash like:
# $argon2id$v=19$m=65536,t=3,p=4$...
```

Then, create a ConfigMap with users:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: authelia-users
  namespace: security
data:
  users_database.yml: |
    users:
      admin:
        displayname: "Admin User"
        password: "$argon2id$v=19$m=65536,t=3,p=4$..."  # Your generated hash
        email: admin@tosih.org
        groups:
          - admins
```

Apply this and restart Authelia pods to load the users.

## Accessing Authelia

- **URL**: https://auth.tosih.org
- **Default**: File-based authentication (no users configured initially)

## Next Steps

1. Configure OIDC clients in Authelia for your applications
2. Set up access control rules for different domains
3. Enable SMTP for email notifications (currently using filesystem)
4. Consider integrating with Pocket ID for centralized user management

## Troubleshooting

### Pod won't start

Check logs for database connection issues:
```bash
kubectl -n security logs -l app.kubernetes.io/name=authelia
```

### Database errors

Verify database was created:
```bash
kubectl -n databases get postgres authelia-db
```

### Secret issues

Verify external secret is syncing:
```bash
kubectl -n security get externalsecret authelia-secret
kubectl -n security describe externalsecret authelia-secret
```
