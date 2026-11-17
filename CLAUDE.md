# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) and other AI coding agents when working
with code in this repository.

`agent-setup-microsoft-office365` is a **setup package**: human-readable docs plus an AI-runnable
prompt that enable Microsoft Office 365 (Microsoft Graph) access for an AI coding agent on a target
git repository. It documents the integration *flow* — it ships **no runnable code and no MCP server**.

## ⚠️ This is a PUBLIC repository

Everything committed here is world-readable. **Never** write into file contents, commit messages, or
file/directory names:

- secrets or client secret values (the secret lives only in the env var
  `MCP_MS_OFFICE365_CLIENT_SECRET`);
- real tenant IDs, client IDs, object IDs, or other tenant-specific identifiers;
- real user emails / UPNs (e.g. a real `bot@…` address), personal data;
- internal hostnames, SharePoint/OneDrive URLs, or other non-public links.

Use **org-neutral placeholders only**: `<TENANT_ID>`, `<CLIENT_ID>`, `<BOT_UPN>`, and the env-var name
`MCP_MS_OFFICE365_CLIENT_SECRET`. Any tenant-specific value belongs in a consumer's private config, not
here. Before committing, scan the diff for leaked identifiers.

## Repository structure

```
.
├── README.md                          # human landing page: context, rationale, install guide
├── CLAUDE.md                          # this file
├── LICENSE                            # MIT (Devrise Technology)
├── docs/
│   ├── solution-rationale.md          # why Microsoft Graph client-credentials (options A/B/C)
│   ├── entra-admin-setup.md           # Part 1: human Entra administrator runbook
│   ├── authentication.md              # REST API contract (token + call + throttling)
│   └── permissions-reference.md       # application-permission matrix + app-only gotchas
├── prompts/
│   └── enable-office365.md            # Part 2: the AI install prompt (takes a target repo path)
└── templates/
    └── office365-access.snippet.md    # the block the install prompt injects into the target repo
```

## Conventions

- **Documentation**: write with the `sdlc:author-technical-doc` approach — one Diátaxis type per page
  (`authentication.md` / `permissions-reference.md` are *reference*; `entra-admin-setup.md` and the
  install prompt are *how-to*; `solution-rationale.md` is *explanation*). Match house style:
  repo-slug/readable H1, fenced ASCII trees with `# comments`, backticks for API names / scopes /
  env vars / endpoints, `> [!NOTE]` / `> [!WARNING]` / `> [!IMPORTANT]` admonitions.
- **Ground Graph facts in official Microsoft knowledge and cite it.** Verify claims against the
  Microsoft Graph docs mirror vendored in the superproject at `vendors/microsoft-graph-docs/`
  (upstream `microsoftgraph/microsoft-graph-docs-contrib`) — note that path is at the **superproject
  level, not inside this repo**. Cite the public `https://learn.microsoft.com/en-us/graph/…` URLs
  (and `…/entra/identity-platform/…` for the OAuth grant), since the vendor path does not ship here.
  Prefer `v1.0` over `/beta`. If a claim can't be grounded officially, mark it as this package's dated
  assessment rather than stating it as Microsoft fact.
- **Keep docs in sync**: the `README.md` section order, the `docs/` pages, the install prompt, and the
  template must agree. Update them together.
- **Commits**: Conventional Commits (`docs(auth): …`, `feat(prompt): …`, `fix(permissions): …`).
- **Ownership**: Devrise Technology (`hello@devrisetech.com`); repo under `Devrisetech/…`. Keep the
  MIT `LICENSE` as-is.

## The integration in one paragraph

An AI agent reaches Office 365 via the Microsoft Graph `v1.0` REST API using the OAuth 2.0
client-credentials (app-only) grant — no signed-in user. An Entra admin registers an app, creates a
secret, grants least-privilege **application** permissions (admin consent required), and provisions a
service account. The agent mints a token from `login.microsoftonline.com`, calls
`graph.microsoft.com/v1.0`, and must name a user explicitly (no `/me`). The secret is delivered only
via `MCP_MS_OFFICE365_CLIENT_SECRET`. See `docs/` for the grounded detail.
