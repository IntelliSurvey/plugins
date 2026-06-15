---
name: getting-started
description: >-
  Orient a new user of the isvy IntelliSurvey plugin. Use when the user invokes
  /isvy:getting-started, asks what this plugin can do, how to start building surveys
  with IntelliSurvey here, or how to connect or sign in to their IntelliSurvey server.
---

# Getting started with the IntelliSurvey plugin

When invoked, orient the user. Keep it warm and short -- they are likely not a
programmer. Cover the points below, adapted to what they asked; don't recite this file.

## What you can do here

- **Build a survey** from a plain-language description, a pasted questionnaire, or a
  document (Word, Excel, PDF). The agent writes the survey program, checks it, builds
  it on their IntelliSurvey server, and reports back -- no code knowledge needed.
- **Edit an existing survey**: describe the change ("add a satisfaction question after
  Q3", "remove the screener age cap") and review a summary of the diff before it is
  applied.
- **Test and launch**: publish the survey, try it in QA, then take it live. Going live
  always requires their explicit confirmation.
- **Check on things**: list their surveys, check build status, review survey settings.

## First-time setup

1. The plugin connects to their IntelliSurvey server over an authenticated link. To
   sign in: run `/mcp`, choose **isvy**, then **Authenticate** -- a browser opens for
   their normal IntelliSurvey sign-in (company SSO included). One sign-in; it refreshes
   silently afterwards.
2. On their first survey build, they'll be asked which **theme** (look and feel) their
   surveys should use; the choice can be saved as their default.

## Example prompts to offer

- "Build a 10-question customer satisfaction survey for a coffee chain."
- "Here's our questionnaire doc -- turn it into a survey." (attach or give the path)
- "Add an NPS question to the end of survey kxq2f."
- "Is my survey ready to launch? Publish it so I can test it."

## If something's wrong

- Server missing from `/mcp`, can't connect, can't sign in, tools unavailable: run the
  doctor skill (`/isvy:doctor`), which diagnoses the connection step by step.
- Permission prompts for routine actions (validating, checking build status): the
  surveys workflow skill isn't active in this session — say `/isvy:surveys` (or "use
  the surveys skill") and the approvals re-arm.
- For anything the tools can't do yet, the IntelliSurvey web app remains the full
  control panel; this plugin focuses on building, editing, and launching surveys.
