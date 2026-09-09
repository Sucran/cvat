---
title: 'CVAT XAIC Custom Edition'
linkTitle: 'XAIC Custom Edition'
weight: 8
description: 'Configure Keycloak OIDC SSO and basic authentication controls for the CVAT XAIC custom edition'
---

The CVAT XAIC custom edition adds server-side Keycloak OpenID Connect (OIDC) authentication
for deployments that use Keycloak as their central identity provider. Existing CVAT API Token,
Django Session, and local-account authentication remain compatible.

## Keycloak OIDC SSO

The integration uses the OIDC Authorization Code Flow. PKCE is enabled by default and the
callback creates or reuses the existing CVAT Django Session. Access tokens are not stored in the
browser by the CVAT frontend.

After SSO is enabled, the login page displays `Continue with Keycloak`. On the first successful
login, CVAT creates a local user. Subsequent logins are associated with the verified external
identity; an existing CVAT account with the same email can be linked without replacing its local
password.

The adapter requires an email claim and `email_verified: true`. An optional `email_domain` can
restrict access to users from a trusted domain.

## Authentication configuration

Create an `auth_config.yml` file and set `AUTH_CONFIG_PATH` to its location before starting or
restarting the CVAT backend:

```yaml
basic:
  registration:
    enabled: false
  login:
    enabled: true

sso:
  enabled: true
  enable_pkce: true
  identity_providers:
    - id: keycloak
      protocol: OIDC
      name: Keycloak
      server_url: https://keycloak.example.com/realms/<realm>/.well-known/openid-configuration
      client_id: cvat
      client_secret: <client-secret>
      email_domain: example.com
```

The supported Community Edition configuration currently accepts one provider with `id: keycloak`.
The `server_url` must point to the Keycloak OIDC discovery document. Do not commit the client
secret or a production `auth_config.yml` to the repository.

### Basic authentication switches

- `basic.registration.enabled: false` removes the registration API endpoint and the registration
  link from the UI.
- `basic.login.enabled: false` removes password login, password reset, and password change
  endpoints while keeping SSO available.
- At least one login method must remain enabled. Keep a documented break-glass administrator
  path before disabling local password login.

When `sso.enabled` is `false`, the provider configuration is ignored and the standard local
authentication behavior is retained.

## Keycloak client settings

Create an OpenID Connect client in the target Keycloak realm with:

- Client authentication enabled.
- Standard flow enabled.
- The redirect URI set to:

  ```text
  https://<cvat-host>/api/auth/oidc/keycloak/login/callback/
  ```

Use the actual public CVAT scheme and host. For a local HTTP deployment, replace the scheme and
host accordingly. The client must provide the `openid`, `profile`, and `email` scopes, and users
must have a verified email address.

## Helm deployment

For Kubernetes deployments, store the configuration in a Secret in the release namespace:

```bash
kubectl -n <namespace> create secret generic cvat-auth-config \
  --from-file=auth_config.yml=./auth_config.yml
```

Reference that Secret when installing or upgrading the chart:

```bash
helm upgrade --install cvat ./helm-chart \
  --set cvat.backend.server.authConfig.existingSecret=cvat-auth-config
```

The chart mounts the selected key at `/home/django/auth_config.yml` and sets
`AUTH_CONFIG_PATH` on the backend server. The Secret is intentionally managed outside Helm values
so client credentials are not copied into chart values or release history.

The relevant Helm values are:

| Value | Default | Description |
| --- | --- | --- |
| `cvat.backend.server.authConfig.existingSecret` | `""` | Existing Secret containing the auth configuration. |
| `cvat.backend.server.authConfig.key` | `auth_config.yml` | Key in the Secret to mount. |
| `cvat.backend.server.authConfig.mountPath` | `/home/django/auth_config.yml` | Container path. |

See [`helm-chart/README.md`](https://github.com/XiamenAIInnovationCenter/cvat/blob/develop/helm-chart/README.md)
for the chart-specific instructions.

## Operational notes

- Restart the CVAT backend after changing `auth_config.yml`.
- Ensure the public CVAT URL used in Keycloak exactly matches the callback URL, including the
  trailing slash.
- Keep the Keycloak issuer reachable from the CVAT backend container.
- Treat the client secret and the authentication Secret as production credentials.
