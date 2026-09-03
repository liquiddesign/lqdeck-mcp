---
name: lqdeck-admin
description: How to create and change LQDeck configuration through the MCP server instead of the admin UI — provisioning projects, setting up AI trigger sources and AI context sources, and crons — including how to walk a person through it interactively, one wizard step at a time, exactly as the admin panel would. Use this whenever the user asks to create, add, change, enable, disable, rename, retune or delete something in LQDeck ("set up a new project", "založ projekt", "add an Asana source", "connect a database as context", "switch project ABC to Opus 5", "add a cron", "delete this source"). Read it before the first write in a session.
---

# Administering LQDeck over MCP

Every configuration change goes through **the same validation and the same
business logic as the admin panel** — one shared rule set, one shared save path.
A configuration a human could not save through `/app` is refused here too, with
the same message. That is the point: you can administer LQDeck without opening
the UI and without risking a state the UI would never have produced.

## The four rules

1. **Read before you write.** `describe_resource` returns, for a given resource
   (`project` | `triage_source` | `cron`) and optionally a specific id: every
   writable field, its type, its allowed values, its current value, and the
   permission a write needs. Call it whenever you are not certain of a field name
   or an allowed value.
2. **Never guess an enum value.** Allowed values come from `describe_resource`
   (or `list_source_types` for a source's `config` payload). Where validity
   depends on a sibling field — a model belongs to exactly one agent harness —
   the option says so in `applies_when`. An option listed as `disabled` carries a
   reason (e.g. "grok-4.5 is not available in the EU"); report the reason to the
   user rather than trying to write it.
3. **Every write is a patch.** A field you omit keeps its stored value. You never
   have to read a project's whole configuration and send it back to change one
   setting — and you never should, because re-sending a value you read a minute
   ago is how you clobber a change someone else made in between.
4. **A field the resource does not accept is refused, not ignored.** If a write
   comes back with "this field cannot be written … in its current state", the
   field is genuinely off limits for that row (see *Narrowing* below) — do not
   retry it under a different name.

## Setting something up *with* a person: walk the wizard

When the user wants to create something rather than tweak one value ("set up a new
project", "add an Asana source"), do **not** ask them for forty keys, and do not
invent a shape of your own. `describe_resource` returns `steps` — the admin
wizard's own grouping, in the admin's own order. Walk it:

1. `describe_resource(resource: "project")` — or, for a source that does not exist
   yet, `describe_resource(resource: "triage_source", project_id: …, kind: …,
   source_type: …)`, which additionally returns that source type's `config_fields`,
   their defaults and their help text.
2. Ask for one step's fields at a time, using each field's `label` and
   `description` — they are the same words the admin panel shows.
3. Skip anything whose `relevant_when` is not satisfied by the answers so far. If
   the user does not want the orchestrator, do not ask for a repository URL; if
   Slack error reporting is off, do not ask for a webhook.
4. Respect `required_when`: a field can be required only in one branch (a browser
   key once browser errors are collected).
5. For a database, log or Asana source, run the matching probe **before** writing,
   and report what it said.
6. Write once, at the end, with everything gathered — `create_project` and
   `configure_source` each take the whole payload in one call. A field you never
   asked about simply keeps the default the empty admin form would have submitted.

Nothing here is admin-panel-only anymore. Git credentials (`auth_method`
"deploy_key"/"pat" plus `git_username` and `orchestrator_git_credentials` on
`update_project`, verified with `test_git_credentials` or `list_git_branches`)
and key rotation (`rotate_project_api_key`) both go through MCP, gated by
`UpdateSensitive:Project` — see *Beyond project/source/cron* below.

## Creating an AI source vs. an AI context source

Both are `configure_source`; the difference is `kind`, and it decides the whole
field set:

- `kind: "trigger"` — **opens** triage tasks (Freelo, Asana, e-mail, monitor
  errors, website downtime, cron failures, security review). Takes the scan
  interval, the hourly cap, and the per-task agent overrides.
- `kind: "context"` — **is read by** the agent while it works a task (errors, job
  runs, logs, a database, connector database, triage history). Takes no agent
  overrides at all, because it has no session of its own.

`list_source_types` is the catalog of what exists; `describe_resource` with
`project_id` + `kind` + `source_type` is what that one type needs. Both `kind` and
`source_type` are write-once, so get them right before the first write — and pass
them on the create call, never on an edit.

## The write tools

| Tool | What it writes |
|---|---|
| `update_project` | project columns + `orchestrator_settings` |
| `configure_source` | create or edit a trigger/context source (`config_id` to edit) |
| `delete_source` | remove a source |
| `save_cron` | create or edit a cron (`cron_id` to edit) |
| `delete_cron` | remove a cron |
| `create_project` | provision a new project — the whole create wizard, not just a name; returns its API key **once** |
| `delete_project` | requires `confirm_name` equal to the project's name |
| `rotate_project_api_key` | mint a fresh connector (`key: "connector"`) or browser (`key: "browser"`) API key and invalidate the old one immediately; the only MCP tool that ever returns a secret, and only this once |

Read-only tools cover the wizard's "try it first" buttons, and you should use
them for the same reason the admin panel has them — so a person is not asked to
trust credentials nobody checked:

| Tool | What it answers |
|---|---|
| `test_source_connection` | does this source's `config` (Freelo, Asana, IMAP/SMTP, database, connector database, logs) actually connect? |
| `test_git_credentials` | can the orchestrator reach the project's git remote with its effective credentials (`git ls-remote`, never a clone)? |
| `list_git_branches` | same check, plus the branch list and which one is the remote default — useful before saving `orchestrator_default_branch` |
| `test_slack_webhook` | send a real test message through a Slack incoming webhook (pass `webhook_url` to try one not saved yet) |
| `test_uptime_url` | one synchronous HTTP GET against a candidate or saved uptime URL |
| `test_project_integration` | the full setup checklist in one call — connector seen, every uptime URL, Slack, git access, every configured source — PASS/FAIL/SKIP per item; the last step of any setup |
| `sync_repo` | queue a clone (first time) or fetch (afterwards) of the orchestrator's repo; check completion via `get_project`'s `last_sync` |
| `lookup_asana_gids` | what are the gids behind these Asana names? (workspaces → projects → sections → custom fields → their options) |

`update_project` returns a `changed` list of exactly which dot-keys moved. Use it
to tell the user what happened — and to notice when a write you thought was a
change was a no-op.

## `orchestrator_settings` merges key by key

```
update_project(project_id: 12, orchestrator_settings: {claude_model: "claude-opus-5"})
```

changes that one key. Its ~30 siblings (effort, agent harness, session limits,
timeouts, watch window) keep their stored values. Sending an
unrecognised key inside the object is an error rather than a silent write into the
JSON column, so a typo surfaces immediately.

## Narrowing: what a row accepts depends on its state

The field set is not fixed per resource, and `describe_resource` reflects the
narrowing for the id you pass:

- A **code-managed cron** (schedule lives in the monitored application's own
  code, i.e. pull execution mode) accepts only `active` and `groups`. Its name,
  URL and schedule belong to that repository.
- A **context source** has no agent session, so it accepts none of the per-task
  overrides (model, effort, harness, watch window, hourly cap).
  Only a **trigger** source does.
- A source's `kind` and `source_type` are **write-once** — changing either would
  reinterpret the whole stored `config` payload. Delete and recreate instead.

## Secrets

Every secret is **write-only** over MCP: `orchestrator_git_credentials`, alert
contacts, the Slack webhook, and a source's `config` payload (a Freelo token, a
database password) can all be set — gated by `UpdateSensitive:Project` for the
project-level ones — but never read back. Discovery reports only whether one is
stored (`secrets_set` / `*_set` / `config_set`). **Omit a secret field when
editing** and the stored value is kept — you never need to ask the user to
re-type a token just to change a polling interval or a branch name.

The one value you cannot set directly is the connector/browser API key itself —
`rotate_project_api_key` mints a new one and hands it back exactly once, since
rotation is the only way to hand one over at all. `claude_oauth_token` and
`cursor_api_key` stay `.env`-only; no MCP tool touches them.

## Beyond project/source/cron

The same four rules apply everywhere else in LQDeck; these resources just have
their own tools instead of `update_project`/`configure_source`/`save_cron`.

**Cron groups** — a label a project's crons can be tagged with, to pause or
report on them as a set:

| Tool | |
|---|---|
| `list_cron_groups` | the group uuids for a project |
| `save_cron_group` | create (`project_id`) or edit (`cron_group_id`) |
| `delete_cron_group` | remove one |

**Error rules** — ignore rules drop or hide matching errors; threshold rules
fire a Slack notification and/or open an AI task once a group of errors
crosses a count within a time window. Both can span several projects in the
same organization:

| Tool | |
|---|---|
| `list_error_ignore_rules` / `save_error_ignore_rule` / `delete_error_ignore_rule` | drop-on-ingest or hide-from-AI rules |
| `list_error_thresholds` / `save_error_threshold` / `delete_error_threshold` | count-within-a-window alerting rules |

**Website downtimes** — `save_downtime` corrects or backfills a manual record;
`delete_downtime` removes one; `acknowledge_downtime` takes responsibility for
an ongoing outage (same as the alert link / admin's Acknowledge button — only
works while it is still open and un-acknowledged).

**Triage tasks** — the full task lifecycle (create, claim/unclaim/assign,
reset/wake/revive/duplicate, client replies, PR actions, pushing to
terminal-ide) is its own tool set: `create_task`, `claim_task`,
`unclaim_task`, `assign_task`, `reset_task`, `wake_task`, `revive_task`,
`dismiss_task_revival`, `duplicate_task`, `send_client_reply`,
`save_client_reply`, `dismiss_client_reply`, `merge_pull_request`,
`set_pull_request_state`, `update_pull_request`, `refresh_pull_request`,
`push_task_to_terminal_ide`, `fail_job_log`, `resolve_asana_filter_labels`.
See the `lqdeck-handoff` skill for the workflow these fit into rather than
calling them ad hoc.

**Organizations, users, roles — super admin only.** Everything here requires
the `super_admin` role, not an in-organization permission:

| Tool | |
|---|---|
| `list_organizations` / `save_organization` / `delete_organization` | tenants |
| `attach_organization_project` / `detach_organization_project` | which projects belong to an organization |
| `attach_organization_user` / `detach_organization_user` | who belongs to an organization |
| `list_organization_requests` / `approve_organization_request` / `reject_organization_request` | self-serve join requests |
| `list_users` / `save_user` / `delete_user` | accounts (password is write-only, e-mail is immutable) |
| `list_roles` / `save_role` / `list_permissions` | roles are seeded, not created — rename/re-permission only |
| `set_global_pause` | instance-wide orchestrator emergency stop (not per-project — that's `update_project`'s `orchestrator_enabled`) |
| `cleanup_stuck_tasks` | sweep tasks stuck past their timeout; `preview: true` (default) to see what would happen first |

## Permissions

Writes enforce the caller's permission inside the target organization:
`Update:Project` for a project and for its sources, `Create:Cron` /
`Update:Cron` / `Delete:Cron` for crons, `Create:Project` / `Delete:Project` for
provisioning or destroying a whole project. A refusal names the missing
permission — pass it on to the user instead of retrying.

A project in another organization, or one that does not exist, both answer "not
found or not accessible". The wording is deliberate: never tell the user that a
project id exists but belongs to someone else.

## Worked examples

**"Switch project Abel to Opus 5."**
`list_projects` (search `Abel`) → `describe_resource(resource: "project", id: …)`
to confirm the model is offered for the project's current agent harness →
`update_project(project_id: …, orchestrator_settings: {claude_model: "claude-opus-5"})`
→ report the `changed` list.

**"Only take Freelo tasks created after 1 August 2026."**
`get_project` for the source's `id` → `configure_source(project_id: …,
config_id: …, trigger_min_created_at: "2026-08-01T00:00:00+02:00")`. Check
`supports_created_at_cutoff` in `list_source_types` first: the cutoff only means
something for source types that report it. Do **not** send `config` — the stored
Freelo token stays put.

**"Set up a new project for shop.example.com."** (interactive)
If this is a repository the user has open right now, prefer the `setup-project`
MCP prompt (or `/lqdeck:setup`) over doing it by hand here — it chains exactly
these steps together with repo discovery and a final verification. Doing it by
hand: `describe_resource(resource: "project")` → walk `steps`: name; uptime URL;
who gets outage alerts; Slack error reporting (skip the webhook if they say no);
orchestrator (skip the whole step if they say no) → one `create_project(...)`
call with everything → hand them the `api_key` **once**, tell them where it goes
(the connector config). If they enabled the orchestrator, git credentials go
through `update_project`'s `auth_method`/`git_username`/
`orchestrator_git_credentials` too — no admin-panel step needed. Follow with the
`lqdeck-connect` skill to actually wire the app up.

**"Add our production database as context for the AI."**
`list_source_types` → `describe_resource(resource: "triage_source", project_id: …,
kind: "context", source_type: "database")` for the exact `config_fields` → collect
host/port/database/user/password → `test_source_connection(project_id: …,
source_type: "database", config: {…})` and report the result → only then
`configure_source(project_id: …, kind: "context", source_type: "database",
config: {…})`. Never write credentials you have not probed.

**"Take tasks from our Asana board."**
`describe_resource(... kind: "trigger", source_type: "asana")` → ask for the PAT →
`lookup_asana_gids(lookup: "workspaces", config: {asana_pat: …})` → let the user
pick → `lookup_asana_gids(lookup: "projects", config: {asana_pat: …, workspace_gid: …})`
→ and so on down to sections / custom fields as the filter needs. Never invent a gid.
When editing later, pass `config_id` and omit `asana_pat` — the stored token is used.

**"Delete the old staging project."**
Ask the user for the exact name and pass it as `confirm_name`. Do not read the
name out of `list_projects` and confirm on their behalf — the confirmation exists
to make a human say the name out loud.
