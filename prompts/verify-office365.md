# Prompt — verify Office 365 access (read-only)

Give this prompt to the target agent after running the [installation](enable-office365.md). It
confirms the credentials, permissions, and network path work. It is **strictly read-only**: it mints
a token and makes `GET` calls only — it creates, updates, and deletes nothing, and never reveals the
secret.

## Before you run it

- The three env vars are set in the agent's environment: `GRAPH_TENANT_ID`, `GRAPH_CLIENT_ID`,
  `GRAPH_CLIENT_SECRET`.
- You know the service account UPN `<BOT_UPN>`.

## The prompt (copy, fill `BOT_UPN`, paste to your agent)

```text
Verify Microsoft Office 365 (Microsoft Graph) access for this repo. READ-ONLY: perform GET requests
only. Do not create, update, or delete anything. Never print GRAPH_CLIENT_SECRET's value.

INPUT:
- BOT_UPN: <service account UPN>

STEPS:
1. Confirm GRAPH_TENANT_ID, GRAPH_CLIENT_ID, GRAPH_CLIENT_SECRET are all set (presence only; do not
   print the secret). If any is missing, STOP and report which.
2. Acquire an app-only token:
   POST https://login.microsoftonline.com/${GRAPH_TENANT_ID}/oauth2/v2.0/token
   with grant_type=client_credentials, client_id=${GRAPH_CLIENT_ID},
   client_secret=${GRAPH_CLIENT_SECRET}, scope=https://graph.microsoft.com/.default
   Report only whether a token was returned (yes/no) and its expires_in — never the token itself.
3. Make a minimal read-only call that reflects the app's granted permissions, e.g.:
   GET https://graph.microsoft.com/v1.0/users/<BOT_UPN>
   (or, if Mail.Read is granted: GET https://graph.microsoft.com/v1.0/users/<BOT_UPN>/messages?$top=1)
   with header Authorization: Bearer <access_token>.
4. Report the outcome:
   - 200 -> access confirmed; name the endpoint that worked.
   - 401/403 -> token or permissions problem (likely missing admin consent or the needed scope).
   - 404 -> the service mailbox/drive for BOT_UPN is not provisioned.
   - 429 -> throttled; honor Retry-After and note it.
5. Do NOT write any file, do NOT commit, and do NOT print secrets or full tokens.

Output a short pass/fail summary with the HTTP status of each call.
```

## Expected result

A token is issued and at least one `GET` returns `200`, confirming the credentials, admin-consented
permissions, and network path are all in place. Any non-`200` is reported with the status and the
likely cause (see the mapping above), so the administrator can fix consent, permissions, licensing,
or the service account.
