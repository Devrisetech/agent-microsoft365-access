<!--
  Office 365 access snippet — injected into a target repo's agent instruction file
  (CLAUDE.md / AGENTS.md / GEMINI.md) by prompts/enable-office365.md.

  HOW TO USE (for the installing agent):
  - Copy everything between the BEGIN and END markers below into the target file.
  - Replace <BOT_UPN> with the operator-supplied service-account UPN.
  - The three credentials come from environment variables (GRAPH_TENANT_ID, GRAPH_CLIENT_ID,
    GRAPH_CLIENT_SECRET). NEVER write their values into the file — reference the env vars only.
  - Keep the BEGIN/END markers so the block can be found and updated idempotently.
-->

<!-- BEGIN: office-365-access (managed by agent-setup-microsoft-office365) -->
## Office 365 data retrieval

Retrieve Office 365 data through the **Microsoft Graph API** (`v1.0`), authenticating with the
OAuth 2.0 **client-credentials (app-only)** flow. The same contract serves both headless and
interactive agents (local, remote, or cloud).

- **Credentials** (from the environment — never hard-code or print them):
  - `GRAPH_TENANT_ID` — Directory (tenant) ID
  - `GRAPH_CLIENT_ID` — Application (client) ID
  - `GRAPH_CLIENT_SECRET` — client secret. **Sensitive: never print, log, echo, write to a file,
    or disclose it.**
- **Service account**: address every call to the user `<BOT_UPN>` — app-only has no `/me`.
- **Get a token** (cache and reuse until it nears expiry; do not mint one per call):
  ```
  POST https://login.microsoftonline.com/${GRAPH_TENANT_ID}/oauth2/v2.0/token
  grant_type=client_credentials
  client_id=${GRAPH_CLIENT_ID}
  client_secret=${GRAPH_CLIENT_SECRET}
  scope=https://graph.microsoft.com/.default
  ```
  You choose the HTTP client / SDK. Present the result as `Authorization: Bearer <access_token>`
  against base `https://graph.microsoft.com/v1.0`, e.g. `/users/<BOT_UPN>/messages`,
  `/users/<BOT_UPN>/calendar`, `/users/<BOT_UPN>/drive`.
- **Throttling**: on `429`, wait the `Retry-After` seconds and retry; back off exponentially if the
  header is absent.
- **Network**: allow outbound HTTPS to `login.microsoftonline.com` and `graph.microsoft.com`.
- **Permissions**: calls work only for Microsoft Graph application permissions an admin has consented
  to for this app.

### Ground every call in the official Graph docs

Do not call Graph from memory. This repo vendors the official Microsoft Graph documentation as a git
submodule at `vendors/microsoft-graph-docs/` (upstream
`git@github.com:microsoftgraph/microsoft-graph-docs-contrib.git`).

Workflow (paths relative to the submodule root):

1. Read `AGENTS.md` first — the upstream agent instructions.
2. Find the API area in `api-reference/v1.0/toc.yml`.
3. Open the operation page `api-reference/v1.0/api/<resource>-<verb>.md` for the HTTP method/path,
   the **Permissions** table (use **Application** permissions for app-only), parameters, and
   request/response examples.
4. Confirm property names and JSON shape in `api-reference/v1.0/resources/<type>.md`.
5. Check `concepts/*.md` for cross-cutting behavior (paging `@odata.nextLink`, throttling, advanced
   queries `ConsistencyLevel`, etc.).
6. Prefer `v1.0`. Use `api-reference/beta/` only when an API is absent from v1.0, and treat it as
   preview/unstable with no production dependence.

If the docs and the live API disagree, trust the live response and report it rather than guessing.
<!-- END: office-365-access (managed by agent-setup-microsoft-office365) -->
