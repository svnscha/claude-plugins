# claude-utils

Onboard and keep a device's personal Claude Code configuration (`~/.claude`) in
sync with versioned templates hosted in this repo.

Part of the [`svnscha`](../../README.md) marketplace.

## Install

```shell
/plugin marketplace add svnscha/claude-plugins
/plugin install claude-utils@svnscha
/reload-plugins
```

## Skills

### `/claude-utils:onboard`

Syncs `~/.claude` from the templates bundled in this plugin
(`templates/`). It:

1. Enumerates every file under `templates/` (e.g. `CLAUDE.md`).
2. Compares each against the device's `~/.claude/<same path>`.
3. For a missing target: shows the content and asks before creating it.
4. For an identical target: reports "already in sync", does nothing.
5. For a differing target: shows a unified diff, then lets you **Merge**
   (section-aware for Markdown, with the proposed result shown before writing),
   **Overwrite** (backs up the old file first), **Keep existing**, or **Cancel**.

Backups are written next to the target as `<file>.bak-<timestamp>`. The skill
only ever touches `templates/` and `~/.claude/`, and never overwrites without an
explicit choice.

```shell
/claude-utils:onboard
```

## Templates

The canonical config lives in [`templates/`](./templates). Edit it here in the
repo — not per device — then run `/claude-utils:onboard` on each machine to sync.

| Template            | Onboards to           |
| :------------------ | :-------------------- |
| `templates/CLAUDE.md` | `~/.claude/CLAUDE.md` |

To manage more of `~/.claude`, drop additional files under `templates/` at the
relative path they should take (e.g. `templates/settings.json`); the `onboard`
skill picks them up automatically.
