# Authentication with Pocket ID

## Overview

This cluster uses **Pocket ID** as the primary OIDC (OpenID Connect) identity provider for Single Sign-On (SSO) across applications.

**Pocket ID URL**: <https://pid.tosih.org>

## Benefits of OIDC Authentication

- **Single Sign-On (SSO)**: Log in once, access multiple apps
- **Centralized User Management**: Manage all users in Pocket ID
- **Better Security**: OAuth2/OIDC is more secure than basic authentication
- **Audit Logging**: Track authentication events
- **2FA Support**: Add two-factor authentication in Pocket ID

## Applications with OIDC Support

### ✅ Immich (Photos)

Immich has native OIDC support and is configured to use Pocket ID.

**URL**: <https://photos.tosih.org>

#### Configuration Details

The following environment variables are configured:

- `IMMICH_OIDC_ENABLED`: `true`
- `IMMICH_OIDC_ISSUER_URL`: `https://pid.tosih.org`
- `IMMICH_OIDC_BUTTON_TEXT`: `Sign in with Pocket ID`
- `IMMICH_OIDC_AUTO_REGISTER`: `true` - Automatically create user accounts on first login
- `IMMICH_OIDC_AUTO_LAUNCH`: `false` - Don't automatically redirect (allows local admin login)

#### First-Time Setup

1. **Create OIDC Client in Pocket ID**:
   - Log into Pocket ID: <https://pid.tosih.org>
   - Navigate to **Applications** → **Create Application**
   - Configure:
     - **Name**: `Immich`
     - **Client Type**: `Confidential`
     - **Redirect URIs**:
       - `https://photos.tosih.org/auth/login`
       - `https://photos.tosih.org/user-settings`
       - `https://photos.tosih.org/api/oauth/callback`
     - **Grant Types**: `Authorization Code`
     - **Scopes**: `openid`, `profile`, `email`

2. **Save Credentials to 1Password**:
   - Create item: `immich` in 1Password
   - Add field: `oidc_client_id` → Client ID from Pocket ID
   - Add field: `oidc_client_secret` → Client Secret from Pocket ID

3. **Sync and Restart**:

   ```bash
   # ExternalSecret will automatically sync credentials
   flux reconcile source git flux-system
   
   # Restart Immich to pick up changes
   kubectl rollout restart deployment/immich-server -n cloud
   ```

4. **Test Login**:
   - Navigate to <https://photos.tosih.org>
   - Click "Sign in with Pocket ID"
   - First-time users will be auto-registered

### ⚠️ Applications Without Native OIDC Support

The following applications do not have native OIDC support:

- **Jellyseerr**: Uses Jellyfin/Plex authentication
- **Home Assistant**: Requires authentication proxy (Authelia/OAuth2 Proxy)
- ***arr apps** (Sonarr, Radarr, Prowlarr): No OIDC support
- **Homepage**: No authentication

#### Options for Non-OIDC Apps

**Option 1: Keep on Internal Gateway** (Recommended)

- Use the `internal` gateway (10.0.50.101)
- Protected by network firewall
- Simplest and most secure for homelab

**Option 2: Deploy oauth2-proxy**

- Acts as authentication proxy in front of the app
- Authenticates users via Pocket ID OIDC
- More complex but provides SSO

**Option 3: Use App's Native Authentication**

- Keep using each app's built-in auth
- Consider using a password manager for credentials

## Gateway Configuration

The cluster has two gateways for different security contexts:

### External Gateway (10.0.50.102)

- **Purpose**: Public internet access
- **Apps**: Pocket ID, Immich, Flux webhook
- **Security**: Requires strong authentication (OIDC preferred)

### Internal Gateway (10.0.50.101)

- **Purpose**: Internal network access only
- **Apps**: Most homelab apps (Jellyseerr, Plex, Homepage, *arr apps)
- **Security**: Protected by network/firewall

## Adding OIDC to a New Application

### Step 1: Create OIDC Client in Pocket ID

1. Log into Pocket ID: <https://pid.tosih.org>
2. Go to **Applications** → **Create Application**
3. Configure:
   - **Name**: Your application name
   - **Client Type**: `Confidential`
   - **Redirect URIs**: Your app's callback URL(s)
   - **Grant Types**: `Authorization Code`
   - **Scopes**: `openid`, `profile`, `email`
4. Save and note the Client ID and Client Secret

### Step 2: Store Credentials in 1Password

1. Create a new item in 1Password with your app name
2. Add fields:
   - `oidc_client_id`: The Client ID from Pocket ID
   - `oidc_client_secret`: The Client Secret from Pocket ID

### Step 3: Configure Application

Configure your application to use:

- **OIDC Issuer**: `https://pid.tosih.org`
- **Client ID**: Retrieved from 1Password via ExternalSecret
- **Client Secret**: Retrieved from 1Password via ExternalSecret
- **Scopes**: `openid profile email`

### Step 4: Test

1. Navigate to your application
2. Look for "Sign in with Pocket ID" or similar
3. Authenticate via Pocket ID
4. Verify user is created/authenticated

## Troubleshooting OIDC Issues

### Application Can't Connect to Pocket ID

Check that Pocket ID is accessible:

```bash
curl -I https://pid.tosih.org
```

### Credentials Not Syncing

Check ExternalSecret status:

```bash
kubectl get externalsecret -n <namespace>
kubectl describe externalsecret -n <namespace> <name>
```

### OIDC Login Fails

1. Verify redirect URIs match exactly in Pocket ID
2. Check application logs:

   ```bash
   kubectl logs -n <namespace> deployment/<app> --tail=100
   ```

3. Ensure scopes are correctly configured

### Users Not Auto-Registering

- Check if auto-registration is enabled in your app
- Some apps require manual user creation even with OIDC

## Security Best Practices

1. **Enable 2FA** in Pocket ID for all users
2. **Use strong passwords** in Pocket ID
3. **Limit redirect URIs** to only what's necessary
4. **Review user access** regularly in Pocket ID
5. **Monitor authentication logs** for suspicious activity
6. **Keep Pocket ID updated** via Flux automation

## Current OIDC Integrations

| Application | OIDC Status | Gateway | Notes |
|------------|-------------|---------|-------|
| Pocket ID | Provider | External | Identity provider itself |
| Immich | ✅ Configured | External | Auto-registration enabled |
| Jellyseerr | ❌ No Support | Internal | Uses Plex/Jellyfin auth |
| Home Assistant | ❌ No Support | Internal | Native auth only |
| Plex | ❌ No Support | Internal | Plex Pass account |
| *arr Apps (Sonarr, Radarr, Prowlarr) | ❌ No Support | Internal | Basic auth |
| qBittorrent | ❌ No Support | Internal | Basic auth |
| NZBGet | ❌ No Support | Internal | Basic auth |
| Homepage | ❌ No Support | Internal | No auth |
| AdGuard Home | ❌ No Support | Internal | Native auth |
| Linkding | ❌ No Support | Internal | Native auth |
| Memos | ❌ No Support | Internal | Native auth |
| Romm | ❌ No Support | Internal | Native auth |
| Syncthing | ❌ No Support | Internal | Native auth |
| ImmichFrame | ❌ No Support | Internal | Read-only display |
| TinyAuth | Service | Internal | Authentication microservice |

## Kubernetes OIDC Authentication

You can configure `kubectl` to authenticate to the Kubernetes API server using Pocket ID OIDC credentials. This provides SSO for cluster access and better audit logging.

### Prerequisites

1. Install the `kubelogin` plugin:

   ```bash
   # macOS
   brew install int128/kubelogin/kubelogin
   
   # Linux
   kubectl krew install oidc-login
   ```

2. Create an OIDC client in Pocket ID for Kubernetes access (if not already created)

### Kubeconfig Setup

Add the following to your `~/.kube/config` (replace sensitive values as needed):

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <BASE64_ENCODED_CA_CERT>
    server: https://10.0.50.50:6443
  name: home-ops-oidc
contexts:
- context:
    cluster: home-ops-oidc
    namespace: default
    user: oidc
  name: home-ops-oidc
current-context: home-ops-oidc
kind: Config
users:
- name: oidc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://pid.tosih.org
      - --oidc-client-id=<YOUR_CLIENT_ID>
      - --oidc-client-secret=<YOUR_CLIENT_SECRET>
      - --oidc-extra-scope=groups
      - --oidc-extra-scope=email
      - --oidc-extra-scope=name
      - --oidc-extra-scope=sub
      - --oidc-extra-scope=email_verified
      command: kubectl
      env: null
      interactiveMode: Never
      provideClusterInfo: false
```

### Getting the CA Certificate

Extract the cluster CA certificate from your existing kubeconfig:

```bash
# From talos-generated kubeconfig
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
```

Or get it directly from Talos:

```bash
talosctl -n 10.0.50.50 get secrets -o yaml | grep -A 1 "ca.crt"
```

### Testing OIDC Authentication

```bash
# Switch to OIDC context
kubectl config use-context home-ops-oidc

# Test access (will trigger browser login on first use)
kubectl get nodes
```

On first authentication, `kubelogin` will open a browser window to authenticate via Pocket ID.

### RBAC Configuration

Grant permissions to OIDC users by creating RoleBindings or ClusterRoleBindings:

```yaml
# Example: Grant cluster-admin to specific user
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: User
  name: <user-email-from-pocket-id>
  apiGroup: rbac.authorization.k8s.io
```

## Future Enhancements

- Consider deploying oauth2-proxy for apps without OIDC support
- Explore Pocket ID's advanced features (groups, roles)
- Set up audit logging for authentication events
- Implement session timeout policies
- Configure Kubernetes API server OIDC flags in Talos
