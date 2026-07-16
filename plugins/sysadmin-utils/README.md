# sysadmin-utils

Skills for Windows system administration, driven through PowerShell and `diskpart`.

Part of the [`svnscha`](../../README.md) marketplace.

## Install

```shell
/plugin marketplace add svnscha/claude-plugins
/plugin install sysadmin-utils@svnscha
/reload-plugins
```

## Requirements

- Windows
- An **elevated** Claude Code session (the disk operations need Administrator)

## Skills

### `/sysadmin-utils:extend-volume`

Extend a volume into unallocated space that another partition blocks — the reason
"Extend Volume" is greyed out in Disk Management. Windows can only extend into
space *immediately following* a partition, and the WinRE recovery partition
usually sits in the way. It:

1. Surveys the disk by physical offset, and infers where the unallocated gap is.
2. Identifies the blocker, and stops if it is an OEM restore or data partition
   rather than a recovery partition.
3. Checks WinRE health and BitLocker status.
4. Presents the plan and the resulting size — and requires a snapshot or backup
   before touching the partition table.
5. Stages `winre.wim` back onto C:, and verifies it landed before deleting anything.
6. Deletes the blocker, extends the volume, and recreates recovery at the end of
   the disk where it can never block an extend again.
7. Re-enables WinRE and verifies the registration actually took — status, location,
   *and* version, since a failed re-register still reports "Enabled".

Steps 1–3 are read-only; nothing is deleted or resized without your approval.

Usage:

```shell
/sysadmin-utils:extend-volume        # extend C:
/sysadmin-utils:extend-volume D      # extend a specific drive
```

## Roadmap

More Windows sysadmin skills will be added here over time. Each new skill is a
folder under `skills/` with its own `SKILL.md`.
