# Home Assistant OIDC Setup with Pocket ID

## Prerequisites

1. Create an OIDC client in Pocket ID (https://pid.tosih.org):
   - Client Name: `Home Assistant`
   - Redirect URIs: `https://home.tosih.org/auth/external/callback`
   - Grant Types: `authorization_code`
   - Response Types: `code`
   - Scopes: `openid`, `profile`, `email`

2. Save the Client ID and Client Secret to 1Password:
   - Create item: `home-assistant` in 1Password
   - Add field: `oidc_client_id` with the Client ID
   - Add field: `oidc_client_secret` with the Client Secret

## Configuration

Add the following to your Home Assistant `configuration.yaml`:

```yaml
homeassistant:
  auth_providers:
    - type: homeassistant
    - type: command_line
      command: /config/scripts/oidc_auth.sh
      meta: true

http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
```

Create a script at `/config/scripts/oidc_auth.sh`:

```bash
#!/bin/bash
# This script handles OIDC authentication for Home Assistant

CLIENT_ID="${OIDC_CLIENT_ID}"
CLIENT_SECRET="${OIDC_CLIENT_SECRET}"
ISSUER="https://pid.tosih.org"

# Handle authentication via OIDC
# This is a simplified example - you may need to adjust based on your needs
```

**Note**: Home Assistant's native OIDC support requires additional setup. You may want to consider using an external authentication provider like Authelia or OAuth2 Proxy in front of Home Assistant for better OIDC integration.

## Alternative: Use Trusted Networks

For now, you can use Home Assistant's built-in authentication with trusted networks as a stepping stone.
