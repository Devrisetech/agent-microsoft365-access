# agent-setup-microsoft-office365

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A **setup package** that enables Microsoft Office 365 access for AI coding agents — Claude Code,
Codex, Gemini, and others. It gives you the human administrator runbook, a grounded explanation of
*why* this approach, and a copy-paste prompt that makes an AI agent wire the integration into any
target git repository. It documents the integration **flow**; it ships no runnable code and no MCP
server.

## Table of contents

- [About](#about)
- [Why Microsoft Graph (client credentials)](#why-microsoft-graph-client-credentials)
- [Requirements](#requirements)
- [Installation](#installation)
  - [Part 1 — Administrator setup (human)](#part-1--administrator-setup-human)
  - [Part 2 — AI-powered installation](#part-2--ai-powered-installation)
- [How it works](#how-it-works)
- [Repository structure](#repository-structure)
- [Security](#security)
- [Maintainers](#maintainers)
- [License](#license)

## About

AI coding agents increasingly need to read and act on Office 365 data — mail, calendar, files in
SharePoint/OneDrive, Teams messages. An agent runs **headless**: there is no human at a browser to
sign in, approve MFA, or click consent. This package standardizes a single, auditable way for an agent
to reach Office 365 through the **Microsoft Graph API** using an application identity, and automates
wiring that contract into a repository your agent works in.

The package is **org-neutral**: it uses placeholders (`<TENANT_ID>`, `<CLIENT_ID>`, `<BOT_UPN>`, and
the env var `MCP_MS_OFFICE365_CLIENT_SECRET`) so any organization can adopt it. Your tenant-specific
values stay in your own private configuration — never in this repository.

## Why Microsoft Graph (client credentials)

Three approaches were considered; Microsoft Graph with the client-credentials grant was chosen because
it is the only one that is genuinely headless, needs no extra licensing, is generally available, and
is centrally governed in Microsoft Entra.

| Option | What it is | Verdict |
| --- | --- | --- |
| A — Agent 365 / per-app MCP servers | Microsoft-hosted per-app MCP servers | Rejected — positioned as legacy/back-compat |
| B — Work IQ MCP | Microsoft's newer agent MCP surface | Rejected — consumption-billed and partly in preview |
| **C — Microsoft Graph REST API v1.0 (client credentials)** | GA REST surface over O365, app-only OAuth2 | **Chosen** |

Read the grounded, fully-cited rationale — including the honest app-only limitations — in
[`docs/solution-rationale.md`](docs/solution-rationale.md).

## Requirements

- A **Microsoft 365 / Entra tenant** and an administrator who can register apps and grant admin
  consent (**Global Administrator** or **Privileged Role Administrator**).
- A **licensed service account** (bot mailbox/OneDrive) for the agent to act as.
- An AI coding agent whose host can hold the env var `MCP_MS_OFFICE365_CLIENT_SECRET` and reach
  `login.microsoftonline.com` and `graph.microsoft.com` over HTTPS.

## Installation

Installation has two parts: a one-time human setup in Microsoft Entra, then an AI-powered step that
wires a specific repository.

### Part 1 — Administrator setup (human)

A Microsoft Entra administrator registers the application, creates a client secret, grants
least-privilege Microsoft Graph **application** permissions (with admin consent), and provisions the
service account. This produces four values — tenant ID, client ID, client secret, and service user
UPN.

➡️ Follow the full runbook: [`docs/entra-admin-setup.md`](docs/entra-admin-setup.md).

> [!IMPORTANT]
> Deliver the client secret only through a secret manager / host secret store as
> `MCP_MS_OFFICE365_CLIENT_SECRET`. Never put it in a repository, chat, or ticket.

### Part 2 — AI-powered installation

Hand the prompt in [`prompts/enable-office365.md`](prompts/enable-office365.md) to your AI agent,
along with the **target repository path** and the three non-secret values. The agent detects the
repo's agent memory file (`CLAUDE.md`, `AGENTS.md`, or `GEMINI.md`), injects the Office 365 access
block from [`templates/office365-access.snippet.md`](templates/office365-access.snippet.md), and
verifies the secret env var is present — without ever writing the secret.

## How it works

An agent authenticates with the OAuth 2.0 client-credentials (app-only) grant — no signed-in user —
and calls Microsoft Graph `v1.0`:

1. **Token** — `POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token` with
   `grant_type=client_credentials` and `scope=https://graph.microsoft.com/.default`.
2. **Call** — present `Authorization: Bearer <token>` against `https://graph.microsoft.com/v1.0`,
   naming a user explicitly (there is no `/me` in app-only), e.g. `/users/<BOT_UPN>/messages`.

The exact REST contract is in [`docs/authentication.md`](docs/authentication.md); the per-workload
permission matrix and app-only gotchas are in
[`docs/permissions-reference.md`](docs/permissions-reference.md).

## Repository structure

```
.
├── README.md                          # this file
├── CLAUDE.md                          # guidance for AI agents working ON this package
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

## Security

- This is a **public repository** — treat everything in it as world-readable. It contains only
  placeholders, never real tenant/client IDs, service emails, internal URLs, or secrets.
- The client secret is delivered to agents **only** via `MCP_MS_OFFICE365_CLIENT_SECRET` and must
  never be printed, logged, or committed.
- Grant the **least privilege** the agent needs (prefer `Sites.Selected` over tenant-wide scopes),
  and review OneDrive/Teams scopes for blast radius — see
  [`docs/permissions-reference.md`](docs/permissions-reference.md).

## Maintainers

Devrise Technology — <hello@devrisetech.com>

## License

[MIT](LICENSE) © Devrise Technology
