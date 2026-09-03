# LQDeck plugin for Claude Code

Connects Claude Code to [LQDeck](https://lqdeck.petrkrehlik.org) — a monitoring
system for cron jobs, application errors, website uptime, and cloud-run AI
"triage tasks" — via LQDeck's MCP server, plus a handful of skills that teach
Claude how to use it well.

## What's included

- **MCP server** (`lqdeck`) — read tools for projects, errors, crons, triage
  tasks; write tools to administer LQDeck without opening the admin UI (projects,
  triage sources, crons), provision a project, mark errors resolved, follow up on
  a task, and generate connector install guides.
- **Skills** (auto-invoked by Claude based on context):
  - `lqdeck-overview` — what LQDeck is, the tool map, first-use OAuth notes.
  - `lqdeck-admin` — changing configuration over MCP: the discovery contract,
    patch semantics, what is deliberately not writable.
  - `lqdeck-handoff` — picking up a cloud agent's stopped work locally (or
    sending it a follow-up to resume).
  - `lqdeck-context` — investigating errors/crons with a project's own
    database and logs, and marking errors resolved once handled.
  - `lqdeck-connect` — wiring a new or existing app up to LQDeck monitoring.
- **Commands**:
  - `/lqdeck:setup` — connect the current repository to LQDeck in one guided
    pass: discovery, project, connector, monitoring, sources, secrets,
    verification. The main way to onboard a new app.
  - `/lqdeck-pickup` — find your waiting triage tasks and prepare a local
    checkout of one.
- **Prompt** — `setup-project` (served by the MCP server itself, and what
  `/lqdeck:setup` calls): the same guided end-to-end connection flow, callable
  directly by name with `project_name`/`framework`/`project_id` arguments.

## Install

Inside a running Claude Code:

```
/plugin marketplace add liquiddesign/lqdeck-mcp
/plugin install lqdeck@lqdeck
```

Or from a shell — same thing, and this is the form to use when you want the
plugin in *every* project rather than the current one (`claude plugin install`
defaults to `--scope user`):

```
claude plugin marketplace add liquiddesign/lqdeck-mcp
claude plugin install lqdeck@lqdeck
```

The [liquiddesign/lqdeck-mcp](https://github.com/liquiddesign/lqdeck-mcp)
repository is public — no credentials needed. (LQDeck developers can alternatively add the
marketplace from a local clone of the main repo:
`/plugin marketplace add /path/to/liquid-monitor-back`.)

## Authentication (OAuth, no manual tokens)

The first time Claude Code calls an `lqdeck` MCP tool, the server responds
`401`, Claude Code discovers its OAuth metadata, registers itself as a client,
and opens your browser to log in to LQDeck. Approve it there and the session
continues automatically — no token needs to be copied anywhere.

### Claude Code on a remote machine (SSH, container, server-hosted IDE)

OAuth ends on a loopback callback (`http://localhost:<port>/callback`) that only
the machine running Claude Code can answer. If your browser is on a different
machine, that address is unreachable there.

LQDeck covers this case: after you approve, it shows a page with the full
callback address instead of redirecting to it. Copy that address and paste it
back into Claude Code when it asks for the URL from your browser — the page
itself walks you through it. The address contains a one-time code, so it works
once and expires quickly; if it fails, just start the sign-in again.

### Manual token fallback

If a browser can't be opened at all (CI, an unattended script), you can
authenticate with a personal access token instead:

1. In the LQDeck admin panel, go to **`/app/api-tokens`** and create a token.
2. Copy the plaintext token shown once.
3. Configure your MCP client to send it as a bearer token / header for the
   `lqdeck` server (consult your Claude Code MCP client configuration docs for
   the exact mechanism, since this bypasses the plugin's bundled `.mcp.json`
   OAuth flow).

Prefer the OAuth flow whenever a browser is available — it doesn't require
managing a long-lived secret.

## Changing the server URL

The MCP endpoint is hardcoded in `.mcp.json` to
`https://lqdeck.petrkrehlik.org/mcp` (LQDeck's production URL, confirmed
against `config/triage.php`, `deploy/inbound-mail/setup.sh`, and
`.claude/skills/lqdeck-prod-debug/SKILL.md` in the `liquid-monitor-back`
repo). If LQDeck is ever hosted at a different domain, or you want to point
this plugin at a staging/dev instance, edit the `url` field in
`claude-plugin/.mcp.json` and reinstall/reload the plugin.

## Required permissions

- Baseline: an LQDeck user account with access to at least one organization.
  Read tools (`list_projects`, `list_errors`, `list_triage_tasks`, ...) work
  with plain organization membership.
- Write tools (`update_project`, `configure_source`, `save_cron`, the `delete_*`
  tools, `create_project`, `send_followup`, `get_connector_install_guide`)
  require the corresponding Spatie permission within the target organization,
  and run the same validation as the admin UI.
- `resolve_errors` requires `Update:Error` in the organization the errors belong
  to — the client-facing `monitor` role does not hold it, exactly as it does not
  see the admin panel's Resolve button.
- `query_project_db` and `read_project_logs` additionally require the
  `UseContextSources:Project` permission and a `database`/`connector_database`
  or `logs` context source configured on the target project.
