---
name: consolidate-dependabot
description: >-
  Consolidate one or more open Dependabot pull requests into a single branch and
  a consolidated PR that supersedes them. Creates a branch, merges the Dependabot
  bumps, regenerates the lock file, verifies the build, opens the PR, and closes
  the superseded Dependabot PRs. Use when the user asks to combine, batch, roll
  up, consolidate, or supersede Dependabot / dependency-update PRs.
---

# Consolidate Dependabot PRs

Combine several open Dependabot dependency-bump PRs into one consolidated PR so
CI runs once, review happens once, and the individual Dependabot PRs are closed
automatically.

`$ARGUMENTS` may contain the PRs to include:
- Empty → discover all open Dependabot PRs and confirm the selection with the user.
- A list of numbers (e.g. `142 145 150`) → include exactly those.
- `all` → include every open Dependabot PR without further prompting.

Work in the user's current repository. Use the `gh` CLI and `git`. Never force-push
or touch unrelated branches.

## 1. Preflight

Run these checks and stop with a clear message if any fails:

1. `gh auth status` — must be authenticated.
2. `git status --porcelain` — the working tree must be clean. If dirty, ask the
   user to stash or commit first; do not proceed.
3. Determine the default branch:
   `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`.
4. Make sure the local default branch is current:
   `git checkout <default>` then `git pull --ff-only`.

## 2. Discover the Dependabot PRs

```bash
gh pr list --author "app/dependabot" --state open \
  --json number,title,headRefName,url,labels \
  --limit 100
```

Present the list to the user as a short table (number, title, branch). Then pick
the set to consolidate based on `$ARGUMENTS` (see above). If discovery returns
nothing, report that there are no open Dependabot PRs and stop.

Record the chosen PR numbers — you will need them for the branch name, the PR
body, and the cleanup step.

## 3. Create the consolidation branch

Branch off the up-to-date default branch. Use a dated, descriptive name:

```bash
git checkout -b deps/consolidate-<YYYY-MM-DD>
```

If a branch by that name already exists, append a short suffix rather than reusing it.

## 4. Merge each Dependabot bump

Fetch first so the Dependabot head branches are available locally:

```bash
git fetch origin
```

For each selected PR, in ascending PR-number order, merge its head branch:

```bash
git merge --no-ff origin/<headRefName> -m "Merge Dependabot: <title>"
```

Handle conflicts by type:

- **Manifest files** (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`,
  `*.csproj`, `Gemfile`, `pom.xml`, `build.gradle`, …): keep **both** version
  bumps. Resolve manually so every dependency ends at the higher requested
  version. Never drop a bump from a PR you were asked to include.
- **Lock files** (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`,
  `poetry.lock`, `go.sum`, `Cargo.lock`, `Gemfile.lock`, …): do **not**
  hand-merge these. Take either side to unblock the merge (`git checkout
  --theirs <lockfile>` or `--ours`), stage it, complete the merge, and
  regenerate the lock file cleanly in the next step.

If a merge is hopelessly tangled, abort that one merge (`git merge --abort`),
tell the user which PR conflicted, and continue with the rest — a partial
consolidation is fine and can be noted in the PR body.

## 5. Regenerate the lock file and verify

Detect the ecosystem from the manifest/lock files present and regenerate the
lock file so it matches the merged manifests exactly. Prefer lockfile-only
commands where they exist:

| Ecosystem            | Regenerate lock file                         | Verify                       |
| :------------------- | :------------------------------------------- | :--------------------------- |
| npm                  | `npm install --package-lock-only`            | `npm ci && npm test`         |
| pnpm                 | `pnpm install --lockfile-only`               | `pnpm install && pnpm test`  |
| yarn                 | `yarn install --mode update-lockfile`        | `yarn install && yarn test`  |
| Python (Poetry)      | `poetry lock --no-update` then `poetry lock` | `poetry install`             |
| Python (uv)          | `uv lock`                                     | `uv sync`                    |
| Go                   | `go mod tidy`                                 | `go build ./... && go test ./...` |
| Rust                 | `cargo update` (or `-p <crate>` per bump)    | `cargo build`                |
| Ruby (Bundler)       | `bundle lock`                                | `bundle install`             |

Only run verification steps that the repo actually supports (check for the
script/target first). Report the results. If the build or tests fail because of
the updates, surface the failure to the user and ask how to proceed rather than
opening a broken PR.

Commit the regenerated lock file:

```bash
git add -A
git commit -m "Regenerate lock file for consolidated dependency updates"
```

## 6. Open the consolidated PR

Push and open the PR against the default branch:

```bash
git push -u origin deps/consolidate-<YYYY-MM-DD>
```

Write the body from a heredoc so `Supersedes` lines are on their own lines. List
every included PR so GitHub links them and reviewers see the scope:

```bash
gh pr create --base <default> --title "Consolidate dependency updates (<count> PRs)" --body "$(cat <<'EOF'
Consolidates the following Dependabot PRs into a single update:

- #142 — bump X from a to b
- #145 — bump Y from c to d

Supersedes #142
Supersedes #145

Lock file regenerated and build verified locally.
EOF
)"
```

Report the new PR URL to the user.

## 7. Close the superseded Dependabot PRs

Once the consolidated PR is open, close each superseded Dependabot PR by
commenting the Dependabot command so Dependabot itself closes it and stops
rebasing the branch:

```bash
gh pr comment <number> --body "@dependabot close"
```

Do this for every included PR. Then summarize for the user: the new PR URL, the
list of superseded PRs that were closed, and any PR you skipped due to conflicts.

## Notes

- If the consolidated PR is later closed **without** merging, Dependabot will
  reopen the superseded PRs on its next run — mention this when relevant.
- Keep each dependency at the exact version Dependabot requested; this skill
  consolidates PRs, it does not choose different versions.
- This workflow only reads and writes within the current repo and the selected
  Dependabot branches. It never rewrites history on the default branch.
