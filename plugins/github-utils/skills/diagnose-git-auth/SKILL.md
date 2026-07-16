---
name: diagnose-git-auth
description: >-
  Diagnose why Git authentication fails over HTTPS and propose a fix. Resolves
  which credential helper actually wins for a host, tests each candidate in
  isolation, and identifies the broken one. Use when the user hits
  "Authentication failed", "Invalid username or token", "Password authentication
  is not supported", "Support for password authentication was removed", is
  prompted for a password on every push, or reports that a GUI client (Fork,
  GitHub Desktop, SourceTree, VS Code) can push but the CLI cannot.
---

# Diagnose Git authentication

Find out *which credential the failing Git command actually used*, prove which
stored credential is healthy, then propose a fix. The user approves before
anything is written.

`$ARGUMENTS` may contain the remote URL, host, or repo path to investigate. If
empty, work in the user's current repository against `origin`.

Steps 1–4 are strictly read-only. Do not change config, re-authenticate, or
erase credentials before presenting the diagnosis in step 5.

## 1. Scope the failure

Identify the host, because credential config is per-host and per-URL:

```bash
git remote -v
git config --get remote.origin.url
```

If the remote is `git@…` / `ssh://`, this skill does not apply — that is SSH key
auth, not credential helpers. Say so and stop.

Get the exact error if the user has not pasted one. Ask them to re-run the
failing command with `--verbose` rather than guessing.

## 2. Collect evidence

Run these together; each answers a distinct question.

```bash
# Which helpers are configured, and from which file?
git config --list --show-origin | grep -i -E 'credential|http\.|url\.'

# Is a token being injected by the environment? (beats every helper)
env | grep -i -E 'GH_TOKEN|GITHUB_TOKEN|GIT_ASKPASS|GCM_|GIT_CONFIG'

# Which binaries are in play?
git --version && which git git-credential-manager gh
```

Then list stored credentials for the host, per platform:

| Platform | Command |
| :------- | :------ |
| Windows  | `cmdkey /list \| grep -i -C 3 github` |
| macOS    | `security find-internet-password -s github.com -g` |
| Linux    | `secret-tool search server github.com` |

Expect to find **several independent credentials for the same host** — GCM's,
`gh`'s, and one per GUI client. That is normal and is the usual root cause: each
tool authenticates separately, so one can rot while the others stay healthy.

If `gh` appears anywhere in the config, check it directly — it is the most
common broken link:

```bash
gh auth status
```

## 3. Resolve which helper actually wins

This is the crux. Git does not use "the" helper; it builds an **ordered list**
and takes the first helper that returns a credential — even a bad one. Two rules
decide the list:

- **An empty `helper =` resets the list**, discarding everything configured
  earlier (including system-wide defaults).
- **URL-scoped config is additive**, appended after global `credential.helper`.

So this — exactly what `gh auth setup-git` writes — silently disables GCM for
GitHub and routes Git to `gh`:

```ini
[credential "https://github.com"]
	helper =                                          # wipes credential.helper=manager
	helper = !'.../gh.exe' auth git-credential        # gh is now the only helper
```

A healthy GCM credential can sit right next to it, unused. Confirm the live
resolution order rather than reading it off:

```bash
GIT_TRACE=1 git ls-remote origin HEAD 2>&1 | grep -i credential
```

## 4. Test each candidate helper in isolation

Prove which stored credential works. Reset the URL-scoped list with an empty
value, then append exactly one helper — the `-c` flags come last, so they win:

```bash
GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=never git \
  -c credential.https://github.com.helper= \
  -c credential.https://github.com.helper=manager \
  ls-remote https://github.com/<owner>/<repo>.git HEAD
```

Substitute the host and each candidate helper (`manager`, `osxkeychain`,
`libsecret`, `store`, or a `!` command such as the `gh` one). `ls-remote` is a
read-only reachability check. `GIT_TERMINAL_PROMPT=0` and `GCM_INTERACTIVE=never`
force a clean failure instead of an interactive prompt that would hang.

A helper that returns a ref is healthy. Note which pass and which fail — the fix
is almost always "route Git at the one that passes".

To see what a helper hands over without a network call:

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
```

This prints a live token. Never echo the value into your reply — report only the
username and whether a credential came back.

## 5. Diagnose

Match the evidence against the common patterns:

| Symptom | Likely cause | Fix |
| :------ | :----------- | :-- |
| `Invalid username or token` + a `gh` helper in `~/.gitconfig` | Git routed to `gh`, whose token expired | Remove the `gh` blocks; fall back to GCM |
| GUI client pushes fine, CLI fails | Different credential stores; the CLI's has rotted | Point the CLI at a healthy helper |
| Prompted for password every push | No helper resolved, or helper list reset to empty | Configure a helper for the host |
| `Password authentication is not supported` with a real password | Password used where a token is required | Use a helper or a PAT |
| Works, but as the wrong account | Stale entry, or a second account in the store | Erase that host's credential and re-auth |
| Fails only in one shell | `GH_TOKEN` / `GITHUB_TOKEN` set in that shell's profile | Unset it or fix its value |
| Azure DevOps, multiple repos, one works | Missing `credential.useHttpPath` | `git config --global credential.https://dev.azure.com.usehttppath true` |

Present to the user, before proposing anything:

1. **What broke** — the helper that answered and why its credential is invalid.
2. **What is healthy** — which credential passed step 4, with the evidence.
3. **Why the GUI client still works**, if relevant — it uses its own store.

## 6. Propose the fix

Offer the options that the evidence supports, with the tradeoff stated, and let
the user choose. Do not assume that the tidiest option is the one they want —
someone may depend on `gh`'s helper for org SSO or multi-account routing.

Typical options:

- **Fall back to the platform helper** — remove the URL-scoped override so the
  host resolves to `credential.helper=manager` (or `osxkeychain` / `libsecret`).
  Best when step 4 shows that credential healthy; it auto-refreshes its OAuth
  token, so it does not rot.
- **Re-authenticate the broken helper** — `gh auth login -h github.com`. Keeps
  the current wiring; recurs if the token keeps expiring.
- **Erase a stale credential and re-auth** — `git credential reject`, or
  `cmdkey /delete:<target>` / `security delete-internet-password`.
- **Long-lived PAT** — for a helper whose OAuth token will not stay alive.

Ask which they want. Then apply only that.

## 7. Apply and verify

Back up before editing, and tell the user the path:

```bash
cp ~/.gitconfig ~/.gitconfig.bak
```

Make the edit, then verify with no `-c` overrides — the point is that the
*default* path now works:

```bash
git config --list --show-origin | grep -i credential
GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=never git ls-remote origin HEAD
```

Report: what changed, where the backup is, and the passing verification. If the
user has an unpushed branch, suggest they retry the real `git push` — do not
push on their behalf.

## Notes

- `gh auth setup-git` re-creates the `gh` helper blocks. If the fix was to remove
  them, tell the user not to run it again, or it silently reverts.
- Removing a *credential helper* never deletes a *stored credential*. Falling
  back to GCM reuses the token already in the OS keychain — no re-login needed.
- `gh`'s helper and GCM can coexist on one host only if `gh` stays authenticated;
  whichever is listed first answers, so a broken first helper breaks Git even
  when the second is healthy.
- The empty-`helper` reset also appears in corporate `/etc/gitconfig` and in
  `GIT_CONFIG_GLOBAL` overrides. If resolution looks impossible, re-check
  `--show-origin` for a file you did not expect.
- Some hosts key credentials by path (Azure DevOps, self-hosted GitLab
  subgroups). If one repo authenticates and a sibling does not, suspect
  `credential.useHttpPath` before suspecting the token.
