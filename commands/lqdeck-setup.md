---
description: Connect the current repository to LQDeck in one guided pass — discovery, project, connector, monitoring, sources, secrets, verification
---

Retrieve the `lqdeck` MCP server's `setup-project` prompt and follow it exactly.

Parse `$ARGUMENTS` for anything the user already told you and pass it as the
prompt's arguments — all optional:

- `project_name` — a project name they mentioned.
- `framework` — `"nette"`, `"laravel"`, or `"js"`, if they said which.
- `project_id` — an existing LQDeck project id, if they said this repository
  already has one (skips straight to filling in what's missing on it).

Leave out anything not mentioned; the prompt detects or asks for it itself.
