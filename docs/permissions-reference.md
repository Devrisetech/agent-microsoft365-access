# Permissions reference — application (app-only) permissions

This page lists the Microsoft Graph **application permissions** an administrator grants so an AI agent
can reach each Office 365 workload, and the limits and traps that apply to app-only access. It is a
reference: grant the **least privilege** the agent actually needs — nothing more.

> [!IMPORTANT]
> All application permissions require **admin consent**. App-only access has no per-user consent
> step; only a **Privileged Role Administrator** or **Global Administrator** can consent.
> Grounded in <https://learn.microsoft.com/en-us/graph/permissions-overview>.

## Permission matrix by workload

| Workload | Recommended (least privilege) | Broader alternative | Notes |
| --- | --- | --- | --- |
| SharePoint sites | `Sites.Selected` + per-site grant | `Sites.ReadWrite.All` | `Sites.Selected` scopes access to named sites/lists only |
| SharePoint / OneDrive files | `Files.ReadWrite.All` | — | Tenant-wide (see OneDrive caveat below) |
| OneDrive (personal drives) | `Files.Read.All` | `Files.ReadWrite.All` | Tenant-wide only — no `Sites.Selected` equivalent |
| Teams — read | `ChannelMessage.Read.All`, `Team.ReadBasic.All`, `Channel.ReadBasic.All` | `Chat.Read.All` | Read-only is the realistic app-only ceiling |
| Teams — manage | `Channel.ReadWrite.All`, `TeamMember.ReadWrite.All` | — | Management, not message sending |
| Mail | `Mail.Read` | `Mail.ReadWrite` | Address as `/users/<BOT_UPN>/messages` |
| Calendar | `Calendars.Read` | `Calendars.ReadWrite` | Address as `/users/<BOT_UPN>/calendar` |
| User discovery | `User.Read.All` | — | Resolve users/UPNs |

The authoritative, exhaustive list of permissions is the Microsoft Graph permissions reference:
<https://learn.microsoft.com/en-us/graph/permissions-reference>.

## `Sites.Selected` — granular SharePoint/OneDrive access

`Sites.Selected` grants **no** site access on its own. After admin consent, an administrator (or an
app holding `Sites.FullControl.All`) grants the app access to each specific site:

```http
POST https://graph.microsoft.com/v1.0/sites/{siteId}/permissions
Content-Type: application/json

{
  "roles": ["write"],
  "grantedToIdentities": [ { "application": { "id": "<CLIENT_ID>" } } ]
}
```

Lists, list items, folders, and files can be scoped the same way. Grounded in
<https://learn.microsoft.com/en-us/graph/permissions-selected-overview>.

## Gotchas and limits (app-only)

> [!WARNING]
> A broad scope such as `Files.Read.All` or `Sites.Read.All` **silently overrides** `Sites.Selected`
> and re-opens tenant-wide access. Keep the app lean — do not add unused scopes.

- **OneDrive blast radius.** There is no `Sites.Selected` equivalent for personal OneDrive.
  `Files.ReadWrite.All` grants access to **every user's** OneDrive tenant-wide. Expect a security
  review and gate it operationally. Address a personal drive as `/users/{id|upn}/drive` — there is no
  `/me` in app-only context.
- **Teams message sending is effectively blocked app-only.** `POST .../messages` only accepts
  `Teamwork.Migrate.All` (migration mode). Live posting needs delegated auth or a Teams bot identity.
- **Tenant-wide chat listing is unsupported app-only.** `GET /chats` is not available; enumerate per
  user via `/users/{id}/chats`.
- **Protected / metered Teams APIs.** High-volume read APIs (`channel/getAllMessages`,
  `chats/getAllMessages`, change-notification subscriptions) are licensed/metered and carry real cost.
- **Upload size.** A simple file upload is capped at 250 MB; larger files need an upload session.
- **Throttling.** Honor `429` + `Retry-After` — see [`authentication.md`](authentication.md).

## References

- Permissions overview (app-only, admin consent) — <https://learn.microsoft.com/en-us/graph/permissions-overview>
- Permissions reference (full list) — <https://learn.microsoft.com/en-us/graph/permissions-reference>
- Selected permissions (`Sites.Selected`) — <https://learn.microsoft.com/en-us/graph/permissions-selected-overview>
