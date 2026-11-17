# Prompt — enable Office 365 access on a target repo

Give the prompt below to an AI coding agent (Claude Code, Codex, Gemini, …). It makes the agent wire
Microsoft Office 365 (Microsoft Graph) access into a **target git repository** by adding an
instruction block to that repo's agent memory file. It writes **no secrets** and ships **no runnable
code** — it configures the contract and verifies the environment.

## Before you run it

Complete the [administrator setup](../docs/entra-admin-setup.md) first. Have these ready:

- **Target repo path** — the local path to the repository to enable.
- `<TENANT_ID>` and `<CLIENT_ID>` — non-secret, from the app registration.
- `<BOT_UPN>` — the service account UPN, e.g. `bot@your-domain.com`.
- The client secret available in the agent's environment as `MCP_MS_OFFICE365_CLIENT_SECRET`
  (exported from your secret store — **not** typed into the repo or this prompt).

## The prompt (copy, fill the four values, paste to your agent)

```text
You are enabling Microsoft Office 365 (Microsoft Graph) access for an AI coding agent on a target
repository. Follow these steps exactly. Do NOT invent values and do NOT write any secret.

INPUTS (I will provide):
- TARGET_REPO_PATH: <absolute path to the target git repo>
- TENANT_ID:        <Directory/tenant ID>   (not secret)
- CLIENT_ID:        <Application/client ID>  (not secret)
- BOT_UPN:          <service account UPN, e.g. bot@your-domain.com>
The client secret is NOT an input: it lives only in the env var MCP_MS_OFFICE365_CLIENT_SECRET.

STEPS:
1. Confirm TARGET_REPO_PATH is a git repository. If it is a PUBLIC repository, STOP and ask me to
   confirm before writing anything (non-secret IDs still end up in its history).
2. Choose the target agent instruction file in TARGET_REPO_PATH:
   - Claude Code -> CLAUDE.md
   - Codex       -> AGENTS.md
   - Gemini      -> GEMINI.md
   Detect which already exist. If exactly one exists, use it. If several exist, update each. If none
   exists, ask me which agent(s) this repo targets and create the matching file(s).
3. Fetch the snippet template from this setup package:
   templates/office365-access.snippet.md  (the block between its BEGIN and END markers).
4. Fill the placeholders in the snippet: <TENANT_ID>, <CLIENT_ID>, <BOT_UPN> with the inputs above.
   Leave MCP_MS_OFFICE365_CLIENT_SECRET exactly as written — never substitute the secret value.
5. Insert the filled block into each chosen file:
   - If a block delimited by "BEGIN: office-365-access" / "END: office-365-access" already exists,
     REPLACE it in place (idempotent — never duplicate it).
   - Otherwise append it as a new top-level section.
6. Verify the environment: check whether MCP_MS_OFFICE365_CLIENT_SECRET is set (presence only — do
   NOT print its value). Report SET or MISSING; if MISSING, tell me to export it from my secret store.
7. Remind me that the agent host's network policy must allow outbound HTTPS to
   login.microsoftonline.com and graph.microsoft.com.
8. Summarize: which file(s) changed, that the secret was never written, and the env-var check result.
   Do NOT commit secrets. Do NOT paste the secret anywhere.

GUARDRAILS:
- Never print, log, echo, or write the client secret. The only reference to it is the env-var name.
- Be idempotent: re-running must not create a second block.
- Make no other changes to the target repo.
```

## What the agent produces

- The target repo's `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` gains an `## Office 365 data retrieval`
  section (the filled [snippet](../templates/office365-access.snippet.md)).
- A report of the changed file(s), the `MCP_MS_OFFICE365_CLIENT_SECRET` presence check, and the
  network-host reminder.
- No secret anywhere in the repo.

## Verifying the result

After the agent runs, the target repo's agent can authenticate using the
[REST contract](../docs/authentication.md). A quick manual check:

```bash
# from an environment where MCP_MS_OFFICE365_CLIENT_SECRET is exported
curl -s -X POST "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" \
  -d "client_id=<CLIENT_ID>" -d "scope=https://graph.microsoft.com/.default" \
  -d "grant_type=client_credentials" \
  --data-urlencode "client_secret=${MCP_MS_OFFICE365_CLIENT_SECRET}" | grep -o '"access_token"'
```

A match on `"access_token"` confirms the credentials and permissions are usable.
