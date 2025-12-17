# Immich OIDC Setup with Pocket ID

## Prerequisites

Pocket ID must be accessible from the same network context as Immich. Since Immich is external-facing, Pocket ID has been configured as external as well.

## Setup Steps

### 1. Create OIDC Client in Pocket ID

1. Log into Pocket ID: https://pid.tosih.org
2. Navigate to **Applications** → **Create Application**
3. Configure the application:
   - **Name**: `Immich`
   - **Client Type**: `Confidential`
   - **Redirect URIs**: 
     - `https://photos.tosih.org/auth/login`
     - `https://photos.tosih.org/user-settings`
     - `https://photos.tosih.org/api/oauth/callback`
   - **Grant Types**: Select `Authorization Code`
   - **Scopes**: Select `openid`, `profile`, `email`
4. Save and note down:
   - **Client ID** (e.g., `immich-abc123`)
   - **Client Secret** (e.g., `secret_xyz789`)

### 2. Save Credentials to 1Password

1. In 1Password, create a new item called `immich`
2. Add these fields:
   - Field name: `oidc_client_id` → Value: [Your Client ID from step 1]
   - Field name: `oidc_client_secret` → Value: [Your Client Secret from step 1]

### 3. Apply Configuration

Once you've added the credentials to 1Password:

```bash
# The ExternalSecret will automatically sync the credentials
flux reconcile source git flux-system

# Wait for the secret to sync
kubectl get externalsecret -n cloud immich-oidc

# Verify the secret was created
kubectl get secret -n cloud immich-oidc-secret
```

### 4. Restart Immich

The configuration will be picked up automatically when Immich restarts:

```bash
kubectl rollout restart deployment/immich-server -n cloud
```

### 5. Test OIDC Login

1. Navigate to https://photos.tosih.org
2. You should see a "Sign in with Pocket ID" button
3. Click it to authenticate via Pocket ID
4. First-time users will be auto-registered (controlled by `IMMICH_OIDC_AUTO_REGISTER`)

## Configuration Details

The following OIDC environment variables are configured:

- `IMMICH_OIDC_ENABLED`: `true`
- `IMMICH_OIDC_ISSUER_URL`: `https://pid.tosih.org`
- `IMMICH_OIDC_BUTTON_TEXT`: `Sign in with Pocket ID`
- `IMMICH_OIDC_AUTO_REGISTER`: `true` - Automatically create user accounts on first login
- `IMMICH_OIDC_AUTO_LAUNCH`: `false` - Don't automatically redirect to OIDC (allows local admin login)
- `IMMICH_OIDC_CLIENT_ID`: From 1Password secret
- `IMMICH_OIDC_CLIENT_SECRET`: From 1Password secret

## Important Notes

- **Pocket ID is now external-facing** to support Immich OIDC authentication
- Ensure strong passwords and enable 2FA in Pocket ID for all users
- The first user to log in via OIDC will need admin privileges assigned manually
- Keep `IMMICH_OIDC_AUTO_LAUNCH: false` to maintain local admin access

## Troubleshooting

If OIDC login doesn't work:

1. Check that the secret synced:
   ```bash
   kubectl describe externalsecret -n cloud immich-oidc
   ```

2. Verify environment variables in the pod:
   ```bash
   kubectl exec -n cloud deployment/immich-server -- env | grep OIDC
   ```

3. Check Immich logs:
   ```bash
   kubectl logs -n cloud deployment/immich-server --tail=100
   ```

4. Verify Pocket ID is accessible externally:
   ```bash
   curl -I https://pid.tosih.org
   ```
