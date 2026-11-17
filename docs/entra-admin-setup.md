# How to set up Office 365 access in Microsoft Entra (administrator)

This guide is for a **Microsoft Entra administrator**. It walks you through registering the
application, creating a client secret, granting the Microsoft Graph permissions, and provisioning the
service account that an AI coding agent will use for headless (app-only) access to Office 365.

You perform this **once per tenant**. The output is four values you hand to the operator who runs the
[AI-powered installation](../prompts/enable-office365.md): the **tenant ID**, the **client ID**, the
**client secret**, and the **service user UPN**. The three credentials are delivered as the
environment variables `GRAPH_TENANT_ID`, `GRAPH_CLIENT_ID`, and `GRAPH_CLIENT_SECRET`.

> [!WARNING]
> The client secret is the one sensitive value here. Deliver it only through a secret manager or a
> host/CI secret store as the environment variable `GRAPH_CLIENT_SECRET`. Never paste it into a
> repository, a chat, a ticket, or a file under version control.

## Prerequisites

- A Microsoft Entra role that can register apps and consent to application permissions:
  **Global Administrator** or **Privileged Role Administrator** (consent requires one of these).
- Access to the Microsoft Entra admin center: <https://entra.microsoft.com>.
- A licensed mailbox/OneDrive for the service account (or the ability to create and license one).

## Step 1 — Register the application

1. In the Entra admin center, go to **Identity → Applications → App registrations → New registration**.
2. **Name**: something descriptive, e.g. `AI Agent Office 365 Access`.
3. **Supported account types**: *Accounts in this organizational directory only* (single tenant).
4. **Redirect URI**: leave blank — the client-credentials flow needs none.
5. Select **Register**.

On the app's **Overview** page, copy the **Application (client) ID** and **Directory (tenant) ID**.
These are not secret.

## Step 2 — Create a client secret

1. In the app, open **Certificates & secrets → Client secrets → New client secret**.
2. Give it a description and an **expiry** (6–12 months is a sensible default; set a rotation
   reminder).
3. Select **Add**, then **copy the secret _Value_ immediately** — it is shown only once. Do not use
   the *Secret ID*.

> [!NOTE]
> For long-lived setups, prefer a **certificate** credential over a client secret.

Hand the secret to the operator through your secret store as `GRAPH_CLIENT_SECRET`.

## Step 3 — Add Microsoft Graph application permissions and grant consent

1. Open **API permissions → Add a permission → Microsoft Graph → Application permissions**.
2. Add the **least-privilege** permissions the agent needs. See
   [`permissions-reference.md`](permissions-reference.md) for the per-workload matrix — for example
   `Mail.Read`, `Calendars.Read`, `Files.Read.All`, `Sites.Selected`, `User.Read.All`.
3. Select **Grant admin consent for \<your tenant\>** and confirm. Application permissions are
   tenant-wide and take effect only after consent.

> [!IMPORTANT]
> Application permissions are highly privileged — they apply without a signed-in user. Scope them
> down: prefer `Sites.Selected` (with per-site grants) over `Sites.ReadWrite.All`, and consider an
> Exchange Online Application Access Policy so the app can reach only the service mailbox. Avoid broad
> scopes like `Files.Read.All` unless genuinely required (they override `Sites.Selected`).

## Step 4 — Provision the service (bot) account

The client-credentials token has no signed-in user, so `/me` does not work — every Graph call must
name a user. Use a dedicated, non-human account so agent activity stays scoped to one identity.

1. Ensure a service user exists, e.g. `bot@<your-domain>` — this is the **`<BOT_UPN>`**.
2. License it for the workloads in scope (Exchange Online for mail/calendar, OneDrive for files) so
   its mailbox and drive are provisioned. Without a provisioned mailbox/drive, Graph returns `404`.

## Step 5 — Hand off to the operator

Give the operator these four values for the [installation prompt](../prompts/enable-office365.md):

| Value | Delivered as | Delivery |
| --- | --- | --- |
| Directory (tenant) ID | `GRAPH_TENANT_ID` | Env var (not secret) |
| Application (client) ID | `GRAPH_CLIENT_ID` | Env var (not secret) |
| Client secret | `GRAPH_CLIENT_SECRET` | Secret store / env var **only** |
| Service user UPN | `<BOT_UPN>` | Plain (not secret) |

## Verification

Confirm the token and a call succeed before handing off (replace placeholders; supply the secret from
your env, never inline in a saved file):

```bash
# Export GRAPH_TENANT_ID, GRAPH_CLIENT_ID, GRAPH_CLIENT_SECRET from your secret store first.
# 1) Get a token
curl -s -X POST "https://login.microsoftonline.com/${GRAPH_TENANT_ID}/oauth2/v2.0/token" \
  -d "client_id=${GRAPH_CLIENT_ID}" \
  -d "scope=https://graph.microsoft.com/.default" \
  -d "grant_type=client_credentials" \
  --data-urlencode "client_secret=${GRAPH_CLIENT_SECRET}"

# 2) Use the returned access_token to read the service mailbox
curl -s "https://graph.microsoft.com/v1.0/users/<BOT_UPN>/messages?\$top=1" \
  -H "Authorization: Bearer <access_token>"
```

A JSON token in step 1 and a `200` with mail JSON in step 2 confirm the setup. A `401`/`403` usually
means missing consent or permissions; a `404` usually means the mailbox/drive is not provisioned.

## Rotation

Rotate before expiry: create a second client secret, switch consumers to the new
`GRAPH_CLIENT_SECRET` value, then delete the old secret.

## References

- Register an app — <https://learn.microsoft.com/en-us/graph/auth-register-app-v2>
- Get access without a user (app-only) — <https://learn.microsoft.com/en-us/graph/auth-v2-service>
- Permissions overview & admin consent — <https://learn.microsoft.com/en-us/graph/permissions-overview>
