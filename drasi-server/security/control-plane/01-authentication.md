# Drasi Server Authentication

**Status:** Draft

## Overview

Drasi Server will validate JSON Web Token (JWT) access tokens from one external OpenID Connect (OIDC) identity provider and establish a trusted caller identity. Authorization will separately decide what that caller may do.

This design covers the Server API, Admin UI, and the authentication contract shared with other clients and component endpoints. It does not introduce a Drasi account database, password service, or server-side browser session system.

Backward compatibility and migration are not requirements.

## Scope and boundaries

| Concern | Responsibility |
|---|---|
| Caller authentication | Server-owned token validation and authenticated caller context |
| Permissions and continuing data access | Roles and permissions design; authentication alone grants no resource access |
| Source/Reaction workload credentials | Instance credential design; existing `IdentityProvider::get_credentials()` supplies outbound credentials, not caller identity |
| Component listeners, webhook proofs, and protocol-specific guards | Outside this document; integration must use the shared host authority rather than independent plugin login systems |
| Native code admission and loading | Separate Code & Plugin Plane |
| Initialization, access recovery, and full development-mode lifecycle | Control-plane administration design |

Listener, webhook-verification, and protocol-specific integration are outside this document's scope. These concerns will be addressed in a separate ingress/egress design, outside the control-plane scope.

Ordinary embedded DrasiLib use does not require OIDC login. SDK interfaces connect endpoints to host authority; they must not create another identity system.

## Production authentication

### Deployment contract

The host configures one HTTPS issuer, one Drasi API audience, and optionally one trusted group claim. The Admin UI additionally needs a public client ID, registered callback URL, and scopes for obtaining API access tokens; no browser client secret.

The API audience must be distinct from the browser client ID and reserved for API access tokens. The provider must issue tokens matching this contract; OIDC support alone does not establish compatibility with arbitrary access-token formats.

Login trust settings are host-managed, not component settings. Resource APIs, token contents, and hidden overrides must not weaken them. Missing or invalid security configuration must not create an unsecured deployment or an administrator.

Remote production access requires HTTPS through trusted deployment infrastructure. Proxy identity, forwarding headers, CORS, and a private backend address are not caller authentication.

### Access tokens and Principal

Ordinary API clients use the HTTP `Authorization` header with the `Bearer` scheme. ID tokens, opaque tokens, API passwords, and URL tokens are not substitutes.

| Validation | Required behavior |
|---|---|
| Signature | Verify using the configured issuer's trusted signing keys and an explicit allowlist of supported asymmetric algorithms; reject unsigned tokens and algorithm substitution |
| Issuer | Require an exact match to the configured issuer |
| Audience and purpose | Require the configured API audience and the issuer's API access-token contract, not its browser ID-token contract |
| Lifetime | Require expiration; enforce expiration and any not-before claim with a bounded clock-skew allowance |
| Subject | Require a nonempty subject; display names and email addresses are not identity |
| Groups | Use only the configured trusted claim's exact string values; absence supplies no membership, and unsupported claim shapes must not invent membership |

The Principal contains exact issuer, subject, and configured groups; the request context also carries credential expiration. Other token claims and payload fields do not become Drasi role grants. `whoami` returns this normalized identity and expiration, never the token.

Authentication must precede protected resource handling across Server entry points, including default-instance aliases and programmatic construction.

### Signing keys and provider outages

Discover metadata and the JSON Web Key Set (JWKS) through the configured issuer using verified HTTPS. Verify the discovered issuer; token-supplied URLs must not select verification infrastructure.

Cache trusted keys with bounded freshness. Successful refresh updates the trusted set; removed keys are not an indefinite fallback. During an outage, still-usable cached keys may verify valid tokens. Otherwise fail closed. Refresh failures must not extend cache validity indefinitely.

Bound input/response sizes, fetch time, cache growth, and refresh frequency, including for unknown key IDs. Libraries, supported algorithm enumeration, and numeric bounds are implementation choices that must be documented and tested.

Drasi does not poll accounts or introspect every token. IdP-only changes follow token lifetime; local revocation follows the authorization contract. Pipelines do not depend on the creator remaining logged in or successful OIDC requests.

## Browser login and credential lifetime

The Admin UI uses an established OIDC library and authorization code flow with Proof Key for Code Exchange (PKCE). Validate transaction state and OIDC responses; use registered callbacks, not untrusted headers or arbitrary return URLs.

- Keep access and refresh tokens only in memory, never in browser persistent storage, cookies, URLs, or logs.
- Short-lived transaction state, including the PKCE verifier, may survive redirects. Clear it on completion or expiry; this does not permit persisting tokens.
- Allow library-managed renewal with memory-only refresh tokens; otherwise reauthorize. A reload loses tokens and returns through the authorization flow.
- Logout clears local credentials, protected UI state, and streams, not global IdP sessions, every issued token, or pipelines.
- Do not fetch protected data before authentication or reuse another identity's cached data/subscriptions.

Authenticated requests include direct fetches, schemas, and Server-Sent Events (SSE). Native `EventSource` cannot attach an Authorization header; use header-capable streaming, not a cookie or URL-token exception.

Close protected streams at credential expiration, even while idle. Renewal does not extend an existing stream's credential lifetime: reconnect with the new token. Authorization changes may end access earlier.

## Development mode and client scope

Explicit private development mode supplies a fixed Principal with Server Admin authority through normal authorization. Its identity remains stable across restarts so ownership does not change.

Clients need no OIDC token, but cannot select this mode through requests or failed login. Localhost does not imply development mode. Validation, component scope, credential protections, and normal policy remain in effect.

Audit identifies development identity explicitly. Control-plane administration defines mode initialization/change; private listener exposure is outside this document's scope. This contract permits neither public fixed-administrator endpoints nor bypassing provider-webhook verification.

| Client | Initial support |
|---|---|
| Admin UI | Production PKCE login/renewal and token-free private development access |
| Scripts and service clients | Supply a suitable access token obtained through the configured provider's supported flow |
| VS Code extension | Private development mode only; clearly report secured deployments as unsupported instead of repeatedly retrying unauthenticated requests |
| Component applications | Follow the shared caller contract; protocol and browser integration are outside this document's scope |

Secured VS Code support is deferred, not a Server-side client-type ban. Later support can supply tokens to API, schema, and stream requests using this contract. No token-entry, login, renewal, or persistence feature is required initially.

Drasi does not bundle or provision an identity provider. An optional secured quickstart/test fixture may provide Keycloak or another provider. Private development needs none and does not demonstrate multi-user authorization.

## HTTP surface and failure contract

Protected routes are the default. Public exceptions must be explicit and contain no protected deployment or component information.

| Surface | Access |
|---|---|
| Health, API version discovery, and any public readiness response | Public, minimal and aggregate |
| Admin UI static assets and login/callback shell | Public; no embedded protected data |
| `GET /api/v1/auth/config` | Public authentication bootstrap: mode and, when applicable, issuer, API audience, public client ID, registered callback, and requested scopes; no secrets or grants |
| `GET /api/v1/whoami` | Authenticated; returns the caller's normalized Principal and credential expiration without requiring a resource grant |
| Management APIs, configuration, plugin inventory/schemas, diagnostics, and results | Authenticated, followed by the applicable authorization checks |
| OpenAPI and Swagger functionality | Authenticated; data-free static assets may be public, but schema access and interactive API calls must use the same authenticated client flow |

Both authentication endpoints have no request body. Successful responses use HTTP `200`, `application/json`, and the existing `ApiResponse` envelope: `success: true`, a `data` object, and `error: null`. The data fields below use camelCase. Failure responses use the existing `ErrorResponse` shape and the status rules below, not a successful response envelope.

### Authentication bootstrap response

`GET /api/v1/auth/config` returns:

| Data field | Type | Contract |
|---|---|---|
| `mode` | `"production"` or `"development"` | The deployment's selected mode; not a client-selectable setting |
| `oidc` | Object or `null` | Required object in production; explicitly `null` in development |
| `oidc.issuer` | String | Configured HTTPS issuer |
| `oidc.audience` | String | Configured Drasi API audience |
| `oidc.clientId` | String or `null` | Browser public client ID; `null` when the Admin UI is disabled |
| `oidc.callbackUrl` | String or `null` | Registered browser callback; `null` when the Admin UI is disabled |
| `oidc.scopes` | Array of strings | Requested browser-login/API scopes; `[]` when the Admin UI is disabled |

Production example with the Admin UI enabled:

```json
{
  "success": true,
  "data": {
    "mode": "production",
    "oidc": {
      "issuer": "https://login.example.com",
      "audience": "drasi-api",
      "clientId": "drasi-admin-ui",
      "callbackUrl": "https://drasi.example.com/ui/callback",
      "scopes": ["openid", "drasi-api"]
    }
  },
  "error": null
}
```

The example's identifiers, callback, and scopes are illustrative host-configured values, not built-in defaults. Production without the Admin UI still returns its issuer and audience, with `clientId: null`, `callbackUrl: null`, and `scopes: []`.

Development example:

```json
{
  "success": true,
  "data": {
    "mode": "development",
    "oidc": null
  },
  "error": null
}
```

The `oidc` field is not omitted, and development must not invent an issuer or other OIDC settings.

### Caller identity response

`GET /api/v1/whoami` returns:

| Data field | Type | Contract |
|---|---|---|
| `principal` | Object | The normalized caller identity, not a token or list of role bindings |
| `principal.issuer` | String | Exact validated issuer, or the fixed development Principal's issuer label |
| `principal.subject` | String | Exact validated subject, or the fixed development Principal's subject |
| `principal.groups` | Array of strings | Normalized configured group values; `[]` when there are none |
| `expiresAt` | RFC 3339 UTC string or `null` | Validated access-token expiration in production; explicitly `null` in development |

Production example:

```json
{
  "success": true,
  "data": {
    "principal": {
      "issuer": "https://login.example.com",
      "subject": "user-123",
      "groups": ["analysts"]
    },
    "expiresAt": "2030-01-01T12:00:00Z"
  },
  "error": null
}
```

Development example:

```json
{
  "success": true,
  "data": {
    "principal": {
      "issuer": "urn:drasi:development",
      "subject": "server-admin",
      "groups": []
    },
    "expiresAt": null
  },
  "error": null
}
```

The development issuer/subject above illustrate the implementation's fixed identity, not an OIDC provider or caller-supplied identity. The response must match the Principal actually used by authorization and remain stable across restarts. `groups` and `expiresAt` are always present; no token expiration is fabricated for development.

### Failures and access termination

| Situation | Behavior |
|---|---|
| Missing, malformed, expired, or invalid credential | `401`; generic authentication error, no protected resource details |
| Verification temporarily unavailable with no usable trusted keys | `503` authentication unavailable; fail closed without treating the credential as known-invalid |
| Valid Principal lacks resource authority | Follow the roles policy's visible-target `403` and hidden/missing-target `404` behavior; do not treat this as a login failure |
| Stream expires or caller loses effective permission | End affected access; a connection or session identifier is not an independent grant |
| Login or key retrieval fails | No alternate issuer, anonymous access, or development-mode fallback |

Audit authentication failures and security-relevant session events under the shared policy. Never log tokens, authorization codes, PKCE verifiers, or credential-bearing headers/URLs. Audit failure never authorizes bypassing authentication.

## Threats, risks, and decisions

| Threat / risk | Simplified example | What is at risk | Mitigation or accepted limit |
|---|---|---|---|
| Forged or incorrectly targeted token | Caller sends a self-signed token or one for another API | Impersonation and unauthorized access | Validate signature, issuer, audience, purpose, and lifetime; no token-selected trust |
| ID token used as an API credential | Browser login token is sent to a management route | Confusing client login with API access | Distinct client ID/API audience and access-token validation |
| Login response substitution | Attacker supplies a crafted callback URL | Wrong-identity login or code misuse | OIDC library, PKCE, transaction-state checks, registered callbacks |
| Stolen bearer credential | Copied token is replayed before expiry | Access within the Principal's current permissions | HTTPS, memory-only browser tokens, no token logging/URL transport; replay until expiry remains possible, subject to local revocation |
| Browser script compromise | Malicious script reads a refresh token in memory | Session compromise | Safe rendering and browser security controls; memory-only storage is not script isolation |
| Stale trust or issuer outage | Removed key stays cached or refresh fails | Excess trust lifetime or unavailable login | Bounded freshness, authoritative refresh, fail-closed verification; no real-time external revocation guarantee |
| Trusted authority compromise | Issuer mints an attacker's token or host changes login trust | Defeat of the authentication boundary | Accepted trust boundary: the issuer, host, and admitted native code are trusted, not contained by this design |
| Key-refresh abuse | Many requests contain different unknown key IDs | Authentication-service resource exhaustion | Bound inputs, fetches, cache size, and refresh frequency |
| Unprotected alternative route | An alias or builder-created API omits authentication | Bypass of the secured entry point | Apply the same contract across protected routers and construction paths |
| Long-lived access outlasts credentials | Idle stream remains open after token expiry | Continued unauthorized disclosure | Enforce stream expiration and permission changes, not just handshake checks |
| Development escape hatch | Caller supplies a development header or failed login enables admin fallback | Unauthenticated administration | Explicit host-selected private mode; no caller selection or downgrade |
| Workload/caller identity confusion | Source credentials are treated as the requester's identity | Incorrect attribution and permissions | Separate inbound authentication from outbound credential acquisition |
| Plugin listener bypass | Another port does not use host security | Direct endpoint bypass | Require ingress integration with shared authority; main-router middleware alone is insufficient |

## Implementation boundaries and verification

Implement the host authenticator/context, protected Server paths, and Admin UI request/stream handling. SDK integration exposes host authority, not duplicated plugin trust or permission stores. Rust modules, traits, FFI layouts, and library selection remain implementation details.

Verification must cover:

- Valid user and service tokens; incorrect signatures, algorithms, issuer, audience, purpose, lifetime, and subject; missing or unsupported group claims.
- Key rotation, bounded caching, unknown key IDs, issuer outages, and absence of any insecure fallback.
- Public bootstrap versus protected routes, aliases, programmatic construction, OpenAPI, and authenticated callers with no resource grants.
- Authentication response envelopes and field types in both modes, including production without the Admin UI, empty groups, explicit nulls, and UTC token expiration.
- Browser login/renewal/logout, reload behavior, identity changes, and absence of token persistence or logging.
- Stream expiry while idle, reconnect with fresh credentials, and authorization-driven closure.
- Token-free private development and VS Code access, plus rejection of caller attempts to enable that mode on a secured deployment.

Authorization, audit persistence, listener integration, and full development setup remain separately owned. Management-API authentication alone does not secure the whole deployment.

## Alternatives considered

Local accounts duplicate the IdP; cookie-backed Server sessions add another credential lifecycle; independent plugin authentication duplicates trust and policy. None is required.

First-class VS Code login is deferred: its initial development workflow needs no OIDC setup, while future secured support can use the existing client contract.

## References

- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
