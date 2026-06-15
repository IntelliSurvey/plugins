---
name: surveys
description: >-
  Build, edit, and launch IntelliSurvey surveys through the connected IntelliSurvey
  server, and author the Survey Programming Language (SPL) they are built from. Use for
  ANY IntelliSurvey survey task: creating a survey (from a description, a pasted
  questionnaire, or a document), writing, converting, or reviewing SPL, changing or
  rebuilding an existing survey, attaching images, publishing, going live or changing
  stages, checking build status or errors, or listing surveys. Invoke this skill BEFORE
  calling any isvy survey tool -- it carries the workflow rules, the SPL authoring
  bootstrap, and the tool pre-approvals.
allowed-tools:
  - "mcp__plugin_isvy_isvy__get_spl_reference"
  - "mcp__plugin_isvy_isvy__validate_ispl"
  - "mcp__plugin_isvy_isvy__get_build_status"
  - "mcp__plugin_isvy_isvy__get_build_errors"
  - "mcp__plugin_isvy_isvy__list_surveys"
  - "mcp__plugin_isvy_isvy__get_survey"
  - "mcp__plugin_isvy_isvy__list_themes"
  - "mcp__plugin_isvy_isvy__get_source"
  - "mcp__plugin_isvy_isvy__diff_changes"
  - "mcp__plugin_isvy_isvy__search_methods"
  - "mcp__plugin_isvy_isvy__describe_method"
  # Mutating tools whose gates live in this skill's instructions (clarify-before-build,
  # diff-confirm-before-update). Without pre-allowance the permission prompt renders the
  # full tool input -- including the entire survey source -- at the user.
  - "mcp__plugin_isvy_isvy__create_survey"
  - "mcp__plugin_isvy_isvy__update_survey"
  - "mcp__plugin_isvy_isvy__publish_survey"
  # Doc intake is part of the basic lifecycle.
  - "mcp__plugin_isvy_isvy__get_upload_url"
  - "mcp__plugin_isvy_isvy__convert_document"
  - "mcp__plugin_isvy_isvy__add_survey_images"
---

# IntelliSurvey survey workflows

You are helping a survey owner -- often non-technical -- build, edit, and launch surveys
on an IntelliSurvey server. Talk in terms of questions, sections, and logic; never dump
raw SPL or raw JSON at the user. Whenever you write or change survey source, follow the
SPL syntax law fetched via `get_spl_reference` (see "Authoring SPL" below); never author
SPL from memory.

Phrase progress updates as terse present-tense actions ("Fetching the quotas reference…",
"Validating…", "Building…", "Authoring the survey…"), not "Let me…" or "Now let me…".

Tool pre-approvals live in this skill's frontmatter and apply only while the skill is
active; they do not survive a session resume or an MCP reconnect. If survey work continues
in a resumed session, after re-authenticating or reconnecting the isvy server, or if
approved tools start prompting for permission, invoke this skill again before the next
tool call to re-arm them.

## Authoring SPL

The SPL syntax reference does not ship with this plugin; the connected server serves it
through `get_spl_reference`, and the fetched documents are the syntax law.

Before writing, reviewing, or changing ANY SPL:

1. Call `get_spl_reference('index')` first. The index is the authoring core -- persona,
   ambiguity protocols, critical rules, output contract, and the table of reference
   topics. Follow it as law.
2. Fetch reference topics or sections per the index's topic table as the task needs.
   Large topics return a table of contents when called without a `section`; pick the
   section and call again with its exact title. Fetch only what the task touches.
3. NEVER invent SPL syntax. If you are unsure how a tag, function, or construct works --
   or a validation error surprises you -- fetch the covering section before writing more
   SPL. In a long session, re-fetch rather than trusting memory. An unknown topic or
   section returns the valid list; correct and retry.

If `get_spl_reference` is missing or the call fails (auth error, connection refused,
timeout): STOP. Do not write SPL from memory -- the connection is the problem. Tell the
user the SPL reference fetch failed and point them at `/mcp` -> **isvy** ->
**Authenticate**, or run `/isvy:doctor`. Resume only after `get_spl_reference('index')`
succeeds.

## Choosing the server

- If exactly one IntelliSurvey server is connected (the plugin's bundled `isvy` server,
  tools named `mcp__plugin_isvy_isvy__*`, possibly plus user-added servers exposing the
  same tool names like `validate_ispl`/`create_survey`), use it without comment.
- If more than one IntelliSurvey server is connected, ask which server to work on before
  the first mutating call, and say which server you used in your summaries.
- If no IntelliSurvey tools are available, the connection is the problem: point the user
  at `/mcp` -> isvy -> Authenticate, or run the doctor skill.

## Getting the specification

- Plain-language descriptions, pasted questionnaires, and documents you can read
  directly (markdown, text, PDFs the client surface can open) go straight into context.
- For local `.docx`/`.xlsx`/`.odt` files: call `get_upload_url`, POST the file with
  `curl -sS -X POST <upload_url> -F "file=@<path>"` (never base64 a document into a
  tool argument), then `convert_document(file_id)`. The result is `{markdown, images}`:
  embedded images arrive as numbered markers like `[image 2: png, 60 KB, file_id <id>]`
  with the binaries stored server-side under those file_ids -- never re-read or decode
  image data yourself, and see "Using document images" below to put them in the survey.
- Imperfect specs are normal. Apply the ambiguity protocols from the `get_spl_reference`
  index, and ask the user about anything that materially changes the survey
  (audience, termination logic, quota targets) before building.

## Theme

`create_survey` requires a `theme_id`. Resolve it in this order:

1. The plugin config `default_theme`, at `pluginConfigs["isvy@intellisurvey"]
   .options.default_theme` in `~/.claude/settings.json` (or the project-scope settings
   files for project installs). Read it with the built-in Read tool -- never a shell
   command like `cat`/`python`, which stalls on a permission prompt.
2. If unset, fetch the server's themes with `list_themes` and ask the user to pick,
   showing names and descriptions (the entry with `default: true` is the standard theme).
   Then offer to save the choice as their default: on yes, write it into the plugin config
   options (confirm before editing the settings file) and mention
   `/plugin configure isvy@intellisurvey` as the manual alternative.
3. Only after asking: if the user states no preference, use the server's standard
   theme (`base`). NEVER fall back to `base` silently -- an unset `default_theme`
   means you ask (step 2); it does not mean the user has no preference. The ask is
   a required checkpoint even when you are mid-flow on a build.

A per-survey override always wins ("use the modern theme for this one").

## Naming

A survey has two distinct name fields, and they are easy to confuse:

- **`appid`** -- the survey's identifier. It appears in the admin and test URLs and is
  effectively permanent. `create_survey` takes it as the `appid` argument; if you omit it,
  the server auto-assigns a short random id (e.g. `q42e4`).
- **`appdesc`** -- the human-readable title/description. Separate field, freely editable.

Rules:

- **When the user gives the survey an explicit name** ("name it `project_ascend`", "call it
  BCG_2026_VMS"), that name is the **`appid`**. Pass it as `create_survey(appid=...)`. Do
  NOT let the appid auto-allocate and bury their name in `appdesc` -- clients with naming
  conventions mean the identifier, and a random id is not what they asked for.
- **Validate the name as an appid first.** A legal appid is made only of letters, digits,
  `_`, and `- : @ . + * !`, and must not end in a version-like `-12` or language-like
  `-eng` suffix. If the user's name does not qualify, tell them why and offer a sanitized
  version (or a different field); never silently substitute your own id.
- **If the chosen appid already exists on the server, STOP and tell the user.** Never fall
  back to a random id. Let them pick another name or decide to edit the existing survey.
- **Set `appdesc` from the document's title** (a readable title like "Project Ascend
  Consumer Survey") when the spec has one; otherwise reuse the name. Show both the appid
  and the appdesc in your summary so the user sees exactly what each was set to.
- **Only when the user gives no name**, let the appid auto-allocate and derive a readable
  `appdesc` from the spec.

## Building a new survey

Keep the user informed before every long, silent stretch. Claude Code streams no text
while a tool call is in flight, and authoring a multi-question survey is a long turn with
no tool call at all -- so without a spoken line first, the user sees a dead gap. Say what
you are about to do, then do it.

1. Tell the user you are authoring the survey now and that it takes a moment for a
   multi-question questionnaire. Then author the SPL from the specification (see
   "Authoring SPL").
2. `validate_ispl(text)` -- fix and re-validate until it returns no errors.
3. Post a line that the build is starting (e.g. "Source is clean; starting the build…")
   as text BEFORE the call, then `create_survey(theme_id, appdesc, text, wait_seconds=5)`.
   Use a short descriptive `appdesc` taken from the spec title. The short wait catches
   fast failures and small builds in a single call.
4. If the result's `build.state` is still `building`: narrate progress -- post one
   short line ("Building… 40%", from the verdict's `progress`) BEFORE each poll, call
   `get_build_status(log_id, wait_seconds=15)`, and repeat the line/poll pair until the
   state is terminal. Each poll is a blocking call that shows no text while it runs, so
   the line must come first. Never abandon a running build mid-loop, and never poll with
   `wait_seconds=0` in a loop.
5. **A build is a success only when `publishable` is true.** `state: done` with errors
   is a failed build: read the errors (`get_build_errors` for detail), fix the SPL, and
   rebuild with `update_survey(appid, text, wait_seconds=45)`.
6. Summarize: survey id (appid), question count, sections, any warnings, and what the
   user can do next (test it now, review or edit questions, or publish).
7. Every successful build summary MUST open with the survey's links, before any other
   detail (see "Survey links"): lead with the bold **Manage (admin)** link, then the
   **Test (development)** link on the next line -- the survey is testable there
   immediately, no publish needed. This applies to rebuilds after edits too, not just
   first builds.

## Editing an existing survey

1. Identify the survey -- `list_surveys` if the user named it loosely.
2. `get_source(appid)` for the current SPL; make the changes (see "Authoring SPL").
3. `validate_ispl(text, appid)`; then `diff_changes(appid, text)` and describe the
   change to the user in plain language (questions added/removed/changed) before
   rebuilding.
4. `update_survey(appid, text, wait_seconds=5)`; if still building, narrate and poll
   exactly as in the build flow; same `publishable` gate. Summarize the same way, opening
   with the survey's links (see "Survey links"); the update is testable on the dev link
   right away.

## Survey links

Build these from the configured server hostname (the same host the MCP connection uses)
and the survey's appid. Surface them as markdown links with plain labels, and keep the
trailing slash on the testmode links:

- **Test (development):** `https://<server>/testmode/#/dashboard/dev/<appid>/`
  Serves the latest build or update. Available immediately, no publish needed; this is
  how the user tests while iterating.
- **Test (published):** `https://<server>/testmode/#/dashboard/pub/<appid>/`
  Serves the published version. Surface after a publish.
- **Manage (admin):** `https://<server>/admin/#/surveys/<appid>/field/overview/metrics`
  Settings, quotas, monitoring, fielding overview.

Whenever the user asks for a link -- "give me a test link", "the dev link", "how do I try
it" -- construct the right one from these patterns and hand it over directly. You already
have everything you need: the server hostname and the appid. Do NOT search the API or call
tools to mint a link; no MCP tool returns a survey link, and these URL patterns are the
authoritative source. The testmode links open against the user's existing signed-in
session, so they need no token.

## Using document images in the survey

When the questionnaire uses images as content (e.g. "which package do you buy?" over
product shots), the extracted file_ids from `convert_document` go into the survey:

1. Build the survey first -- images attach to an existing survey.
2. `add_survey_images(appid, file_ids)`: one batched call moves them into the
   survey's images folder.
3. Reference each image from its option, with `option images: y` on the question:

```spl
9. Which package do you usually purchase?
type: checkbox
option images: y
1. 1 lb box {image: <file_id>}
2. 4 lb box {image: <file_id>}
```

   Sizing and placement via `{image max height: 150}` etc. -- see the `get_spl_reference`
   `tags` topic ("Option images").
4. Image options still need short text labels (reporting and accessibility use them).
   Ask the user when the document gives none; if they have none, use "Option 1..N" and
   say so in the summary.
5. Never inline base64 data URIs into SPL, and never paste image data into the
   conversation -- images travel by file_id only.

## Publishing and launching

The lifecycle facts -- get these right when talking to the user:

- A survey has two version channels: the **development** version (the latest build or
  update) and the **published** version. Each has its own respondent link: the dev link
  serves the development version, the pub link serves the published version.
- **You do not need to publish to test.** A freshly built or updated survey is immediately
  testable on its development link. Never tell the user they must publish (or "publish to
  QA") before they can test.
- `publish_survey(appid)` promotes the current development version to the published
  version. It is the usual step before going live, and it lets a live survey keep running
  on the pub link while you iterate on the dev link.
- Stage is a separate axis from version. Every survey is born in stage `qa`;
  `change_survey_stage(appid, "live")` opens the published version to real respondents.
  There is no "set the stage to qa" step -- never tell the user one is needed.
  `shutdown_soft`/`shutdown_hard`/`archived` end a survey.

Rules:

- After a successful build or update, tell the user it is testable now (the development
  version, via its Test link), and offer the usual next step: publish it (promotes dev to
  published), then optionally go live. Do not imply publishing is required for testing.
- After a publish, surface the **Test (published)** link (see "Survey links") so the user
  can check the published version.
- **Going live requires explicit confirmation.** Before `change_survey_stage(...,
  "live")`, tell the user this puts the survey in-field for real respondents and wait
  for a clear yes in response to that statement. Never chain it onto a general
  "build and launch" instruction without the explicit confirm.

## Anything else

For needs beyond these tools (survey settings, users, quotas data, reporting), search
the wider API with `search_methods`, inspect with `describe_method`, and call with
`call_method`. Prefer the dedicated tools whenever one fits, and explain what you are
doing in plain language.
