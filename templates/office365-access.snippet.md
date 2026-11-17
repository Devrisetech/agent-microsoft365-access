<!--
  Office 365 access snippet — injected into a target repo's agent instruction file
  (CLAUDE.md / AGENTS.md / GEMINI.md) by prompts/enable-office365.md.

  HOW TO USE (for the installing agent):
  - Copy everything between the BEGIN and END markers below into the target file.
  - Replace <TENANT_ID>, <CLIENT_ID>, and <BOT_UPN> with the operator-supplied values.
  - NEVER write the client secret value. Leave the env-var reference exactly as-is.
  - Keep the BEGIN/END markers so the block can be found and updated idempotently.
-->

<!-- BEGIN: office-365-access (managed by agent-setup-microsoft-office365) -->
## Office 365 data retrieval

Retrieve Office 365 data through the **Microsoft Graph API** (`v1.0`), authenticating with the
OAuth 2.0 **client-credentials (app-only)** flow. There is no signed-in user.

- **Credentials**
  - Directory (tenant) ID: `<TENANT_ID>`
  - Application (client) ID: `<CLIENT_ID>`
  - Client secret: read from env var `MCP_MS_OFFICE365_CLIENT_SECRET`.
    **Sensitive — never print, log, echo, write to a file, or disclose it.**
- **Get a token** (cache and reuse until it nears expiry; do not mint one per call):
  ```
  POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
  grant_type=client_credentials
  client_id=<CLIENT_ID>
  client_secret=$MCP_MS_OFFICE365_CLIENT_SECRET
  scope=https://graph.microsoft.com/.default
  ```
  You choose the HTTP client / SDK. Present the result as `Authorization: Bearer <access_token>`
  against base `https://graph.microsoft.com/v1.0`.
- **Address every call to a user** — app-only has no `/me`. Use the service account UPN
  `<BOT_UPN>` as the `user-id`, e.g. `/users/<BOT_UPN>/messages`, `/users/<BOT_UPN>/calendar`,
  `/users/<BOT_UPN>/drive`.
- **Throttling**: on `429`, wait the `Retry-After` seconds and retry; back off exponentially if the
  header is absent.
- **Network**: allow outbound HTTPS to `login.microsoftonline.com` and `graph.microsoft.com`.
- **Permissions**: calls work only for Microsoft Graph application permissions an admin has consented
  to for this app. Ground API shapes and permission names in the official Microsoft Graph docs
  (<https://learn.microsoft.com/en-us/graph/>); prefer `v1.0` over `/beta`.
<!-- END: office-365-access (managed by agent-setup-microsoft-office365) -->
