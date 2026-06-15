# IntelliSurvey plugins for Claude Code

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
containing the **isvy** plugin: survey authoring for IntelliSurvey. Two workflows:

- **Build from a document** — hand over a questionnaire (Word or Markdown) and the plugin writes
  it into ISPL, validates it, builds it on your IntelliSurvey server, and reports back, with as
  much or as little discussion along the way as you want. No code knowledge required.
- **Author ISPL in your own agent** — use the server's ISPL reference and validator as tools in an
  agent workflow to generate and check survey source, then submit it to build a survey.

## What's in the plugin

| Component | What it does |
|---|---|
| `/isvy:surveys` skill | The workflow: build / edit / launch surveys via the server tools, document intake, theme handling, etc. Also authors SPL: the syntax reference is fetched from your connected server on demand |
| `/isvy:getting-started` skill | Orientation for new users: what the plugin does, how to sign in, example prompts |
| `/isvy:doctor` skill | Diagnoses the connection to your server: config, hostname sanity, reachability, sign-in state |
| `isvy` MCP server | Connects to your IntelliSurvey server for survey creation and management, plus ISPL reference documentation and validation for your own workflows |

## Install

Two commands; fill in your IntelliSurvey server's hostname in the second:

```
claude plugin marketplace add IntelliSurvey/plugins
claude plugin install isvy@intellisurvey --config server_name=your-server.intellisurvey.com
```

`--config` saves the server name at install time. Only the hostname is configured: `https://` and `/mcp` are added automatically.
There is no default: the plugin requires a server name, and a missing one leaves the connection unusable (see Troubleshooting).

Omitting `--config` still installs: the CLI warns and names the fix (`/plugin configure isvy@intellisurvey`), while an **in-session** install
(`/plugin install isvy@intellisurvey`) will prompt for the value interactively.

`claude plugin` commands exist only in the **Claude CLI**. The desktop app does not support 'plugin' or '/plugin', but it runs CLI-installed plugins.
Install once via CLI then use either program.
Connection origin differs by Claude surface:
  the CLI and the desktop **code** tab connect from your machine;
  the desktop **chat** tab connects from Anthropic's cloud.

Then, in a new Claude Code session, authenticate: `/mcp` → **isvy** → **Authenticate**. A browser opens to your IntelliSurvey server.
Log in with your normal IntelliSurvey credentials (SSO included) and authorize the connection. After that, the connection refreshes silently;
no further authorization is required.

Optional: `--config default_theme=<theme-id>` presets the theme used for new surveys. When unset, the plugin asks for a survey theme
on your first build and offers to save the answer as the default.

## Using it

### Build a survey from a document

Hand the plugin a questionnaire and it does the rest — convert the document, author the ISPL,
validate, build, and summarize — running straight through or pausing to discuss anything
ambiguous (question types, quota targets, skip logic) before it builds.

- "Here's our questionnaire — turn it into a survey." (attach the file or give its path)
- "Build the survey from this doc, but check with me on anything ambiguous first."
- "Add an NPS question to the end of survey `kxq2f`."
- "Please remove the option text below the option images in Q9."
- "Is my survey ready? Publish it so I can test it."

The agent talks in terms of questions, sections, and logic — not raw survey code. Going live
always requires your explicit confirmation. For anything the tools don't cover yet, the
IntelliSurvey web app remains the full control panel.

Document intake differs by surface. The desktop chat tab reads attached documents natively;
the CLI and desktop code tab read Markdown/text/PDF locally and route local `.docx`/`.xlsx`/`.odt`
files through the server — so office-document import works everywhere except as a *file path* given
to the chat tab (attach the file there instead).

### Author ISPL in your own agent workflow

The server exposes IntelliSurvey's ISPL syntax reference (`get_spl_reference`) and the survey
validator (`validate_ispl`) as MCP tools. An agent can pull the reference, author ISPL itself,
validate and fix it in a loop, then submit the result with `create_survey` / `update_survey` to
build on the server. The plugin's own skills are one consumer of these tools; the same tools
compose into any agent workflow you build.

## Troubleshooting

Run `/isvy:doctor` for step-by-step connection diagnostics. Common cases:

- **isvy is missing from `/mcp`** (or shows an unresolved `${user_config.server_name}` URL):
  no server name is saved — an entry whose URL can't be resolved is dropped silently. Set one
  with `/plugin configure isvy@intellisurvey` (or reinstall with `--config server_name=<host>`),
  then run `/reload-plugins`.
- **`✘ Failed to connect` with a redacted URL** (`https://[REDACTED]@…` in `claude mcp list`):
  the saved server name contains an `@` — everything before it parses as a URL credential and
  gets masked. The server name is saved as-is and unvalidated; on any connection failure, check
  that `claude mcp list` shows exactly `https://<your-host>/mcp`.
- **Permission prompts for routine actions** (validating, checking build status): the surveys
  workflow skill isn't active in this session — say `/isvy:surveys` and the approvals re-arm.

## Multiple servers

The plugin bundles one configurable connection. To work against additional IntelliSurvey
servers at the same time, add them natively in Claude Code:

```
claude mcp add --transport http isvy-prod https://your-other-server.example.com/mcp
```

Each server authenticates independently — sign-in is per server, so every entry gets its own
browser sign-in and its own tokens. The native route is also the escape hatch for anything the
hostname template can't express (plain `http://`, a non-standard port, a different path).

## Authentication

The plugin ships no authentication configuration: the connection uses standard MCP OAuth,
discovered from the server. IntelliSurvey is its own authorization server. It wraps your
organization's existing federated login (SSO included), shows a consent screen, and issues
short-lived tokens with silent refresh. A token resolves to your IntelliSurvey user, and your
existing IntelliSurvey permissions govern everything the plugin can do on your behalf.
