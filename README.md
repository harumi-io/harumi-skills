# harumi-skills

Harumi's [Claude Code](https://code.claude.com/docs) **plugin marketplace** — skills for working with the Harumi platform.

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add harumi-io/harumi-skills
/plugin install harumi-cli-plugin@harumi-skills
```

If the install summary says `Run /reload-plugins to activate.`, run that.

To update later:

```
/plugin marketplace update harumi-skills
```

## Install the `harumi` CLI

With the plugin installed, ask Claude to set the CLI up for you — the `harumi-cli-setup` skill walks the whole path from nothing installed to a verified, logged-in CLI:

```
Use the harumi-cli-setup skill to install and configure the harumi CLI on this machine.
```

Or invoke the skill directly:

```
/harumi-cli-plugin:harumi-cli-setup install the CLI and get me logged in
```

Claude will pick an install method (pipx or pip), confirm you got the real CLI rather than another tool with the same name, then hand the login step to you — `harumi login` emails a one-time code, so it needs you at the keyboard. Tip: type `! harumi login` in the Claude Code prompt so the output lands in the conversation.

Phrasings like "install the harumi cli", "harumi: command not found", or "my harumi login isn't working" trigger the skill on their own — you don't have to name it.

## Plugins

### `harumi-cli-plugin`

Two skills for the [`harumi` CLI](https://github.com/harumi-io/harumi-cli) — run local optimization/solver code (Gurobi, OR-Tools, plain Python) on Harumi's infrastructure.

| Skill | Use when |
|---|---|
| `harumi-cli-plugin:harumi-cli-setup` | The CLI isn't installed or logged in — install/upgrade, authenticate, pick the backend environment, set the org, diagnose a broken install. |
| `harumi-cli-plugin:harumi-cli` | Actually using the CLI — running code, managing projects, repos, runs, datasources, schedules, secrets, and organizations. |

Both are model-invoked: Claude picks them up automatically from the task context. You can also invoke them directly as slash commands.

## Repository layout

```
harumi-skills/
├── .claude-plugin/
│   └── marketplace.json          # the marketplace catalog
└── plugins/
    └── harumi-cli-plugin/
        ├── .claude-plugin/
        │   └── plugin.json       # plugin manifest
        └── skills/
            ├── harumi-cli/
            │   ├── SKILL.md
            │   └── references/
            │       └── commands.md   # full flag-by-flag reference
            └── harumi-cli-setup/
                └── SKILL.md
```

Each plugin's `source` in `marketplace.json` is a full path relative to the repo root (`./plugins/harumi-cli-plugin`). A `./`-prefixed source always resolves from the marketplace root, so it ignores `metadata.pluginRoot` — set one or the other, not both.

## Local development

Load a plugin without installing it:

```bash
claude --plugin-dir ./plugins/harumi-cli-plugin
```

Or add the marketplace from a local path:

```
/plugin marketplace add ./
```

After editing a skill, run `/reload-plugins` to pick up the change without restarting. The reported skills count only covers `commands/` directories, so it may say `0 skills` even when a `skills/` change reloaded fine.

Validate before pushing:

```bash
claude plugin validate ./plugins/harumi-cli-plugin
```

## Adding a plugin to this marketplace

1. Create `plugins/<your-plugin>/.claude-plugin/plugin.json` with at least `name` and `description`.
2. Add skills under `plugins/<your-plugin>/skills/<skill-name>/SKILL.md`. Keep `commands/`, `agents/`, `hooks/`, and `skills/` at the **plugin root** — only `plugin.json` belongs inside `.claude-plugin/`.
3. Append an entry to the `plugins` array in `.claude-plugin/marketplace.json` with `name` and `source` — e.g. `"./plugins/<your-plugin>"`, relative to the repo root.
4. Bump `version` in both the plugin entry and `plugin.json` — users only get updates when it changes.
5. Run `claude plugin validate` and test with `--plugin-dir`.

## Links

- [Harumi CLI source](https://github.com/harumi-io/harumi-cli) · [CLI docs](https://docs.harumi.io/docs/cli)
- [Claude Code plugins](https://code.claude.com/docs/en/plugins) · [plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) · [reference](https://code.claude.com/docs/en/plugins-reference)

## License

Apache-2.0. See [LICENSE](LICENSE).
