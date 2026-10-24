# Prerequisites and Starting Point

This document describes the environment assumed by the rest of the guide.

The actual implementation is covered in the following parts. This first document only explains what is already in place, which components are involved, and which local configuration files are relevant.

The setup shown here is based on Angie with Lua support, `lua-resty-openidc`, Keycloak, Navidrome External Authentication, and CrowdSec AppSec through the Lua bouncer.

It is not a stock deployment, but the important parts are straightforward once the request flow is understood.

---

## Goals

This setup provides the Holy Trinity of Navidrome server access.
The final setup should support all three authentication paths at the same time:

```text
Browser
    -> OIDC
    -> Keycloak
    -> Angie
    -> Navidrome External Auth
```

```text
Wavio
    -> OIDC Authorization Code + PKCE
    -> Keycloak
    -> Bearer access token
    -> Angie
    -> Navidrome External Auth
```

```text
Traditional Subsonic client
    -> /rest/*
    -> Angie
    -> Navidrome native Subsonic authentication
```

The main goal of this guide is to add the second path without breaking either of the existing ones. 

In particular:

- normal browser login through OIDC must continue to work;
- Wavio should use OIDC/PKCE and bearer-token authentication;
- existing Subsonic/OpenSubsonic clients must continue to work on `/rest/*`;
- public Navidrome shares must remain public;
- CrowdSec/AppSec must remain in front of all protected traffic if it is part of the deployment;
- clients must never be able to spoof `Remote-User`.

The hybrid authentication logic is the main subject of this guide.

The initial Angie, Keycloak, Navidrome, and browser-OIDC setup is assumed to already exist. Explaining the complete setup of each component is outside the scope of this guide.

---

## Prerequisites

Before starting the Wavio-specific part, the following should already be working:

- Angie with Lua support;
- `lua-resty-openidc`;
- Keycloak;
- a working OIDC login for browser access;
- Navidrome;
- Navidrome External Authentication;
- HTTPS on the public Navidrome hostname;
- traditional Subsonic/OpenSubsonic access to `/rest/*`.

## Scope and Assumptions

This guide is not intended to replace the existing documentation for Angie, Keycloak, Navidrome, `lua-resty-openidc`, or CrowdSec.

There are already good installation and configuration guides available for each of these components. You do not need to be an expert in all of them to reproduce this setup, but the individual parts should already be working before following the hybrid configuration in this guide.

In particular, this guide does not cover:

- installing Angie;
- installing or configuring Keycloak from scratch;
- installing Navidrome;
- setting up HTTPS or DNS;
- installing `lua-resty-openidc`;
- building a complete browser OIDC setup from scratch;
- installing CrowdSec or AppSec/WAF;
- general reverse-proxy configuration.

Following the upstream documentation for those components is usually enough to establish the required baseline.

Some familiarity with Lua is useful, however.

The authentication decision happens inside Lua code running directly in the reverse proxy's access phase. You do not need to be a Lua developer, but you should be able to read the code well enough to understand which requests are accepted, rejected, redirected, or forwarded.

This is especially important for authentication code.

As with any configuration copied from the Internet:

> Do not run or deploy scripts you do not understand.

Read the Lua code, adapt it to your environment, and verify what each branch does before placing it in front of an Internet-facing service.

---

## CrowdSec AppSec / WAF

CrowdSec is not required for the Wavio hybrid authentication setup.
It only appears in this guide because this deployment already uses the CrowdSec Lua bouncer together with AppSec/WAF in Angie's access phase:

```text
Request
    -> CrowdSec / AppSec
    -> OIDC / bearer authentication
    -> Navidrome
```

If CrowdSec AppSec is not part of your setup, all CrowdSec-related configuration shown in this guide can be ignored.

### Local CrowdSec modification

This deployment contains one custom change to the CrowdSec Lua bouncer.
In the version used here, an allowed or no-decision request could terminate Lua access processing with:

```lua
ngx.exit(ngx.DECLINED)
```

That prevented authentication logic placed later in the same `access_by_lua_block` from running.
The local behavior was changed to:

```lua
return
```

This allows requests accepted by CrowdSec to continue through the remaining authentication logic:

```text
CrowdSec allows request
    -> continue access phase
    -> OIDC / Wavio authentication
```

CrowdSec can still block requests when remediation is required. Only the successful/no-decision path was changed.

This custom behavior may therefore be visible in CrowdSec-related files shown later in the guide. In this deployment the modification is also kept persistent across CrowdSec updates by a small systemd-based mechanism, but that mechanism is outside the scope of this guide.
If your CrowdSec integration already allows subsequent Lua access-phase logic to continue normally, no equivalent modification is required.

## Components and Their Roles

Before looking at the actual configuration, it helps to understand which component is responsible for which part of the authentication flow.

None of the individual components has to understand the complete hybrid setup. Angie and the Lua logic connect the existing authentication mechanisms and decide which one applies to a request.

---

### Angie

Angie is the reverse proxy in front of Navidrome and acts as the central authentication boundary.

Its responsibilities are:

- terminate HTTPS;
- proxy requests to Navidrome;
- run the Lua access logic;
- integrate the existing CrowdSec/AppSec checks;
- start the existing browser OIDC flow when required;
- validate Wavio bearer tokens;
- remove untrusted identity headers supplied by clients;
- pass an authenticated user to Navidrome through `Remote-User`.

Angie does not authenticate users by itself. Authentication is delegated to Keycloak, while the Lua code decides which authentication path should be used for a request.

Conceptually:

```text
Client
    -> Angie
    -> authentication decision
    -> Navidrome
```

---

### Lua and `lua-resty-openidc`

Lua provides the routing and authentication glue inside Angie.

`lua-resty-openidc` is used for two related but separate purposes.

The existing setup already uses it for interactive browser authentication:

```text
Browser
    -> Angie
    -> OIDC redirect
    -> Keycloak
    -> OIDC callback
    -> authenticated user
```

The hybrid extension adds a second use:

```text
Wavio request
    -> Authorization: Bearer <token>
    -> Angie
    -> lua-resty-openidc JWT validation
    -> authenticated user
```

Both paths eventually produce the same trusted identity header:

```text
Remote-User
```

and both use the same Keycloak claim as the Navidrome username:

```text
preferred_username
```

This is important because browser authentication and Wavio authentication should resolve to the same Navidrome account.

The Lua code also owns the main hybrid-routing decision:

```text
Bearer token present?
    |
    +-- yes -> validate Wavio bearer token
    |
    +-- no  -> use the existing authentication path
```

`/rest/*` receives additional handling because it must also remain compatible with traditional Subsonic clients.

---

### Keycloak

Keycloak is the OpenID Connect provider and token issuer.

It is used by both OIDC authentication paths:

```text
Browser OIDC
    -> Keycloak
```

and:

```text
Wavio Authorization Code + PKCE
    -> Keycloak
```

For Wavio, Keycloak provides:

- Authorization Code Flow;
- PKCE support;
- access tokens;
- refresh/offline access;
- the user identity claim;
- the `wavio` audience used during bearer validation.

The Wavio client is configured as a public client and does not require a client secret.

The access token is validated by Angie before any identity is passed to Navidrome.

Keycloak therefore authenticates the user, while Angie decides whether the resulting token is acceptable for the protected Navidrome resource.

This guide uses Keycloak as the IdP, but the same approach can be adapted to other OIDC providers.

---

### Navidrome

Navidrome is the final application and supports two authentication mechanisms that are important for this setup.

#### External Authentication

When Angie sends a trusted:

```text
Remote-User: alice
```

Navidrome uses External Authentication and treats the request as belonging to that user.

Only Angie is configured as a trusted External Auth source.

#### Native Subsonic authentication

If no trusted external-auth identity is supplied to `/rest/*`, Navidrome can use its normal Subsonic authentication instead:

```text
u + p
```

or:

```text
u + t + s
```

This behavior is the key to the hybrid setup.

Angie does not need to emulate Subsonic authentication or identify individual Subsonic clients.

It only has to make sure that requests intended for native Subsonic authentication reach Navidrome **without** a trusted `Remote-User` header.

The decision then happens naturally inside Navidrome:

```text
trusted Remote-User present
    -> External Auth

trusted Remote-User absent
    -> normal Subsonic authentication
```

---

### Wavio

Wavio is the OIDC-capable OpenSubsonic client.

One especially useful part of its implementation is that SSO support is discovered dynamically.

When a server address is entered, Wavio first requests:

```text
/.well-known/oauth-protected-resource
```

The response tells Wavio which authorization server protects the resource.

Wavio can then offer SSO and perform:

```text
Authorization Code + PKCE
```

against Keycloak.

After a successful login, its authenticated application requests carry:

```http
Authorization: Bearer <access_token>
```

This makes the hybrid routing deterministic.

Angie does not need to detect Wavio through its User-Agent or any custom application-specific header.

For the relevant authenticated requests, the distinction is simply:

```text
Bearer present
    -> bearer authentication path
```

The dynamic protected-resource discovery step is therefore an important part of what makes this setup practical.

---

### CrowdSec / AppSec

CrowdSec remains the security layer in front of the authentication logic in this deployment.

Its job is independent from OIDC:

```text
CrowdSec / AppSec
    -> security decision

OIDC / bearer validation
    -> authentication decision
```

The hybrid Wavio logic does not replace or bypass CrowdSec.

The only unusual part of this setup is the access-phase behavior described in the previous section: a request accepted by CrowdSec must continue into the later Lua authentication logic instead of terminating processing.

If CrowdSec is not part of your environment, it can simply be omitted from the architecture below.

---

## How the Components Fit Together

The important separation of responsibilities is:

| Component | Responsibility |
| --- | --- |
| Angie | Reverse proxy and authentication boundary |
| Lua | Request routing and authentication decision logic |
| `lua-resty-openidc` | Browser OIDC and bearer JWT validation |
| CrowdSec / AppSec | Request security and remediation |
| Keycloak | User authentication and token issuance |
| Navidrome External Auth | Accept authenticated identity from Angie |
| Navidrome Subsonic Auth | Authenticate traditional Subsonic clients |
| Wavio | OIDC/PKCE client using bearer access tokens |

The hybrid setup does not replace any of these systems.

It connects their existing behavior.

---

## Complete Request Flow

The resulting architecture can be represented as follows:

```text
                                      +----------------------+
                                      |       Keycloak       |
                                      |                      |
                                      | OIDC / PKCE / Tokens |
                                      +----------+-----------+
                                                 ^
                                                 |
                          +----------------------+----------------------+
                          |                                             |
                          |                                             |
                   Browser OIDC                                  Wavio PKCE
                          |                                             |
                          |                                             |
+-------------------------+---------------------------------------------+------+
|                                  Clients                                     |
+-----------+-----------------------------+-----------------------------+------+
            |                             |                             |
            | Browser                     | Wavio                       | Subsonic
            |                             |                             |
            v                             v                             v
+--------------------------------------------------------------------------------+
|                                     Angie                                      |
|                                                                                |
|                            CrowdSec / AppSec                                    |
|                                    |                                           |
|                                    v                                           |
|                         Lua authentication logic                               |
|                                                                                |
|      Browser request             Bearer request              /rest/* no Bearer |
|             |                          |                              |         |
|             v                          v                              v         |
|      existing OIDC             validate JWT                  clear/omit       |
|             |                  issuer / audience              Remote-User      |
|             |                  preferred_username                    |         |
|             |                          |                              |         |
|             +-------------+------------+                              |         |
|                           |                                           |         |
|                           v                                           |         |
|                  set trusted Remote-User                              |         |
|                           |                                           |         |
+---------------------------+-------------------------------------------+---------+
                            |                                           |
                            v                                           v
                 +--------------------------------------------------------------+
                 |                          Navidrome                            |
                 |                                                              |
                 |   Remote-User present                    no Remote-User       |
                 |           |                                    |             |
                 |           v                                    v             |
                 |    External Authentication              Subsonic Auth         |
                 |                                               u/p            |
                 |                                            or u/t/s           |
                 +--------------------------------------------------------------+
```

There are therefore three final authentication paths.

### Browser

```text
Browser
    -> Angie
    -> CrowdSec / AppSec
    -> existing browser OIDC
    -> Keycloak
    -> preferred_username
    -> Remote-User
    -> Navidrome External Auth
```

### Wavio

Before login:

```text
Wavio
    -> /.well-known/oauth-protected-resource
    -> discover Keycloak
    -> Authorization Code + PKCE
```

After login:

```text
Wavio
    -> Authorization: Bearer <access_token>
    -> Angie
    -> CrowdSec / AppSec
    -> bearer JWT validation
    -> preferred_username
    -> Remote-User
    -> Navidrome External Auth
```

### Traditional Subsonic client

```text
Subsonic client
    -> /rest/*
    -> Angie
    -> CrowdSec / AppSec
    -> no bearer token
    -> no trusted Remote-User forwarded
    -> Navidrome
    -> native Subsonic authentication
```

The important part is that both OIDC paths converge on the same Navidrome External Auth mechanism, while requests without an external identity can still use Navidrome's existing Subsonic authentication.

## Trust Boundary and `Remote-User`

The most important security assumption in this setup is that `Remote-User` is trusted **only when Angie creates it after successful authentication**.

Navidrome External Authentication treats the configured user header as an authenticated identity when the request originates from a trusted source.

That means this:

```http
Remote-User: alice
```

is not just metadata.

Inside the External Auth trust boundary, it effectively means:

```text
This request has already been authenticated as alice.
```

For that reason, clients must never be allowed to choose this value themselves.

Only the reverse proxy should be present in Navidrome's:

```text
ExtAuth.TrustedSources
```

and any client-supplied `Remote-User` header must be removed before authentication decisions are made.

Whenever this guide mentions `Remote-User`, pay close attention to whether the header is being:

- removed;
- created after successful authentication;
- intentionally left absent;
- or cleared before proxying to Navidrome.

It is worth checking this one time too often rather than one time too little.

A mistake here can turn an authentication-routing error into an authentication bypass.

### Example: `/rest/*`

The `/rest/*` route supports both Wavio and traditional Subsonic authentication.

For a successfully authenticated Wavio request:

```text
valid bearer token
    -> Angie validates token
    -> Angie sets Remote-User
    -> Navidrome External Auth
```

For a traditional Subsonic request:

```text
no bearer token
    -> Angie removes any client-supplied Remote-User
    -> Angie does not create a new Remote-User
    -> Navidrome receives no external identity
    -> Navidrome uses native Subsonic authentication
```

This distinction is essential.

If a client-supplied `Remote-User` were allowed through on the second path, the client could potentially bypass normal Subsonic authentication.

### Example: `/share/*`

Public shares are intentionally unauthenticated.

They therefore must not inherit an authenticated identity from another request path or from a client-supplied header.

A share location should explicitly make sure no external identity reaches Navidrome:

```nginx
proxy_set_header Remote-User "";
```

The intended flow is:

```text
/share/*
    -> no External Auth identity
    -> public Navidrome share handling
```

The general rule throughout this guide is:

```text
Authenticated OIDC request
    -> Angie may set Remote-User

Unauthenticated or native-auth request
    -> Angie must not provide Remote-User

Client-supplied Remote-User
    -> never trusted
```

If you modify the routes shown later in this guide, verify this behavior again afterwards.

---

## Important Endpoints

The hybrid setup only depends on a small number of HTTP paths.

Each path has a clearly defined role, and the authentication behavior differs depending on whether the request carries a bearer token.

---

### `/.well-known/oauth-protected-resource`

This endpoint is used by Wavio to discover whether the server supports SSO and which authorization server protects it.

Example:

```text
https://music.example.com/.well-known/oauth-protected-resource
```

It returns metadata similar to:

```json
{
  "resource": "https://music.example.com",
  "authorization_servers": [
    "https://auth.example.com/realms/example"
  ]
}
```

This endpoint must remain publicly accessible.

It is queried before Wavio has authenticated, so it must not be protected by browser OIDC or bearer authentication.

Its purpose is only discovery.

---

### `/redirect_uri`

This is the callback endpoint used by the existing browser OIDC flow.

After authentication at Keycloak, the browser is redirected back to this endpoint so `lua-resty-openidc` can complete the authorization flow and create the authenticated session.

Conceptually:

```text
Browser
    -> Keycloak
    -> /redirect_uri
    -> existing OIDC session
```

This endpoint belongs to the existing browser authentication setup and is not specific to Wavio.

---

### `/auth/logout`

This is the logout endpoint used by the existing browser OIDC implementation.

It is also part of the existing browser authentication setup and is not modified by the Wavio integration.

---

### `/`

The normal Navidrome web interface continues to use the existing browser OIDC flow.

```text
/
    -> browser OIDC
    -> Keycloak
    -> Remote-User
    -> Navidrome
```

No Wavio-specific handling is required for the root path.

---

### `/api/*`

Navidrome's application API can be accessed by both the browser application and Wavio.

The authentication path depends on whether a bearer token is present.

```text
/api/* + Bearer
    -> Wavio bearer validation

/api/* without Bearer
    -> existing browser OIDC
```

A bearer token that is present but invalid results in `401`.

It does not fall back to browser authentication.

---

### `/app/*`

`/app/*` follows the same routing model as `/api/*`.

```text
/app/* + Bearer
    -> Wavio bearer validation

/app/* without Bearer
    -> existing browser OIDC
```

This allows Wavio to authenticate using its access token while normal browser requests continue through the existing OIDC flow.

---

### `/rest/*`

`/rest/*` is the most important endpoint in the hybrid design.

It is used by both:

- Wavio;
- traditional Subsonic/OpenSubsonic clients.

The authentication behavior is therefore:

```text
/rest/* + valid Bearer
    -> validate Wavio token
    -> set Remote-User
    -> Navidrome External Auth
```

```text
/rest/* without Bearer
    -> remove any client-supplied Remote-User
    -> do not inject an external identity
    -> Navidrome native Subsonic authentication
```

```text
/rest/* + invalid Bearer
    -> HTTP 401
```

This is the only endpoint where the absence of a bearer token does **not** result in browser OIDC.

That distinction is what preserves compatibility with existing Subsonic clients.

---

### `/share/*`

Public Navidrome shares remain outside the proxy authentication system.

```text
/share/*
    -> Navidrome
```

No browser OIDC or Wavio bearer validation is applied.

Any external-auth identity header should be cleared before proxying the request.

---

## Endpoint Overview

The complete routing behavior can be summarized as follows:

```text
Client request
     |
     v
+-------------------------------------------------------+
|                       Angie                           |
+-------------------------------------------------------+
     |
     +-- /.well-known/oauth-protected-resource
     |       |
     |       +--> public SSO discovery
     |
     +-- /share/*
     |       |
     |       +--> public Navidrome share
     |
     +-- /
     |       |
     |       +--> existing browser OIDC
     |
     +-- /redirect_uri
     |       |
     |       +--> browser OIDC callback
     |
     +-- /auth/logout
     |       |
     |       +--> browser OIDC logout
     |
     +-- /api/*
     |       |
     |       +-- Bearer present
     |       |       |
     |       |       +--> Wavio bearer validation
     |       |
     |       +-- no Bearer
     |               |
     |               +--> existing browser OIDC
     |
     +-- /app/*
     |       |
     |       +-- Bearer present
     |       |       |
     |       |       +--> Wavio bearer validation
     |       |
     |       +-- no Bearer
     |               |
     |               +--> existing browser OIDC
     |
     +-- /rest/*
             |
             +-- valid Bearer
             |       |
             |       +--> Wavio bearer validation
             |       +--> Remote-User
             |       +--> Navidrome External Auth
             |
             +-- no Bearer
             |       |
             |       +--> no Remote-User
             |       +--> Navidrome Subsonic Auth
             |
             +-- invalid Bearer
                     |
                     +--> HTTP 401
```

The resulting distinction is simple:

```text
Browser routes without Bearer
    -> existing OIDC

Wavio routes with Bearer
    -> bearer JWT validation

/rest/* without Bearer
    -> native Subsonic authentication

Public discovery and shares
    -> no proxy/OIDC authentication
```

This routing model is the basis for the configuration shown in the next part of the guide.

## Affected Local Files

The deployment involves a small number of local Angie configuration files.

The exact paths may differ between installations, but their roles should be equivalent.

| File | Purpose |
| --- | --- |
| `oidc_auth.lua` | Existing browser OIDC logic and the new Wavio bearer-token validation |
| `oidc_secrets.lua` | Local OIDC secrets loaded by the Lua module |
| `protected-access.conf` | Shared access-phase logic containing CrowdSec and authentication routing |
| Angie Lua shared-dictionary configuration | OIDC discovery, JWKS, session, and JWT verification caches |
| Navidrome vhost | HTTP locations for browser access, `/rest/*`, `/share/*`, and protected-resource discovery |
| CrowdSec Lua bouncer file | Contains the local allow/no-decision continuation change used by this deployment |

The next part of the guide shows the relevant custom files in full where practical, with secrets and private infrastructure replaced by example values.

`oidc_secrets.lua` itself is not shown with real values.

A public example may reference values such as:

```lua
secrets.client_id
secrets.client_secret
secrets.session_secret
```

but actual credentials should remain local.

The CrowdSec source modification is included only because it is visible in this deployment's request flow. It is not required for Wavio and can be ignored entirely when CrowdSec is not used.

---

## Baseline Check Before Continuing

Before adding any Wavio-specific configuration, verify the existing setup first.

This gives you a known-good starting point and makes later troubleshooting much easier.

The following should already work:

- browser access redirects unauthenticated users to Keycloak;
- a successful browser login reaches the correct Navidrome user;
- the browser OIDC path uses `preferred_username` as the Navidrome identity;
- Angie removes untrusted client-supplied `Remote-User` headers;
- only Angie is trusted by Navidrome for External Authentication;
- traditional Subsonic/OpenSubsonic clients can authenticate through `/rest/*`;
- `/rest/*` does not require browser OIDC;
- public `/share/*` links remain accessible without authentication;
- CrowdSec/AppSec still evaluates protected traffic if it is part of the deployment;
- a request accepted by CrowdSec continues into the later OIDC logic.

A useful baseline test for `/rest/*` is a request without Subsonic credentials:

```bash
curl -i \
  'https://music.example.com/rest/ping.view?v=1.16.1&c=test'
```

It should reach Navidrome rather than redirecting to Keycloak.

An authentication error from Navidrome, for example because the `u` parameter is missing, is a useful result here: it proves that the request reached the native Subsonic authentication path.

You should also verify that a forged external-auth header is not accepted:

```bash
curl -i \
  -H 'Remote-User: admin' \
  'https://music.example.com/rest/ping.view?v=1.16.1&c=test'
```

This must not authenticate the request as `admin`.

If these baseline behaviors are correct before the Wavio changes are introduced, the next part can focus entirely on adding the new bearer-authentication path without having to debug the existing stack at the same time.

