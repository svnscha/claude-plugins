---
name: onboard
description: >-
  Onboard or sync this device's ~/.claude configuration from the versioned
  templates bundled with the claude-utils plugin. Reads any existing config,
  shows a diff, and lets the user merge, overwrite (with backup), or keep their
  current file. Use when the user wants to set up, onboard, sync, or update their
  personal ~/.claude / CLAUDE.md config on a machine.
---

# Onboard ~/.claude from templates

Sync this device's personal Claude Code configuration from the canonical
templates shipped in the `claude-utils` plugin. Be safe and transparent: never
overwrite anything without showing the user what changes and getting an explicit
choice, and always back up before replacing a file.

## Paths

- **Template source:** `${CLAUDE_PLUGIN_ROOT}/templates/` — this placeholder is
  already resolved to the plugin's absolute install directory when you read this.
- **Target:** the user's home Claude directory, `$HOME/.claude/` (on Windows,
  `%USERPROFILE%\.claude`). Each template file maps to the same relative path
  under the target, e.g. `templates/CLAUDE.md` → `~/.claude/CLAUDE.md`.

Only ever read and write inside `${CLAUDE_PLUGIN_ROOT}/templates/` and
`$HOME/.claude/`. Never touch files elsewhere.

## Workflow

### 1. Enumerate templates

List every file under `${CLAUDE_PLUGIN_ROOT}/templates/` (recurse into
subdirectories). Each one is a config file to onboard. If there are none, tell
the user and stop.

### 2. For each template file, compare against the target

Compute the target path (`$HOME/.claude/<relative path>`) and branch:

- **Target missing** → this is a fresh onboard. Show the user the full content
  that will be created, then ask for confirmation before writing it.
- **Target identical to template** (compare with `diff -q` or by hashing) →
  report "already in sync" and move on. Do nothing.
- **Target exists and differs** → show a clear unified diff of **current
  (target) vs template**, so the user sees exactly what would change, then
  present the options in step 3.

Present the diff readably (use `diff -u <target> <template>` and show it in a
fenced block). Summarize in one line what the change amounts to (e.g. "adds a
new `## Testing` section", "template drops the old X rule").

### 3. Ask how to apply the change

Ask the user to choose per differing file (use the AskUserQuestion tool so it's
a clear pick, not free text):

- **Merge** — combine the two. For Markdown configs like `CLAUDE.md`, merge
  *section by section* (`##` headings): keep sections that exist only on the
  device, add sections that exist only in the template, and for a heading
  present in both with different bodies, show both versions and ask which to
  keep (or to keep both). Assemble the merged result, **show the full proposed
  file**, and only write it after the user approves.
- **Overwrite** — replace the target with the template verbatim.
- **Keep existing** — leave the device file untouched, skip it.
- **Cancel** — stop the whole onboard.

### 4. Back up before any destructive write

Before an **Overwrite** or a **Merge** write, copy the existing target to a
timestamped backup next to it, then write:

```bash
cp "$HOME/.claude/CLAUDE.md" "$HOME/.claude/CLAUDE.md.bak-$(date +%Y%m%d-%H%M%S)"
```

Create parent directories as needed (`mkdir -p`) when writing a fresh file into
a subdirectory that doesn't exist yet. Fresh onboards (target missing) need no
backup.

### 5. Report

Summarize what happened for each file: created, overwritten (with the backup
path), merged (with the backup path), already in sync, or skipped. If you wrote
or changed `~/.claude/CLAUDE.md`, remind the user that user-scope instructions
load at the start of a session, so a new session (or the current one going
forward) picks them up.

## Notes

- The templates are the single source of truth. Encourage the user to edit the
  canonical copy in the `svnscha/claude-plugins` repo and re-run
  `/claude-utils:onboard` on each device, rather than hand-editing per machine.
- Run `/plugin marketplace update svnscha` (then `/reload-plugins`) first if the
  user wants the latest templates before onboarding.
- This skill is idempotent: running it again when everything matches reports
  "already in sync" and changes nothing.
