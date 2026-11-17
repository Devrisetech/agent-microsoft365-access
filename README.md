# agent-setup-microsoft-office365

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A **setup package** that enables Microsoft Office 365 access for AI coding agents — Claude Code,
Codex, Gemini, and others. It gives you the human administrator runbook, a grounded explanation of
*why* this approach, and copy-paste prompts that make an AI agent wire the integration into any
target git repository and verify it. It documents the integration **flow**; it ships no runnable code
and no MCP server.

## Table of contents

- [About](#about)
- [Why Microsoft Graph (client credentials)](#why-microsoft-graph-client-credentials)
- [Requirements](#requirements)
- [Installation](#installation)
  - [Part 1 — Administrator setup (human)](#part-1--administrator-setup-human)
  - [Part 2 — AI-powered installation](#part-2--ai-powered-installation)
  - [Part 3 — Verify (read-only)](#part-3--verify-read-only)
- [Security](#security)
- [Maintainers](#maintainers)
- [License](#license)

## About

AI coding agents increasingly need to read and act on Office 365 data — mail, calendar, files in
SharePoint/OneDrive, Teams messages. An agent often runs **headless** (CI, a scheduled job, an
operator server) and may also run interactively, across local, remote, and cloud deployments — in
none of these is there a reliable human at a browser to sign in, approve MFA, or click consent. This
package standardizes a single, auditable way for an agent to reach Office 365 through the **Microsoft
Graph API** using an application identity, and automates wiring that contract into a repository your
agent works in.

The package is **org-neutral**: it uses the placeholder `<BOT_UPN>` and the credential env vars
`GRAPH_TENANT_ID`, `GRAPH_CLIENT_ID`, and `GRAPH_CLIENT_SECRET`, so any organization can adopt it.
Your tenant-specific values stay in your own private configuration — never in this repository.

## Why Microsoft Graph (client credentials)

Three approaches were considered; Microsoft Graph with the client-credentials grant was chosen because
it is the only one that is genuinely non-interactive, needs no extra licensing, is generally
available, and is centrally governed in Microsoft Entra.

| Option | What it is | Assessment |
| --- | --- | --- |
| A — Agent 365 / per-app MCP servers | Microsoft-hosted per-app MCP servers | Not chosen — positioned as legacy/back-compat |
| B — Work IQ MCP | Microsoft's newer agent MCP surface | Not chosen — consumption-billed and partly in preview |
| **C — Microsoft Graph REST API v1.0 (client credentials)** | GA REST surface over Office 365, app-only OAuth2 | **Chosen** |

Read the grounded, fully-cited rationale — including the honest app-only limitations — in
[`docs/solution-rationale.md`](docs/solution-rationale.md).

## Requirements

- A **Microsoft 365 / Entra tenant** and an administrator who can register apps and grant admin
  consent (**Global Administrator** or **Privileged Role Administrator**).
- A **licensed service account** (bot mailbox/OneDrive) for the agent to act as.
- An AI coding agent whose host can hold the `GRAPH_TENANT_ID` / `GRAPH_CLIENT_ID` /
  `GRAPH_CLIENT_SECRET` env vars and reach `login.microsoftonline.com` and `graph.microsoft.com` over
  HTTPS.

## Installation

Installation has three parts: a one-time human setup in Microsoft Entra, an AI-powered step that wires
a specific repository, and a read-only check that it works.

### Part 1 — Administrator setup (human)

A Microsoft Entra administrator registers the application, creates a client secret, grants
least-privilege Microsoft Graph **application** permissions (with admin consent), and provisions the
service account. This produces the credentials delivered as `GRAPH_TENANT_ID`, `GRAPH_CLIENT_ID`, and
`GRAPH_CLIENT_SECRET`, plus the service user UPN.

➡️ Follow the full runbook: [`docs/entra-admin-setup.md`](docs/entra-admin-setup.md). Grant scopes per
the matrix in [`docs/permissions-reference.md`](docs/permissions-reference.md).

> [!IMPORTANT]
> Deliver the client secret only through a secret manager / host secret store as
> `GRAPH_CLIENT_SECRET`. Never put it in a repository, chat, or ticket.

### Part 2 — AI-powered installation

Hand the prompt in [`prompts/enable-office365.md`](prompts/enable-office365.md) to your AI agent,
along with the **target repository path** and the service user UPN. The agent detects the repo's agent
memory file (`CLAUDE.md`, `AGENTS.md`, or `GEMINI.md`), injects the Office 365 access block from
[`templates/office365-access.snippet.md`](templates/office365-access.snippet.md), makes the official
Graph docs available as a submodule, and verifies the credential env vars are present — without ever
writing the secret.

### Part 3 — Verify (read-only)

Hand the prompt in [`prompts/verify-office365.md`](prompts/verify-office365.md) to the agent. It mints
a token and makes `GET`-only calls to confirm the credentials, consented permissions, and network
path work. It writes nothing and never reveals the secret.

## Security

- The client secret is delivered to agents **only** via `GRAPH_CLIENT_SECRET` and must never be
  printed, logged, or committed.
- Grant the **least privilege** the agent needs (prefer `Sites.Selected` over tenant-wide scopes),
  and review OneDrive/Teams scopes for blast radius — see
  [`docs/permissions-reference.md`](docs/permissions-reference.md).

## Maintainers

Devrise Technology — <hello@devrisetech.com>

## License

[MIT](LICENSE) © Devrise Technology
