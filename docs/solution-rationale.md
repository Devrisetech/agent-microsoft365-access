# Solution rationale: Microsoft Graph for headless agent access

This document records *why* this package gives a headless AI coding agent (Claude Code, Codex, Gemini, and similar) access to Microsoft Office 365 data through the **Microsoft Graph REST API v1.0** with an **OAuth 2.0 client-credentials (app-only) grant**, rather than through Microsoft's agent-oriented MCP surfaces. It states the options considered, the decision drivers, and the honest limitations of the chosen approach.

> [!NOTE]
> Audience: an engineer or architect evaluating this package. Configuration values appear only as placeholders — `<TENANT_ID>`, `<CLIENT_ID>`, `<BOT_UPN>`, and the client secret read from the environment variable `MCP_MS_OFFICE365_CLIENT_SECRET`. Substitute your own tenant's values.

## The problem

A headless agent runs in CI, a scheduled job, or an operator server. It has:

- **no interactive user** at the keyboard to complete a sign-in prompt,
- **no browser** to render a consent or login page,
- **no second factor** (MFA) device in the loop, and
- a need for a **stable, documented, auditable** contract that will not shift under it.

The question is which Microsoft access mechanism satisfies all of these at once.

## Options considered

| Option | What it is | Verdict & why |
| --- | --- | --- |
| **A — Microsoft-hosted per-app MCP servers** (Copilot, Calendar, Mail, SharePoint, OneDrive, Teams, User, Word) | Workload-specific Microsoft-hosted MCP servers exposed through the Agent 365 tooling surface. | **Rejected.** Microsoft states these application-specific MCP tools "are only intended for backward compatibility purposes and shouldn't be used in production environments or new projects" ([Agent 365 — MCP server options](https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview)). A legacy/back-compat surface is the wrong foundation for a reusable package. |
| **B — Work IQ MCP** (Microsoft's forward path for agent MCP access) | A single MCP endpoint collapsing Microsoft 365 operations into ~10 generic verbs (`fetch`, `create_entity`, `update_entity`, `ask`, `retrieve`, …) over mail, calendar, files, people, chat, and sites ([Work IQ MCP overview](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/mcp/overview)). | **Rejected for this package — see nuance below.** It is Microsoft's recommended path "for new projects that require access to Microsoft 365 resources" ([MCP server options](https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview)), but it is tied to Work IQ **usage-based billing** (Copilot Credits / pay-as-you-go), parts of the surface are **in preview**, and the practical licensing picture for non-Copilot identities is unsettled. |
| **C — Microsoft Graph REST API v1.0 + OAuth 2.0 client-credentials (app-only)** | Direct HTTPS calls to `https://graph.microsoft.com/v1.0/...` with an app-only bearer token minted from the tenant's token endpoint. | **Chosen.** Satisfies all four decision drivers simultaneously (below): non-interactive auth, zero extra licensing, GA maturity, and central Entra governance. |

### Decision drivers (why C)

1. **Non-interactive authentication.** The client-credentials flow acquires a token with no user, no browser, and no MFA. A `POST` to `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token` with `grant_type=client_credentials`, `scope=https://graph.microsoft.com/.default`, and the app's credential returns a `Bearer` token with an `expires_in` lifetime (3599 s in Microsoft's example) ([Get access without a user](https://learn.microsoft.com/en-us/graph/auth-v2-service); [client-credentials grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant)). This is exactly the headless shape the problem demands.
2. **Zero extra licensing.** App-only Graph access is governed by **application permissions** granted to the app registration in Microsoft Entra — "your app can call Microsoft Graph with its own identity, without a signed-in user" — and carries no Microsoft 365 Copilot or per-seat dependency ([Permissions overview](https://learn.microsoft.com/en-us/graph/permissions-overview)). Options A and B are both bound to the Copilot/Work IQ billing-and-licensing surface.
3. **GA maturity.** Graph v1.0 is the generally available, fully documented contract, with a complete [application-permission reference](https://learn.microsoft.com/en-us/graph/permissions-reference). The chosen verbs and resource paths are stable, versus preview tooling whose tool names and scopes can change.
4. **Central governance.** A tenant administrator grants or revokes the app's permissions in Entra, and consent is itself privileged: "Only Privileged Role Administrator and Global Administrator can consent to application permissions" ([Permissions overview](https://learn.microsoft.com/en-us/graph/permissions-overview)). Access is auditable and reversible from one place.

> [!NOTE]
> Microsoft's own guidance is that **delegated** permissions are preferred where they meet the need, on least-privilege grounds ([Permissions overview](https://learn.microsoft.com/en-us/graph/permissions-overview)). App-only is chosen here *because* the agent is headless — there is no user to delegate from. That trade-off (broader-by-default app permissions in exchange for non-interactive operation) is made deliberately and is the reason the scoping discipline below matters.

## App-only reality: honest limitations of Option C

App-only access is not a superset of what a signed-in user can do. These constraints are inherent to the chosen option and should be designed around, not discovered later.

> [!WARNING]
> **Teams — sending normal messages is effectively blocked app-only.** `POST /teams/{id}/channels/{id}/messages` (and the chat-message equivalents) only accept app-only calls in Teams **migration** mode (`Teamwork.Migrate.All`), not for ordinary live posting. Live message posting needs delegated auth or a Teams bot identity. *(Behavior of the Teams messaging endpoints; verify against the current [application-permission reference](https://learn.microsoft.com/en-us/graph/permissions-reference) and the relevant `chatMessage`/`channel` operation pages before relying on it.)*

- **Teams — no tenant-wide chat listing app-only.** A tenant-wide `GET /chats` is not available app-only; chats must be enumerated **per user** via `GET /users/{id}/chats`. *(Shape of the Teams chat API; confirm against the current [permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference).)*
- **OneDrive — no `Sites.Selected` equivalent.** SharePoint supports granular, per-site least-privilege via `Sites.Selected` ([Sites.Selected overview](https://learn.microsoft.com/en-us/graph/permissions-selected-overview)). OneDrive has no equivalent granular app permission, so `Files.ReadWrite.All` is **tenant-wide** — a large blast radius.

### Realistic app-only scope for this package

| Workload | Practical app-only capability | How to scope |
| --- | --- | --- |
| SharePoint | Read / write | Least-privilege per site via `Sites.Selected` ([docs](https://learn.microsoft.com/en-us/graph/permissions-selected-overview)) |
| OneDrive | Read / write | **Tenant-wide only** (`Files.ReadWrite.All`); no site-scoped grant |
| Teams | Read-only | Per-user enumeration; no live posting app-only |

## Operational constraint: throttling

Graph enforces service limits. On a `429 Too Many Requests` the client **must** honor the `Retry-After` header and back off ([Throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)). There is also a global ceiling of **130,000 requests per 10 seconds per app across all tenants** ([Throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits)). Any agent built on this package should implement `Retry-After`-aware retry rather than tight-loop retrying.

## Status of the rejected agent surfaces (verified 2026-11-17)

- **Option A (per-app MCP servers)** is confirmed by Microsoft as a legacy/backward-compatibility surface not for new projects ([MCP server options](https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview)). This matches the internal research proposal (2026-11-06).
- **Option B (Work IQ MCP)** — the licensing claim needs care. The internal proposal (2026-11-06) described it as requiring Microsoft 365 Copilot per-seat licensing; that is **not independently confirmed as a flat per-seat requirement**. Microsoft's current Work IQ overview states usage is "independent of Microsoft 365 Copilot licensing and available through usage-based billing," with cost managed via Copilot Credits / pay-as-you-go in the Microsoft 365 admin center ([Work IQ overview](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq)). The **Work IQ in Copilot Studio** integration is explicitly in **preview** and uses the Copilot harness and Copilot Credits ([Work IQ in Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/agents-experience/add-work-iq)).

> [!NOTE]
> Regardless of the exact licensing label, Option B still fails this package's drivers: it requires enabling **usage-based (consumption) billing** and a Work IQ/Agent 365 posture, and parts of the surface are in preview — whereas Option C is GA, carries no consumption-billing dependency, and is governed purely by Entra app permissions. The usage-based-billing coupling and partial-preview status are the load-bearing reasons to prefer C for a reusable, zero-extra-cost package; the precise seat-licensing question is a secondary, still-evolving detail. *(This paragraph is this package's assessment, dated 2026-11-17, not a statement of Microsoft fact.)*

## Summary

For a headless agent, **Microsoft Graph v1.0 with app-only client-credentials** is the only one of the three options that is simultaneously non-interactive, free of extra licensing or consumption billing, generally available, and centrally governable in Entra. Its limitations (Teams posting/listing, OneDrive blast radius) are real and are designed around by scoping SharePoint via `Sites.Selected`, treating OneDrive access as tenant-wide, and keeping Teams read-only.

## References

- Get access without a user (client-credentials / app-only): https://learn.microsoft.com/en-us/graph/auth-v2-service
- OAuth 2.0 client-credentials grant (Entra): https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant
- Microsoft Graph permissions overview: https://learn.microsoft.com/en-us/graph/permissions-overview
- `Sites.Selected` granular permissions: https://learn.microsoft.com/en-us/graph/permissions-selected-overview
- Microsoft Graph application-permission reference: https://learn.microsoft.com/en-us/graph/permissions-reference
- Throttling guidance (`429` / `Retry-After`): https://learn.microsoft.com/en-us/graph/throttling
- Throttling limits (global 130,000 req / 10 s per app): https://learn.microsoft.com/en-us/graph/throttling-limits
- Agent 365 — MCP server options (Option A legacy status; Work IQ recommended): https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview
- Work IQ overview (licensing & usage-based billing): https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq
- Work IQ MCP overview (tool surface; `retrieve` in preview): https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/mcp/overview
- Work IQ in Copilot Studio (preview; Copilot harness & Credits): https://learn.microsoft.com/en-us/microsoft-copilot-studio/agents-experience/add-work-iq
