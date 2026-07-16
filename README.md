# svnscha/claude-plugins

My personal [Claude Code](https://code.claude.com) plugin marketplace. One repo,
many plugins — add the marketplace once, then install whichever plugins you want.

## Plugins

| Plugin                                     | What it gives you                                                                 |
| :----------------------------------------- | :-------------------------------------------------------------------------------- |
| [`github-utils`](./plugins/github-utils)   | GitHub workflow skills — e.g. consolidate several Dependabot PRs into one PR.      |
| [`claude-utils`](./plugins/claude-utils)   | Onboard/sync a device's `~/.claude` config from versioned templates in this repo. |
| [`sysadmin-utils`](./plugins/sysadmin-utils) | Windows sysadmin skills — e.g. extend C: past a blocking recovery partition.     |

## Get started

Add the marketplace:

```shell
/plugin marketplace add svnscha/claude-plugins
```

Install the plugins you want, then reload:

```shell
/plugin install github-utils@svnscha
/plugin install claude-utils@svnscha
/plugin install sysadmin-utils@svnscha
/reload-plugins
```

Run a skill — plugin skills are namespaced `plugin:skill`:

```shell
/github-utils:consolidate-dependabot
/claude-utils:onboard
/sysadmin-utils:extend-volume
```

See each plugin's README (linked above) for what its skills do and how to use them.

## Stay up to date

```shell
/plugin marketplace update svnscha
/reload-plugins
```
