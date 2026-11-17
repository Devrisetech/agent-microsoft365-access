# Authentication — Microsoft Graph REST contract

This page is the **REST API contract** an AI agent must satisfy to call Microsoft Graph with an
application identity (OAuth 2.0 client-credentials / app-only grant). It describes **what** to send
and **what** comes back. It deliberately does **not** prescribe *how* to connect — the choice of HTTP
client, SDK, or token cache is left to the agent and its host.

> [!IMPORTANT]
> The client secret is sensitive. Read it only from the environment variable
> `MCP_MS_OFFICE365_CLIENT_SECRET`. Never print, log, echo, hard-code, or commit it.

## Identifiers the caller needs

| Value | Placeholder | Secret? | Source |
| --- | --- | --- | --- |
| Directory (tenant) ID | `<TENANT_ID>` | No | App registration — Overview page |
| Application (client) ID | `<CLIENT_ID>` | No | App registration — Overview page |
| Client secret | `MCP_MS_OFFICE365_CLIENT_SECRET` (env var) | **Yes** | App registration — Certificates & secrets |
| Service user (UPN) | `<BOT_UPN>` | No | The provisioned bot/service mailbox |

See [`entra-admin-setup.md`](entra-admin-setup.md) for how an administrator produces these.

## Step 1 — Request an access token

```http
POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id=<CLIENT_ID>
&scope=https://graph.microsoft.com/.default
&client_secret=<value of MCP_MS_OFFICE365_CLIENT_SECRET>
&grant_type=client_credentials
```

| Body parameter | Required | Value |
| --- | --- | --- |
| `client_id` | Yes | `<CLIENT_ID>` |
| `scope` | Yes | `https://graph.microsoft.com/.default` — requests every application permission the tenant admin has consented to for this app |
| `client_secret` | Yes | The secret value, from `MCP_MS_OFFICE365_CLIENT_SECRET` |
| `grant_type` | Yes | `client_credentials` |

There is no user, browser, or MFA prompt in this flow.

Grounded in Microsoft Learn — *Get access without a user*:
<https://learn.microsoft.com/en-us/graph/auth-v2-service> and the OAuth 2.0 client-credentials grant:
<https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant>.

## Step 2 — Token response

```json
{
  "token_type": "Bearer",
  "expires_in": 3599,
  "ext_expires_in": 3599,
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6..."
}
```

| Field | Meaning |
| --- | --- |
| `access_token` | The bearer token to present to Microsoft Graph. |
| `token_type` | Always `Bearer`. |
| `expires_in` | Token lifetime in seconds (commonly ~3600, i.e. ~60 minutes). |
| `ext_expires_in` | Extended lifetime used for resiliency when the token service is unavailable. |

> [!NOTE]
> Token minting is itself rate-limited. Reuse a valid token until close to `expires_in` rather than
> requesting one per call. The caching strategy is the agent's choice.

## Step 3 — Call Microsoft Graph

Base URL: `https://graph.microsoft.com/v1.0` (the `v1.0` endpoint is generally available; `/beta` is
preview — prefer `v1.0`). Present the token in the `Authorization` header:

```http
GET https://graph.microsoft.com/v1.0/users/<BOT_UPN>/messages
Authorization: Bearer <access_token>
```

> [!IMPORTANT]
> **App-only has no signed-in user, so `/me` does not work.** Every request must name a user
> explicitly, e.g. `/users/<BOT_UPN>/...` for that user's mail, calendar, or drive. Use the service
> account's UPN (`<BOT_UPN>`) as the `user-id`.

## Throttling contract

Microsoft Graph throttles per several overlapping limits (per app across all tenants, per tenant, per
app per tenant, and per service). The first limit reached triggers throttling; the documented global
ceiling is **130,000 requests per 10 seconds per app across all tenants**.

When throttled, Graph returns **HTTP `429 Too Many Requests`** with a `Retry-After` response header.
The caller must:

1. Wait the number of seconds given in `Retry-After`.
2. Retry the request.
3. If `429` recurs, keep honoring `Retry-After`; if the header is absent, fall back to exponential
   backoff.

Grounded in Microsoft Learn — throttling guidance
<https://learn.microsoft.com/en-us/graph/throttling> and service limits
<https://learn.microsoft.com/en-us/graph/throttling-limits>.

## Network hosts to allow

An agent host with an outbound network policy must allow HTTPS to exactly these two hosts:

| Host | Purpose |
| --- | --- |
| `login.microsoftonline.com` | Token endpoint (Step 1) |
| `graph.microsoft.com` | Graph API (Step 3) |

## What not to use

- **Do not** use the ROPC / password grant (`grant_type=password`). It is deprecated and breaks under
  MFA and Conditional Access.
- **Do not** embed the client secret in source, config, `.mcp.json`, or a committed `.env`.

## References

- Get access without a user (app-only) — <https://learn.microsoft.com/en-us/graph/auth-v2-service>
- OAuth 2.0 client-credentials grant — <https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant>
- Throttling guidance — <https://learn.microsoft.com/en-us/graph/throttling>
- Throttling limits — <https://learn.microsoft.com/en-us/graph/throttling-limits>
