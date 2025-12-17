# Pocket ID OIDC Integration Guide

## Overview

This guide covers setting up OIDC authentication with Pocket ID for your applications.

## Applications Ready for OIDC

### ✅ Jellyseerr (Easy - Native OIDC Support)
Jellyseerr has built-in OIDC support and is the easiest to configure.

### ⚠️ Home Assistant (Requires Auth Proxy)
Home Assistant doesn't have native OIDC support. You'll need to deploy an authentication proxy like:
- **Authelia** (Recommended for home labs)
- **OAuth2 Proxy** (Lightweight option)
- **Authentik** (Full-featured but heavier)

## Setup Instructions

### 1. Jellyseerr OIDC Setup

#### Step 1: Create OIDC Client in Pocket ID

1. Log into Pocket ID: https://pid.tosih.org
2. Go to **Applications** → **Create Application**
3. Configure the application:
   - **Name**: `Jellyseerr`
   - **Client Type**: `Confidential`
   - **Redirect URIs**: `https://requests.tosih.org/login/oidc/callback`
   - **Grant Types**: Select `Authorization Code`
   - **Scopes**: Select `openid`, `profile`, `email`
4. Save and note down:
   - **Client ID** (e.g., `jellyseerr-abc123`)
   - **Client Secret** (e.g., `secret_xyz789`)

#### Step 2: Save Credentials to 1Password

1. In 1Password, create a new item called `jellyseerr`
2. Add these fields:
   - Field name: `oidc_client_id` → Value: [Your Client ID]
   - Field name: `oidc_client_secret` → Value: [Your Client Secret]

#### Step 3: Apply Configuration

The ExternalSecret will automatically sync these credentials to Kubernetes.

#### Step 4: Configure Jellyseerr

1. Log into Jellyseerr: https://requests.tosih.org
2. Go to **Settings** → **General** → **Authentication**
3. Enable **OIDC Authentication**
4. Configure:
   - **OIDC Issuer URL**: `https://pid.tosih.org`
   - **OIDC Client ID**: [Will be auto-configured from secret]
   - **OIDC Client Secret**: [Will be auto-configured from secret]
   - **Button Label**: `Sign in with Pocket ID`
5. Save changes

### 2. Home Assistant OIDC Setup (Future)

Home Assistant requires an authentication proxy. Options:

#### Option A: Deploy Authelia (Recommended)
- Full-featured authentication server
- Supports OIDC, 2FA, and more
- Best for multiple applications

#### Option B: Deploy OAuth2 Proxy
- Lightweight
- Simple OIDC proxy
- Good for single application

#### Option C: Keep Native Auth
- Continue using Home Assistant's built-in authentication
- Consider trusted networks for local access

**Recommendation**: Start with Jellyseerr, then decide if you want to add Authelia for Home Assistant and other apps.

## Next Steps

1. ✅ Set up Jellyseerr OIDC (follow steps above)
2. ⏳ Decide on authentication strategy for Home Assistant
3. ⏳ Consider deploying Authelia if you want SSO for multiple apps

## Benefits of OIDC with Pocket ID

- **Single Sign-On**: Log in once, access multiple apps
- **Centralized User Management**: Manage all users in Pocket ID
- **Better Security**: OAuth2/OIDC is more secure than basic auth
- **Audit Logging**: Track authentication events
- **2FA Support**: Add two-factor authentication in Pocket ID
