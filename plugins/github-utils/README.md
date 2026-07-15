# github-utils

Skills for common GitHub workflows, driven through the `gh` CLI and `git`.

Part of the [`svnscha`](../../README.md) marketplace.

## Install

```shell
/plugin marketplace add svnscha/claude-plugins
/plugin install github-utils@svnscha
/reload-plugins
```

## Requirements

- [`gh`](https://cli.github.com/) authenticated (`gh auth status`)
- `git`

## Skills

### `/github-utils:consolidate-dependabot`

Consolidate one or more open Dependabot PRs into a single PR that supersedes
them. It:

1. Verifies auth and a clean working tree, syncs the default branch.
2. Discovers open Dependabot PRs (`gh pr list --author "app/dependabot"`).
3. Creates a `deps/consolidate-<date>` branch.
4. Merges each selected Dependabot bump, keeping every version bump.
5. Regenerates the lock file for your ecosystem (npm, pnpm, yarn, Poetry, uv,
   Go, Cargo, Bundler) and verifies the build/tests.
6. Opens a consolidated PR with `Supersedes #…` lines.
7. Closes the superseded PRs by commenting `@dependabot close`.

Usage:

```shell
/github-utils:consolidate-dependabot           # discover all, confirm selection
/github-utils:consolidate-dependabot all       # include every open Dependabot PR
/github-utils:consolidate-dependabot 142 145    # include only these PRs
```

## Roadmap

More GitHub workflow skills will be added here over time. Each new skill is a
folder under `skills/` with its own `SKILL.md`.
