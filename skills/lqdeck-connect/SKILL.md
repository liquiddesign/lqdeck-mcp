---
name: lqdeck-connect
description: Connecting an application (Nette, Laravel, or a plain website/frontend) to LQDeck for cron and/or error monitoring — detecting the framework, choosing or creating the LQDeck project, and following the generated install guide. Use when the user wants to "hook this app up to LQDeck", "add error monitoring", "wire up the connector", "set up monitoring for this project", or asks how to integrate LQDeck.
---

# Connecting an app to LQDeck

LQDeck ships connector packages for Nette and Laravel apps (cron + error
monitoring, or error-only), and a plain JavaScript snippet for browser error
reporting on any site. `get_connector_install_guide` generates a complete,
ready-to-follow prompt with the project's real API key and endpoint URLs
already filled in — never hand-write the connector configuration yourself.

## The main path: `/lqdeck:setup`

For a full setup, run the `/lqdeck:setup` command (or invoke the `setup-project`
MCP prompt directly) inside the repository being connected. It chains together
everything this integration needs, not just the connector: verifying MCP
access, discovering the framework/git remote/connector endpoints from the repo
itself, finding or creating the matching LQDeck project, following the connector
install guide, configuring uptime monitoring and triage sources, gathering
secrets (Slack webhook, git credentials, alert contacts, Freelo/Asana) right in
the conversation, and finishing with a `test_project_integration` PASS/FAIL
report. It is idempotent — safe to re-run against a project that already
exists to fill in whatever is still missing, including pointing it at a known
project with `project_id` to skip project creation entirely.

Prefer this over the manual steps below whenever the user wants the app
properly wired up, not just the connector package installed.

## Doing it by hand (fallback)

Use this when only the connector itself is wanted — e.g. the project already
exists and is fully configured, and this is just another repository being
connected to it, or the user explicitly wants to walk through each step
themselves.

1. **Detect the framework** of the current repository:
   - `composer.json` requiring `nette/application` (or other `nette/*`
     packages) → `"nette"`.
   - `composer.json` requiring `laravel/framework` → `"laravel"`.
   - A `package.json` with no matching PHP framework (plain website or
     frontend-only project) → `"js"` (browser connector).
   - If both a PHP framework and a `package.json` are present, prefer the PHP
     framework for backend error/cron monitoring; the `"js"` guide can
     additionally be run later for browser-side error reporting.

2. **Pick or create the LQDeck project.** Call `list_projects` and ask the
   user which project this repository corresponds to. If none match, ask
   whether to create one and call `create_project` with a sensible name (e.g.
   the repository name). `create_project` returns the plaintext API key —
   this is the only time it is exposed outside the admin UI (besides
   `rotate_project_api_key`), so use it immediately in the install guide and
   don't try to fetch it again later.

3. **Get the install guide.** Call `get_connector_install_guide` with the
   `project_id` and the detected `framework`. For `"nette"`/`"laravel"`, ask
   the user whether they want cron monitoring or error-reporting only and pass
   `errors_only` accordingly (ignored for `"js"`). The response includes a
   complete `install_prompt` with the real API key, endpoint URLs, package
   name, and (for `"js"`) a public browser key — minted and saved
   automatically if the project doesn't have one yet.

4. **Follow the `install_prompt` exactly**: install the package, wire up the
   configuration it describes, and make the code changes it calls for
   (registering crons, wiring the error logger/reporter, adding the script tag
   or calling `init()`). Don't invent a different configuration shape — the
   prompt already reflects this specific project's real values. Leave the
   changes uncommitted and tell the user what changed.

5. **Verify it actually works** before finishing:
   - `"nette"`/`"laravel"`: trigger a real cron run (or a test error if
     `errors_only`) and confirm in the LQDeck admin panel (the `admin_url`
     from the tool response) that it was recorded — or call
     `test_project_integration` for the same check plus everything else
     already configured on the project.
   - `"js"`: trigger a test error in the browser (e.g.
     `throw new Error('lqdeck test')`) and confirm it appears in the admin
     panel's error list for the project.
   - If verification isn't possible in the current environment (e.g. no way
     to deploy or run the app here), tell the user exactly what to check once
     it's deployed — don't silently skip this step.
