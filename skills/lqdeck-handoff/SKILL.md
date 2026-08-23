---
name: lqdeck-handoff
description: Workflow for taking over a triage task that LQDeck's cloud AI orchestrator has stopped on (needs human input, needs more work, was handed off, or has a PR ready to merge) and continuing it in a local Claude Code session — checking out its branch, finishing the work, and pushing back. Also covers the reverse — sending a follow-up message to wake the cloud agent back up. Use when the user says things like "what's waiting on me", "pick up task X", "continue the agent's work locally", "take over this PR", or "tell the agent to keep going".
---

# Picking up (or handing back) cloud-agent work

LQDeck's orchestrator runs an AI agent per triage task, in its own git branch
inside a repo it cloned. Sometimes that agent gets stuck or finishes and needs a
human to review/merge, or a human needs to jump in and finish the last mile
locally instead of prompting the cloud agent further.

## Finding waiting work

1. Call `list_tasks_awaiting_human` (optionally with `project_id`) to list
   tasks stopped in a human-facing state: needs human attention, needs more
   work, handed off, or PR ready to merge.
2. Show the user the candidates (task id, project, title/summary, state) and
   let them pick one — or if there's exactly one obvious match, proceed.

## Taking over locally

1. Call `get_task_handoff` with the chosen `task_id`. It returns everything
   needed to continue: the repository URL, the task's branch name, any linked
   pull request, a diff stat, the verification result (if any), and the full
   human-facing conversation thread — plus specific checkout instructions.
2. In a local clone of that repository:
   - `git fetch origin`
   - `git checkout <branch>` (create a local tracking branch if it doesn't
     exist yet: `git checkout -b <branch> origin/<branch>`)
3. Read the conversation thread and diff stat from the handoff to understand
   what the agent already did and why it stopped.
4. Make whatever changes are needed locally, same as any other branch.
5. **Push with `--force-with-lease`, not a plain push.** The orchestrator
   rebases task branches onto the default branch between turns, so the branch
   history may not be what you last fetched — a plain `git push` can be
   rejected or, worse, a plain `--force` can silently discard the agent's
   commits. `--force-with-lease` fails safely if the remote moved since your
   fetch.
   ```
   git push --force-with-lease origin <branch>
   ```
6. If there's already an open pull request (see the handoff's PR info), finish
   it up and merge as usual. If not, open one from the branch.

## Sending work back to the cloud agent instead

If the task just needs *more agent work* rather than a human finishing it by
hand, use `send_followup` with the `task_id` and a `body` describing what to
do next.

**Warning: `send_followup` wakes the cloud agent up and reopens the task for
processing.** Only use it when you actually want the orchestrator to resume
autonomous work — not as a way to leave a note, and not on a task you are
about to take over locally yourself (that would race the cloud agent against
your local edits on the same branch). It only works while the task's
conversation is still open (its agent session hasn't been archived) — check
`conversation_open` via `get_triage_task` first if unsure.

## Quick reference

| Situation | Action |
|---|---|
| Want to see what's waiting | `list_tasks_awaiting_human` |
| Ready to finish a task's work yourself | `get_task_handoff` → checkout → edit → `push --force-with-lease` |
| Want the cloud agent to keep going | `send_followup` (wakes it up) |
| Task is gate-parked (state `handoff`, never ran) and the agent should take it | `dispatch_task` (same as the admin "Send to agent" button) |
| Just want to read the current state | `get_triage_task` |

Note the dispatch gate: trigger sources default to `dispatch_mode: handoff`, so a
freshly discovered task has **no run and no branch yet** — it waits for a person.
`dispatch_task` starts the autonomous run; `get_task_handoff` on such a task returns
a fresh-start brief (create the branch from base) instead of a take-over.
