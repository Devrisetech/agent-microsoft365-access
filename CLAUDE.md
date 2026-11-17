# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) and other AI coding agents when working
with code in this repository.

`agent-setup-microsoft-office365` is a **setup package**: human-readable docs plus AI-runnable
prompts that enable Microsoft Office 365 (Microsoft Graph) access for an AI coding agent on a target
git repository. It documents the integration *flow* — it ships **no runnable code and no MCP server**.

## ⚠️ This is a PUBLIC repository

Everything committed here is world-readable. **Never** write into file contents, commit messages, or
file/directory names:

- secrets or client secret values (the secret lives only in the env var `GRAPH_CLIENT_SECRET`);
- real tenant IDs, client IDs, object IDs, or other tenant-specific identifiers;
- real user emails / UPNs (e.g. a real `bot@…` address), personal data;
- internal hostnames, SharePoint/OneDrive URLs, or other non-public links.

Use **org-neutral placeholders only**: the service-account UPN as `<BOT_UPN>`, and the credential
env-var names `GRAPH_TENANT_ID`, `GRAPH_CLIENT_ID`, `GRAPH_CLIENT_SECRET`. Any tenant-specific value
belongs in a consumer's private config, not here. Before committing, scan the diff for leaked
identifiers.

## Repository structure

```
.
├── README.md                          # human landing page: context, rationale, install guide
├── CLAUDE.md                          # this file
├── LICENSE                            # MIT (Devrise Technology)
├── .gitignore                         # OS/editor temp files + .claude/plans/
├── .claude/
│   └── settings.json                  # Claude Code settings (model, plugins, marketplaces)
├── .github/
│   └── CODEOWNERS                     # review protection for sensitive paths
├── docs/
│   ├── solution-rationale.md          # why Microsoft Graph client-credentials (options A/B/C)
│   ├── entra-admin-setup.md           # Part 1: human Entra administrator runbook
│   ├── authentication.md              # REST API contract (token + call + throttling)
│   └── permissions-reference.md       # application-permission matrix + app-only gotchas
├── prompts/
│   ├── enable-office365.md            # Part 2: the AI install prompt (takes a target repo path)
│   └── verify-office365.md            # Part 3: read-only verification prompt
└── templates/
    └── office365-access.snippet.md    # the block the install prompt injects into the target repo
```

## Conventions

- **Documentation**: write each page as a single Diátaxis type — `authentication.md` and
  `permissions-reference.md` are *reference* (austere, declarative); `entra-admin-setup.md` and the
  prompts are *how-to* (second person, imperative); `solution-rationale.md` is *explanation*
  (reasoning and trade-offs). Match house style: readable H1, fenced ASCII trees with `# comments`,
  backticks for API names / scopes / env vars / endpoints, `> [!NOTE]` / `> [!WARNING]` /
  `> [!IMPORTANT]` admonitions.
- **Ground Graph facts in official Microsoft knowledge and cite it.** Verify claims against the
  Microsoft Graph docs (upstream `microsoftgraph/microsoft-graph-docs-contrib`, often vendored as a
  submodule at `vendors/microsoft-graph-docs/`). Cite the public
  `https://learn.microsoft.com/en-us/graph/…` URLs (and `…/entra/identity-platform/…` for the OAuth
  grant). Prefer `v1.0` over `/beta`. If a claim can't be grounded officially, mark it as this
  package's dated assessment rather than stating it as Microsoft fact.
- **Keep docs in sync**: the `README.md` section order, the `docs/` pages, the prompts, and the
  template must agree. Update them together.
- **Never mention internal tooling/skill names** in committed content, commit messages, or PR text.
  This is a public repo and such names are internal implementation detail — describe the approach,
  not the tool that produced it.
- **Commits**: Conventional Commits (`docs(auth): …`, `feat(prompt): …`, `fix(permissions): …`).
- **Ownership**: Devrise Technology (`hello@devrisetech.com`); repo under `Devrisetech/…`. Keep the
  MIT `LICENSE` as-is. Sensitive paths are protected by `.github/CODEOWNERS`.

## The integration in one paragraph

An AI agent reaches Office 365 via the Microsoft Graph `v1.0` REST API using the OAuth 2.0
client-credentials (app-only) grant — no signed-in user. An Entra admin registers an app, creates a
secret, grants least-privilege **application** permissions (admin consent required), and provisions a
service account. The agent mints a token from `login.microsoftonline.com`, calls
`graph.microsoft.com/v1.0`, and must name a user explicitly (no `/me`). The credentials are delivered
via `GRAPH_TENANT_ID` / `GRAPH_CLIENT_ID` / `GRAPH_CLIENT_SECRET`, and the secret is never written to
a file. See `docs/` for the grounded detail.
