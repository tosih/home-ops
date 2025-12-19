# Home Assistant OAuth Setup with Pocket-ID

This guide explains how to complete the OAuth2 integration between Home Assistant and Pocket-ID.

## Overview

Home Assistant is now protected by OAuth2 Proxy, which authenticates users via Pocket-ID before allowing access to Home Assistant.

## Prerequisites Setup

### 1. Generate Cookie Secret

First, generate a random cookie secret for OAuth2 Proxy:

```bash
python3 -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
```

Save this value - you'll need it in step 3.

### 2. Create OAuth Client in Pocket-ID

1. Open Pocket-ID at https://pid.tosih.org
2. Log in with your admin account
3. Navigate to **OAuth Clients** or **Applications**
4. Click **Create New Client** / **Add Application**
5. Fill in the following details:

   **Application Name:** `Home Assistant`
   
   **Redirect URIs:**
   ```
   https://home.tosih.org/oauth2/callback
   ```
   
   **Grant Types:** `Authorization Code`
   
   **Scopes:** `openid`, `profile`, `email`
   
   **Client Type:** `Confidential`

6. Click **Create** or **Save**
7. **IMPORTANT:** Copy the generated **Client ID** and **Client Secret** immediately - you won't be able to see the secret again!

### 3. Store Credentials in 1Password

Store the OAuth credentials in your 1Password vault:

1. Open 1Password
2. Create a new item with the following structure:
   - **Title:** `home-assistant-oidc`
   - **Type:** API Credential or Secure Note
   - **Fields:**
     - `client_id`: [Paste the Client ID from Pocket-ID]
     - `client_secret`: [Paste the Client Secret from Pocket-ID]
     - `cookie_secret`: [Paste the cookie secret you generated in step 1]

3. Save the item in the same vault referenced by your `1password-sdk` ClusterSecretStore

### 4. Deploy the Configuration

The Kubernetes manifests are already in place. Commit and push the changes:

```bash
cd /Users/sohailahmed/Developer/home-ops
git add kubernetes/apps/home/home-assistant/
git commit -m "Add OAuth2 Proxy authentication to Home Assistant via Pocket-ID"
git push
```

### 5. Trigger Flux Reconciliation

Force Flux to sync the new configuration:

```bash
flux reconcile kustomization cluster-apps --with-source
```

### 6. Verify Deployment

Check that OAuth2 Proxy is running:

```bash
kubectl get pods -n home -l app=oauth2-proxy-home-assistant
kubectl get svc -n home oauth2-proxy-home-assistant
kubectl get httproute -n home home-assistant
```

### 7. Test Authentication

1. Open https://home.tosih.org in a browser
2. You should be redirected to Pocket-ID login
3. Log in with your Pocket-ID credentials
4. After successful authentication, you'll be redirected back to Home Assistant
5. Home Assistant may still ask for its own login - this is normal and provides an additional layer of security

## Architecture

```
User Browser
    ↓
External Gateway (Cilium)
    ↓
HTTPRoute: home.tosih.org
    ↓
OAuth2 Proxy (Port 4180)
    ↓ (authenticates via Pocket-ID)
Home Assistant Service (Port 8123)
```

## Troubleshooting

### OAuth2 Proxy won't start

Check the ExternalSecret:
```bash
kubectl get externalsecret -n home home-assistant-oidc
kubectl get secret -n home home-assistant-oidc-secret -o yaml
```

### Redirect loop or 403 errors

1. Verify the redirect URI in Pocket-ID matches exactly: `https://home.tosih.org/oauth2/callback`
2. Check OAuth2 Proxy logs:
   ```bash
   kubectl logs -n home -l app=oauth2-proxy-home-assistant
   ```

### Home Assistant shows "400 Bad Request"

This usually means the `trusted_proxies` configuration needs updating. Check that the OAuth2 Proxy pod IP range is trusted.

## Security Notes

- OAuth2 Proxy provides authentication at the gateway level
- Home Assistant's built-in authentication remains active as a second layer of security
- The cookie secret should be a random 32-byte base64-encoded string
- All traffic is encrypted via HTTPS through the Cilium gateway
