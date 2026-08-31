---
name: lqdeck-admin
description: How to create and change LQDeck configuration through the MCP server instead of the admin UI — provisioning projects, setting up AI trigger sources and AI context sources, and crons — including how to walk a person through it interactively, one wizard step at a time, exactly as the admin panel would. Use this whenever the user asks to create, add, change, enable, disable, rename, retune or delete something in LQDeck ("set up a new project", "založ projekt", "add an Asana source", "connect a database as context", "switch project ABC to Opus 5", "turn off automerge", "add a cron", "delete this source"). Read it before the first write in a session.
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

Steps you cannot fill over MCP, and must hand back to the user: **git credentials**
(deploy key / PAT) and **key rotation**. Say so plainly and point at
`/app/projects/{id}/edit` — the orchestrator will not clone until a human has put
the credentials in.

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

Two read-only tools cover the wizard's "try it first" buttons, and you should use
them for the same reason the admin panel has them — so a person is not asked to
trust credentials nobody checked:

| Tool | What it answers |
|---|---|
| `test_source_connection` | does this `database` / `connector_database` / `logs` config actually connect? |
| `lookup_asana_gids` | what are the gids behind these Asana names? (workspaces → projects → sections → custom fields → their options) |

`update_project` returns a `changed` list of exactly which dot-keys moved. Use it
to tell the user what happened — and to notice when a write you thought was a
change was a no-op.

## `orchestrator_settings` merges key by key

```
update_project(project_id: 12, orchestrator_settings: {claude_model: "claude-opus-5"})
```

changes that one key. Its ~30 siblings (effort, agent harness, session limits,
timeouts, automerge, watch window) keep their stored values. Sending an
unrecognised key inside the object is an error rather than a silent write into the
JSON column, so a typo surfaces immediately.

Two settings that are easy to confuse:

- `auto_merge_when_safe` — whether a pull request may be merged without a human.
- `auto_merge_max_risk` (`low` | `medium` | `high`) — the highest merge-safety
  rating still merged automatically. Turning automerge off does not change it.

## Narrowing: what a row accepts depends on its state

The field set is not fixed per resource, and `describe_resource` reflects the
narrowing for the id you pass:

- A **code-managed cron** (schedule lives in the monitored application's own
  code, i.e. pull execution mode) accepts only `active` and `groups`. Its name,
  URL and schedule belong to that repository.
- A **context source** has no agent session, so it accepts none of the per-task
  overrides (model, effort, harness, verification, watch window, hourly cap).
  Only a **trigger** source does.
- A source's `kind` and `source_type` are **write-once** — changing either would
  reinterpret the whole stored `config` payload. Delete and recreate instead.

## Secrets

API keys, git credentials and agent tokens are **never returned** and **cannot be
written** over MCP; discovery reports only whether one is stored
(`secrets_set` / `current_set`). Rotating a key is an admin-panel action — tell
the user that rather than attempting it.

The one exception is a source's `config` payload, which is how credentials get
into a source (a Freelo token, a database password). It is write-only: you can set
it, you can never read it back. **Omit it when editing** and the stored
configuration is kept — so you never need to ask the user to re-type a token just
to change a polling interval.

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

**"Turn off automerge for XYZ."**
`update_project(project_id: …, orchestrator_settings: {auto_merge_when_safe: false})`.
Do not also send `auto_merge_max_risk` — it stays as it was, ready for when
automerge is turned back on.

**"Only take Freelo tasks created after 1 August 2026."**
`get_project` for the source's `id` → `configure_source(project_id: …,
config_id: …, trigger_min_created_at: "2026-08-01T00:00:00+02:00")`. Check
`supports_created_at_cutoff` in `list_source_types` first: the cutoff only means
something for source types that report it. Do **not** send `config` — the stored
Freelo token stays put.

**"Set up a new project for shop.example.com."** (interactive)
`describe_resource(resource: "project")` → walk `steps`: name; uptime URL; who gets
outage alerts; Slack error reporting (skip the webhook if they say no); orchestrator
(skip the whole step if they say no) → one `create_project(...)` call with everything
→ hand them the `api_key` **once**, tell them where it goes (the connector config),
and mention that git credentials, if they enabled the orchestrator, still need the
admin panel. Follow with the `lqdeck-connect` skill to actually wire the app up.

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
