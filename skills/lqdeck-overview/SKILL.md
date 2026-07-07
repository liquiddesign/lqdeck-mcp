---
name: lqdeck-overview
description: Orientation for LQDeck, a monitoring system for cron jobs, application errors, website uptime, and cloud-run AI "triage tasks". Use this whenever the user mentions LQDeck, asks what tools the lqdeck MCP server offers, wants to browse monitored projects/errors/crons, or asks something LQDeck-shaped ("what's failing", "any errors today", "what's the AI agent working on") without saying which specific workflow they want. Also read this on first use of the lqdeck MCP server to understand the OAuth login flow.
---

# LQDeck overview

LQDeck is a monitoring backend that tracks, for a portfolio of applications:

- **Crons** — scheduled jobs, their recent runs, and failure history.
- **Errors** — application errors reported by the monitored app's error logger.
- **Uptime** — website availability checks.
- **Triage tasks** — tickets (from Freelo, Asana, e-mail, or the monitor's own
  error/cron alerts) that are worked autonomously by a cloud-hosted AI agent
  ("the orchestrator"). Each task has a conversational thread, a git branch, and
  usually ends in a pull request.

Everything is reached through the `lqdeck` MCP server's tools — there is no need
to open the admin UI unless the user wants a link to click.

## First use: OAuth login

The `lqdeck` MCP server requires authentication. The **first** time you call any
`lqdeck` tool in a session, Claude Code will get a 401, discover the server's
OAuth metadata, register itself as a client, and open the user's browser to log
in. Just proceed normally — call the tool you need, let the browser flow
complete, and continue once it succeeds. No manual token entry is needed. If a
browser cannot be opened in the current environment, see the "Manual token"
fallback in this plugin's README.

## Data scoping

Every result is scoped to the organizations the logged-in user belongs to. A
project id you already know (e.g. from a previous session or a URL) is not
guaranteed to still be accessible — always resolve project ids through
`list_projects` rather than guessing or hardcoding one.

## Tool map

**Discovery / read**
- `list_projects` — start here. Optional `search` by name.
- `get_project` — one project's crons, source configs, open task count.
- `list_errors` / `get_error` — application errors, filterable by project,
  level, resolved state, date, message substring.
- `list_crons` / `get_cron_runs` — scheduled jobs and their run history.
- `list_triage_tasks` / `get_triage_task` — AI triage tasks and their detail
  (conversation thread, Finish-flow stage, linked PR).
- `list_tasks_awaiting_human` — tasks stopped and waiting on a human (needs
  human attention, needs more work, handed off, or PR ready to merge). This is
  the entry point for "what needs my attention" — see the `lqdeck-handoff`
  skill for what to do next.
- `list_source_types` — catalog of trigger/context source types a project can
  be configured with.

**Write**
- `create_project` — provision a new monitored project (returns its API key
  once).
- `configure_source` — create/edit a project's trigger or context source.
- `send_followup` — reply into an open triage task's conversation. **This
  wakes the cloud agent up again** — see `lqdeck-handoff` for when this is
  (and isn't) the right move.
- `get_connector_install_guide` — generates a ready-to-follow install prompt
  for wiring a new app to LQDeck. See `lqdeck-connect`.

**Prompt**
- `integrate-lqdeck` — a guided end-to-end flow for connecting the current
  repository to LQDeck; mirrors what `lqdeck-connect` describes.

## Where to go next

- Picking up a cloud agent's work locally, or sending it a follow-up →
  `lqdeck-handoff`.
- Investigating an error or cron failure using the project's own DB/logs →
  `lqdeck-context`.
- Wiring a new or existing app up to LQDeck monitoring → `lqdeck-connect`.
