---
description: Find your waiting LQDeck triage tasks and prepare to take one over locally
---

Find work waiting on the user in LQDeck and prepare to pick it up locally.

1. Call the `lqdeck` MCP server's `list_tasks_awaiting_human` tool (pass
   `project_id` only if the user already named a specific project in
   `$ARGUMENTS`). If it returns nothing, tell the user there's nothing waiting
   and stop.
2. Present the results as a short numbered list: task id, project name, a
   short title/summary, and its state (needs human / needs work / handed off /
   PR ready to merge).
3. Ask the user which one to pick up — unless there is exactly one result, in
   which case confirm with the user before proceeding.
4. Call `get_task_handoff` with the chosen task's id. Summarize what it
   returns for the user: repository, branch name, linked pull request (if
   any), diff stat, and a short recap of the conversation thread so far.
5. If a local clone of that repository is available in or near the current
   working directory, offer to run the checkout for the user:
   - `git fetch origin`
   - `git checkout <branch>` (or `git checkout -b <branch> origin/<branch>` if
     the local branch doesn't exist yet)
   Otherwise, tell the user the repository URL and branch so they can clone or
   fetch it themselves.
6. Remind the user (see the `lqdeck-handoff` skill for detail): once local
   changes are ready, push with `git push --force-with-lease origin <branch>`,
   not a plain push or a plain `--force` — the orchestrator may have rebased
   this branch since it was last fetched.

Do not call `send_followup` as part of this command — that wakes the cloud
agent back up and would race against the local takeover this command is
preparing.
