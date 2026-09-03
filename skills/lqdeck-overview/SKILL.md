---
name: lqdeck-overview
description: Orientation for LQDeck, a monitoring system for cron jobs, application errors, website uptime, and cloud-run AI "triage tasks". Use this whenever the user mentions LQDeck, asks what tools the lqdeck MCP server offers, wants to browse monitored projects/errors/crons, or asks something LQDeck-shaped ("what's failing", "any errors today", "what's the AI agent working on") without saying which specific workflow they want. Also read this on first use of the lqdeck MCP server to understand the OAuth login flow.
---

# LQDeck overview

LQDeck is a monitoring backend that tracks, for a portfolio of applications:

- **Crons** — scheduled jobs, their recent runs, and failure history.
- **Errors** — application errors reported by the monitored app's error logger.
  Each stored occurrence is open until someone resolves it (`resolve_errors`).
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
complete, and continue once it succeeds. No manual token entry is needed.

### When Claude Code runs on a remote machine

OAuth finishes on a loopback callback (`http://localhost:<port>/callback`), which
only the machine running Claude Code can answer. Over SSH, in a container, or in
a server-hosted IDE the browser is somewhere else, so that callback is
unreachable — and the flow stalls with the browser showing nothing useful.

LQDeck handles this: after the user approves, it serves a page carrying the whole
callback URL instead of redirecting. **Ask the user to copy that address and
paste it back into Claude Code** when it prompts for the URL from their browser;
the page spells out the same three steps. The address holds a one-time code, so
it works once and expires quickly — if it fails, start the sign-in again.

If the environment cannot open a browser at all (CI, an unattended script), fall
back to a personal access token — see "Manual token" in this plugin's README.

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
- `list_uptime` — live website uptime monitors (last check, consecutive
  failures, open outage). Optional `project_id`. A confirmed outage is an
  open downtime row, not a single failed check.
- `list_crons` / `get_cron_runs` — scheduled jobs and their run history.
- `list_triage_tasks` / `get_triage_task` — AI triage tasks and their detail
  (conversation thread, display state, linked PR).
- `list_tasks_awaiting_human` — tasks stopped and waiting on a human (needs
  human attention, needs more work, handed off, or PR ready to merge). This is
  the entry point for "what needs my attention" — see the `lqdeck-handoff`
  skill for what to do next.
- `list_source_types` — catalog of trigger/context source types a project can
  be configured with.
- `describe_resource` — what can be written on a `project`, `triage_source` or
  `cron`: fields, allowed values, current values, and `steps` — the admin
  wizard's own grouping and order, for walking a person through a setup one
  step at a time. **Call this before any write you are not certain of** — see
  `lqdeck-admin`.
- `test_source_connection` — probe a source's credentials (database, connector
  database, logs, Freelo, Asana, e-mail/IMAP) before saving them, like the
  admin's "Test connection".
- `lookup_asana_gids` — turn Asana names into the gids an Asana trigger's filter
  is built from.

**Write / administer**
- `update_project` — change a project's configuration, including the
  orchestrator's default model, effort and agent harness.
- `configure_source` / `delete_source` — a project's trigger or context sources.
- `save_cron` / `delete_cron` — cron definitions.
- `create_project` — provision a new monitored project, accepting everything the
  admin's create wizard does: uptime URLs, outage alert contacts, Slack error
  reporting, the orchestrator configuration (returns its API key once).
- `delete_project` — requires `confirm_name` matching the project name exactly.
- `rotate_project_api_key` — mint a fresh connector/browser API key; the only
  tool that ever returns a secret, and only once.
- `send_followup` — reply into an open triage task's conversation. **This
  wakes the cloud agent up again** — see `lqdeck-handoff` for when this is
  (and isn't) the right move.
- `resolve_errors` — mark application errors as resolved once they are dealt
  with, or reopen them with `resolved: false`. Per stored occurrence, not per
  error class; needs `Update:Error`. See `lqdeck-context`.
- `archive_task` — archive a triage task that has been dealt with, so it stops
  showing up in `list_tasks_awaiting_human`. A wider task tool set (claim,
  assign, reset, revive, PR actions, terminal-ide handoff) lives behind
  `lqdeck-handoff`.
- `dispatch_task` — send a handed-off task to the AI agent (the dispatch gate;
  starts an autonomous run). See `lqdeck-handoff`.
- `get_connector_install_guide` — generates a ready-to-follow install prompt
  for wiring a new app to LQDeck. See `lqdeck-connect`.

Configuration writes go through the same validation as the admin panel, so a
setting a human could not save is not saveable here either — `lqdeck-admin` has
the rules, plus the tool sets for cron groups, error ignore/threshold rules,
website downtimes, and (super admin only) organizations/users/roles.

**Verification** — `test_git_credentials`, `test_slack_webhook`,
`test_uptime_url`, and `list_git_branches` each try one credential before or
after it is saved; `test_project_integration` runs the whole checklist for a
project in one call (connector seen, uptime, Slack, git, every source) and is
the last step of any setup.

**Prompt**
- `setup-project` — the complete guided flow for connecting the current
  repository to LQDeck end to end: discovery, project, connector, monitoring,
  sources, secrets, and a final `test_project_integration` check. Bound to the
  `/lqdeck:setup` command; mirrors what `lqdeck-connect` describes.

## Where to go next

- Creating a project, an AI trigger source or an AI context source — including
  walking a person through it interactively → `lqdeck-admin`.
- Changing a project's / source's / cron's configuration → `lqdeck-admin`.
- Picking up a cloud agent's work locally, or sending it a follow-up →
  `lqdeck-handoff`.
- Investigating an error or cron failure using the project's own DB/logs, and
  closing errors off once handled → `lqdeck-context`.
- Wiring a new or existing app up to LQDeck monitoring → `lqdeck-connect`.
