# next

A Claude Code plugin that hands a session off to a fresh one.

`/next:go` writes a short handoff note (goal, state, decisions, next
task), clears the context, and starts a new session that reads the note
and continues with the next task. You do not confirm anything.

## Install

```text
/plugin marketplace add pablofontanilla/claude-next
/plugin install next@pablofontanilla
```

Then restart Claude Code.

Requirements:

- A Claude Code version that supports plugin function hooks
  (tested on 2.1.289). On older versions the note is still written and
  loads on your next `/clear`, but the automatic clear does not happen.
- `python3` on your `PATH`.

## Use

```text
/next:go
/next:go rerun the failing e2e with the new fixture
```

The optional text sets the next task.

## How it works

- The skill writes the note to `~/.claude/next/<key>.md` (one note per
  launch directory) and arms it with a 60-minute expiry.
- When the turn ends, the plugin runs `/clear` and then sends a prompt
  that starts with "Starting from a new session".
- A `SessionStart` hook injects the note into the new session and moves
  it to `<key>.consumed.md`, so it fires only once.

Nothing is written inside your repository.

## Feedback

Open an issue in this repository.
