# svnscha/claude-plugins

My personal [Claude Code](https://code.claude.com) plugin marketplace. One repo,
many plugins — add the marketplace once, then install whichever plugins you want.

## Add the marketplace

```shell
/plugin marketplace add svnscha/claude-plugins
```

## Install a plugin

```shell
/plugin install github-utils@svnscha
```

Then run `/reload-plugins` to activate it. Plugin skills are namespaced, e.g.
`/github-utils:consolidate-dependabot`.

## Plugins

| Plugin                                         | Description                                                                      |
| :--------------------------------------------- | :------------------------------------------------------------------------------- |
| [`github-utils`](./plugins/github-utils)       | Skills for common GitHub workflows (consolidate Dependabot PRs, and growing).    |
| [`claude-utils`](./plugins/claude-utils)       | Onboard/sync a device's `~/.claude` config from versioned templates in this repo.|

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json        # the catalog: name "svnscha", lists every plugin
└── plugins/                    # metadata.pluginRoot — each plugin lives here
    └── github-utils/
        ├── .claude-plugin/
        │   └── plugin.json      # plugin manifest (name = namespace)
        ├── README.md
        └── skills/
            └── <skill-name>/
                └── SKILL.md     # one folder per skill
```

Each plugin entry in the catalog points at its folder with a relative `source`
(e.g. `"./plugins/github-utils"`), resolved from the repo root.

## Adding a new plugin

1. Create `plugins/<new-plugin>/.claude-plugin/plugin.json` with at least a
   `name`.
2. Add skills under `plugins/<new-plugin>/skills/<skill>/SKILL.md`.
3. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`.
4. Validate: `claude plugin validate .`
5. Commit and push. Users refresh with `/plugin marketplace update svnscha`.

## Versioning

Plugins here omit an explicit `version`, so Claude Code uses the git commit SHA —
every pushed commit is treated as a new version and users get updates
automatically. To pin releases instead, add a `version` field to a plugin's
`plugin.json` and bump it on every change. (Setting a `version` and forgetting to
bump it means users never see updates.)

## Local development

Test a plugin without installing it from the marketplace:

```shell
claude --plugin-dir ./plugins/github-utils
```

Iterate, then `/reload-plugins` to pick up changes.
