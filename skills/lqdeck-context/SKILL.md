---
name: lqdeck-context
description: Investigating an application error, failed cron, or production issue using LQDeck's context sources — reading error details, cron run history, running a read-only SQL query against a project's own database, or reading a project's production logs. Use when the user wants to debug something reported in LQDeck, e.g. "why did this cron fail", "look up this error", "check the production DB for this record", "read the logs around this error's timestamp".
---

# Investigating with LQDeck's context sources

LQDeck stores its own monitoring records (errors, cron runs) but can also reach
into a *monitored project's own* database and log files, if that project has a
`database`/`connector_database` or `logs` context source configured
(`configure_source`, catalog via `list_source_types`). Those two tools require
the `UseContextSources:Project` permission on top of normal organization
access — if a call fails with a permission error, tell the user they need that
permission or the source isn't configured yet.

## Typical debugging flow

1. **Find the error or cron failure.**
   - `list_errors` (filter by `project_id`, `level`, `resolved`, `since`, or a
     `search` substring on the message) to find candidates.
   - `get_error` with the `error_id` for full detail: message, (truncated)
     request data and body, timestamps.
   - For a cron: `list_crons` to find the cron, then `get_cron_runs` with
     `cron_id` (and optionally `hours`, default 24, capped at 168) for recent
     execution history and stats.

2. **Cross-check against the project's own data**, if a database context
   source is configured:
   - `query_project_db` with `project_id` and a single **read-only** `query`
     (`SELECT`/`WITH` only — no DML/DDL, no multiple statements, enforced
     server-side). Use `params` for positional `?` placeholders instead of
     string-interpolating values. Results are capped in row count and total
     size; a truncated response is marked as such — narrow the query
     (add `LIMIT`, tighter `WHERE`) rather than assuming you got everything.

3. **Read the application's logs**, if a `logs` context source is configured:
   - `read_project_logs` with `project_id` and an `action`:
     - `list` — list files/dirs (optional `path` subdirectory, `search`
       substring filter on file names, `page`/`itemsPerPage`).
     - `stat` — metadata for one `file`.
     - `view` — page through a `file`'s contents (`page`).
     - `search` — find a `q` string inside a `file`, with `context` lines of
       surrounding text (1–300) and `direction` for which side to include.
   - A good pattern: `list` to find the right log file, `search` for the error
     message or a request id near the error's timestamp, then `view` around
     the matching page for full context.

4. **Synthesize.** Combine the error/cron record, the DB query result, and the
   log excerpt into a root-cause explanation before proposing a fix — don't
   guess from the error message alone when the context tools are available.

## Notes

- `list_errors` defaults to unresolved errors across every accessible project
  when no `project_id`/`resolved` filter is given.
- These context tools reach into the *monitored application's* own
  infrastructure and credentials (configured per-project on LQDeck's side),
  not LQDeck's own database — treat query results as belonging to that
  project's data, not LQDeck's.
