# Prompt — enable Office 365 access on a target repo

Give the prompt below to an AI coding agent (Claude Code, Codex, Gemini, …). It makes the agent wire
Microsoft Office 365 (Microsoft Graph) access into a **target git repository** by adding an
instruction block to that repo's agent memory file and making the official Graph docs available. It
writes **no secrets** and ships **no runnable code** — it configures the contract and verifies the
environment.

## Before you run it

Complete the [administrator setup](../docs/entra-admin-setup.md) first. Have these ready:

- **Target repo path** — the local path to the repository to enable.
- `<BOT_UPN>` — the service account UPN, e.g. `bot@your-domain.com`.
- The three credentials exported in the agent's environment (from your secret store / host config —
  **not** typed into the repo or this prompt):
  - `GRAPH_TENANT_ID` — Directory (tenant) ID (not secret)
  - `GRAPH_CLIENT_ID` — Application (client) ID (not secret)
  - `GRAPH_CLIENT_SECRET` — client secret (**secret**)

## The prompt (copy, fill `BOT_UPN` + the repo path, paste to your agent)

```text
You are enabling Microsoft Office 365 (Microsoft Graph) access for an AI coding agent on a target
repository. Follow these steps exactly. Do NOT invent values and do NOT write any secret.

INPUTS (I will provide):
- TARGET_REPO_PATH: <absolute path to the target git repo>
- BOT_UPN:          <service account UPN, e.g. bot@your-domain.com>
The credentials are NOT inputs: they live only in the env vars GRAPH_TENANT_ID, GRAPH_CLIENT_ID, and
GRAPH_CLIENT_SECRET. Never read, print, or write GRAPH_CLIENT_SECRET's value.

STEPS:
1. Confirm TARGET_REPO_PATH is a git repository. If it is a PUBLIC repository, STOP and ask me to
   confirm before writing anything.
2. Choose the target agent instruction file in TARGET_REPO_PATH:
   - Claude Code -> CLAUDE.md
   - Codex       -> AGENTS.md
   - Gemini      -> GEMINI.md
   Detect which already exist. If exactly one exists, use it. If several exist, update each. If none
   exists, ask me which agent(s) this repo targets and create the matching file(s).
3. Make the official Microsoft Graph docs available in TARGET_REPO_PATH. If a submodule at
   vendors/microsoft-graph-docs is not already present, add it (shallow):
     git submodule add -b main https://github.com/microsoftgraph/microsoft-graph-docs-contrib.git vendors/microsoft-graph-docs
     git config -f .gitmodules submodule.vendors/microsoft-graph-docs.shallow true
     git submodule update --init --depth 1 vendors/microsoft-graph-docs
   If you cannot add a submodule (policy/offline), note it and tell me the agent must instead consult
   https://learn.microsoft.com/en-us/graph/ at runtime.
4. Fetch the snippet template from this setup package:
   templates/office365-access.snippet.md  (the block between its BEGIN and END markers).
5. Fill <BOT_UPN> with the input above. Leave GRAPH_TENANT_ID / GRAPH_CLIENT_ID / GRAPH_CLIENT_SECRET
   as env-var references — never substitute their values.
6. Insert the filled block into each chosen file:
   - If a block delimited by "BEGIN: office-365-access" / "END: office-365-access" already exists,
     REPLACE it in place (idempotent — never duplicate it).
   - Otherwise append it as a new top-level section.
7. Verify the environment: check whether GRAPH_TENANT_ID, GRAPH_CLIENT_ID, and GRAPH_CLIENT_SECRET
   are set (presence only — do NOT print GRAPH_CLIENT_SECRET's value). Report each as SET or MISSING;
   if any is MISSING, tell me to export it from my secret store / host config.
8. Remind me that the agent host's network policy must allow outbound HTTPS to
   login.microsoftonline.com and graph.microsoft.com.
9. Summarize: which file(s) changed, whether the Graph-docs submodule was added, that no secret was
   written, and the env-var check results. Do NOT commit secrets.

GUARDRAILS:
- Never print, log, echo, or write the client secret. The only reference to it is the env-var name.
- Be idempotent: re-running must not create a second block or a duplicate submodule.
- Make no other changes to the target repo.
```

## What the agent produces

- The target repo's `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` gains an `## Office 365 data retrieval`
  section (the filled [snippet](../templates/office365-access.snippet.md)).
- The official Graph docs available at `vendors/microsoft-graph-docs/` (when submodules are allowed).
- A report of the changed file(s), the `GRAPH_*` env-var presence checks, and the network-host
  reminder — with no secret anywhere in the repo.

## Next

Confirm the wiring works with the read-only [verification prompt](verify-office365.md).
