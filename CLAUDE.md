# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A one-plugin Claude Code marketplace (`pablofontanilla`) that ships the
`next` plugin. `/next:go` writes a handoff note, clears the context, and
resumes in a fresh session from the note. The user does not confirm
anything. Users install with
`/plugin marketplace add pablofontanilla/claude-next` and
`/plugin install next@pablofontanilla`.

The plugin came from `plugins/handoff` and a `handoff-go` dev mod in
`openshift-eng/edge-tooling`. The two copies are not synced.

## Commands

All paths are relative to the repo root.

```bash
python3 plugins/next/tests/test_handoff.py                 # script tests (stdlib unittest)
python3 plugins/next/tests/test_handoff.py TestRead        # one class
python3 plugins/next/tests/test_handoff.py TestRead.<name> # one test
claude plugin test plugins/next                            # mod tests (tests/*.test.ts)
claude plugin validate plugins/next                        # manifest, hooks, $.state contract
claude plugin validate .                                   # marketplace
claude --plugin-dir plugins/next                           # run the working copy
```

`tsc -p plugins/next` works only after Claude Code has loaded the plugin
from this folder once (`--plugin-dir`). That load writes the API types
to `plugins/next/.claude-plugin/types/`, which is gitignored.

## How it works

Three parts work together. A change to one usually needs a change to
another.

1. **Skill** `skills/go/SKILL.md`: the model writes the note to the path
   that `handoff.py path` gives, then runs `handoff.py arm`.
2. **Mod** `hooks/register.ts` (function hooks, listed under `modules` in
   `hooks/hooks.json`): a `command.run` hook on `next:go` sets the phase
   to `running`. A `tool.call` hook on Bash sets `armed` when the `arm`
   command returns `"status": "ok"`. At `turn.complete`, if armed, the
   mod runs `/clear` and then submits the `RESUME` prompt.
3. **Script** `scripts/handoff.py`, run by the `SessionStart` command
   hook (`startup|clear`, in the same `hooks/hooks.json`): `read` injects
   the live note as `additionalContext` and moves it to
   `<key>.consumed.md`.

Contracts that break silently if they drift:

- The mod's `ARM` regex must match the `arm` command string in SKILL.md.
- The mod's `SKILL` constant (`next:go`) must match the plugin name and
  the skill folder name.
- `note_path()` is the only place that computes a note's location. The
  skill and the hook must agree on it, or the note never fires.
- The skill passes `--project-dir "${CLAUDE_PROJECT_DIR}"` because the
  Bash tool does not export that variable. Only hook subprocesses get it
  as a real environment variable.
- `HOOK_CONTEXT_LIMIT` (10,000 characters) is a Claude Code limit.
  Larger `additionalContext` becomes a file-path preview, so `arm`
  refuses a note that does not fit.
- The `RESUME` prompt starts with "Starting from a new session". Users
  rely on that line to see that a clear happened.

## Design intentions

- **Minimum friction.** `/next:go` always clears and resumes. There is no
  confirmation step and no flag. Do not add a "wait for the user" step to
  the skill or to `build_context()`.
- **`/next:later` is reserved, not built.** It will arm without a clear
  and accept a resume date. `handoff.py arm --until` already supports
  this. Do not make `/next:go` read timeframes from its arguments.
- **No dependencies outside Claude Code and `python3`.** `handoff.py` is
  stdlib-only because plugins cannot install Python packages. The
  frontmatter parser is hand-written for this reason; do not add YAML.
  Moving the script logic into the TypeScript mod (to remove `python3`)
  was discussed and not done.
- **Nothing in the user's repo.** Notes live in `~/.claude/next/`
  (`HANDOFF_DIR` overrides this). This folder is separate from the
  edge-tooling plugin's `~/.claude/handoffs/`, so both plugins can be
  installed. Never write to the repo or edit its CLAUDE.md.
- **One note per launch directory, fired once.** The key is the resolved
  launch directory (each git worktree has its own). Any `read` retires
  the note, live or expired, so a second `/clear` cannot fire it again.
  `--resume` and `--continue` do not consume it.
- **Lossy on purpose.** The note holds only what CLAUDE.md and the repo
  do not: state, decisions, dead ends, next task, at most 60 lines. A
  fresh context with a short note is the intended trade against a
  directed `/compact`. It works best at task boundaries, and badly in
  the middle of a debug session whose evidence lives only in the
  conversation.
- **Expiry guards against stale notes.** The default TTL is 60 minutes
  and the maximum is 7 days. A note that fires into an unrelated session
  is the risk.
- **The hooks never fail loudly.** `read` swallows all errors, and the
  mod shows a toast and keeps the session when the arm did not succeed
  or the turn was interrupted.

## Releasing

Installed copies come from git, so users get a change only after it is
pushed and they run `claude plugin update`. Change `version` in both
`plugins/next/.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json`.

The function-hooks API is early access and changes between Claude Code
releases. It was tested on 2.1.289. On a version without function hooks,
the note still loads on the next manual `/clear`, but the automatic clear
does not happen.
