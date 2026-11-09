---
name: go
description: Use at a natural breaking point to write a handoff note, clear the context, and resume in a fresh session from the note, in any repo
argument-hint: "[focus for the next session]"
user-invocable: true
disable-model-invocation: true
allowed-tools: Bash, Read, Write
---

# Hand Off and Go

Write a note that lets a fresh session pick up this work cold, then arm it.
When this turn ends, the plugin runs `/clear` and starts the new session
from the note by itself. The repo's CLAUDE.md already loads in every
session; the note carries only what CLAUDE.md does not: this session's
state, decisions, and next step.

## Steps

### Step 1: Locate the Note

Run via Bash:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/scripts/handoff.py" path --project-dir "${CLAUDE_PROJECT_DIR}"
```

Parse the JSON. On `status: "error"`, show `message` and stop. Keep `path`
and `workspace_detected`.

### Step 2: Update the Workspace Project (Optional)

Do this step only if all three are true:

- `workspace_detected` is true.
- A workspace project is loaded in this conversation.
- The `workspace:update-project` skill is available.

Otherwise skip it. If they hold, update the project docs now, in this
turn, with the rules of `workspace:update-project`. Do the edits inline:
do not invoke that skill and do not start a fork or a background agent.
The handoff is cancelled if this turn ends before Step 5 arms the note,
so the docs update, the note, and the arm must all happen in one turn.

### Step 3: Capture the Git State

Run via Bash:

```bash
git branch --show-current; git status --short | head -30; git log --oneline -5; git log --oneline @{upstream}..HEAD 2>/dev/null
```

Skip this step outside a git repo. Uncommitted and unpushed work is the
detail the next session most often gets wrong, so record it exactly.

### Step 4: Compose the Note

Write from this conversation, not from re-reading files. If `$ARGUMENTS`
is not empty, make it the next task unless the conversation clearly
contradicts it; say so if it does.

Use exactly this structure, and keep it under 60 lines:

```markdown
# Handoff: <short task label>

**Goal:** <one or two sentences: what we are trying to achieve and why>

## State
- Branch `<branch>`; <uncommitted files / unpushed commits, or "clean">
- <what is done, one line each, the facts the next session would otherwise re-derive>

## Decisions
- <decision> — <why; include rejected alternatives worth not retrying>

## Dead ends
- <what was tried and failed, and the evidence>

## Next task
<the single next action, concrete enough to start without asking>

## Read first
- `<repo-relative path>` — <why it matters for the next task>

## Open questions
- <anything waiting on the user or an external answer>
```

Omit any section that would be empty, except **Goal**, **State** and
**Next task**. List under **Read first** only files the next task needs,
at most five; never the repo's CLAUDE.md, which loads by itself. If
Step 2 ran, list the project's `CLAUDE.md` first, then the detail files
the next task needs.

### Step 5: Write and Arm

Write the note to `path` from Step 1 with the Write tool, then run:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/scripts/handoff.py" arm --project-dir "${CLAUDE_PROJECT_DIR}"
```

On `status: "error"`, show `message` and stop: the session is not
cleared. A note over the size limit must be shortened and armed again.

### Step 6: Report

Tell the user, in one or two lines, that the handoff is armed and the
session clears and resumes at `<next task>` as soon as this turn ends.
End the turn there; do not start the next task in this session.

## Notes

- Writes one file under `~/.claude/next/`. Never edits the repo's
  CLAUDE.md and never commits. The only other writes are the workspace
  project docs in Step 2, and only in a workspace session.
- Keyed by launch directory: each git worktree gets its own note.
- After the note fires, or expires (60 minutes by default), it moves to
  `<key>.consumed.md` beside it, so it can still be pasted by hand.
