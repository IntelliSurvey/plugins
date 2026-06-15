---
name: doctor
description: >-
  Diagnose and repair the isvy MCP connection. Use when the user invokes /isvy:doctor,
  or reports that the isvy server is missing from /mcp, fails to connect, won't
  authenticate, or that IntelliSurvey tools like validate_ispl are unavailable.
---

# isvy doctor — diagnose the IntelliSurvey MCP connection

Run the checks in order and STOP at the first failure — each failure explains all later
ones. Report a short ✓/✗ checklist of the checks you ran, then a single clearly-stated
next step. Translate raw output to plain English; never dump curl headers at the user.

## Check 1 — plugin installed and enabled

    claude plugin list

Expect `isvy@intellisurvey`, enabled. Missing → point at the README install one-liner.
Disabled → `claude plugin enable isvy@intellisurvey`.

## Check 2 — server name configured

Read `pluginConfigs["isvy@intellisurvey"].options.server_name` from `~/.claude/settings.json`
(user-scope install; for project/local scope check `.claude/settings.json` or
`.claude/settings.local.json` in the project).

Missing → the most common failure: the MCP entry is *silently dropped* without it.
Fix: `/plugin configure isvy@intellisurvey`, or offer to write the value into the settings
file yourself (ask the user for their hostname). Then `/reload-plugins`.

## Check 3 — server name is a plausible bare hostname

The value is substituted into `https://<server_name>/mcp` with no validation. Flag and
offer to fix if it:

- contains `@` — parses as URL credentials and shows as `https://[REDACTED]@…` in
  `claude mcp list` (a real, observed typo: `acme@intellisurvey.com`)
- contains `://` or `/` — the user entered a URL; strip it to the bare hostname
- fails `^[A-Za-z0-9.-]+(:[0-9]+)?$`

## Check 4 — MCP entry present and its status

    claude mcp list

| `plugin:isvy:isvy` line shows | Meaning | Action |
|---|---|---|
| absent | config missing or not loaded | recheck 2–3, then `/reload-plugins` |
| literal `${user_config.server_name}` | config not picked up | `/reload-plugins`, else new session |
| `✘ Failed to connect` | bad hostname or server problem | go to check 5 |
| `! Needs authentication` | healthy, not signed in | `/mcp` → isvy → **Authenticate** |
| `✔ Connected` | transport healthy | go to check 6 |

## Check 5 — server reachability and identity (only after a connect failure)

    curl -sS -m 10 -D - -o /dev/null https://<server_name>/mcp

- DNS error → hostname typo or non-public server; confirm spelling with the user
- TLS error / timeout → unreachable from this machine (VPN, firewall)
- **401 with `www-authenticate: Bearer … resource_metadata=…`** → server is healthy and
  the problem is client-side; `/reload-plugins` and re-run check 4
- 404 → host exists but isn't an IntelliSurvey MCP endpoint — wrong server name
- 502/503 → IntelliSurvey is up but the MCP backend is down — escalate to the server admin

On a 401, also verify the OAuth discovery chain (both must return JSON; the second must
include `registration_endpoint`, since Claude Code relies on dynamic client registration):

    curl -sS -m 10 https://<server_name>/.well-known/oauth-protected-resource/mcp
    curl -sS -m 10 https://<server_name>/.well-known/oauth-authorization-server

A missing endpoint is a server-side deployment problem — escalate; nothing client-side
will fix it.

## Check 6 — tools actually loaded (only when Connected)

Search this session's available tools for `isvy` (e.g. `validate_ispl`). Connected but
no tools → the session predates the connection: `/reload-plugins` or a new session.

## Check 7 — approved tools prompting for permission (only when Connected)

If isvy tools work but routine pre-approved calls (validate, build status, source)
ask for permission: the surveys workflow skill isn't active in this session. Tool
pre-approvals are carried by skill invocation and do not survive a session resume.
Fix: invoke `/isvy:surveys` (or say "use the surveys skill"), then retry.

## Repair rules

- Settings fixes (write/correct `server_name`): offer, apply on confirmation, then tell
  the user to run `/reload-plugins`.
- Never touch tokens or the keychain. Authentication is always `/mcp` → Authenticate.
