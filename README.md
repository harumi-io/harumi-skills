# harumi-skills

Agent skills for the [`harumi` CLI](https://github.com/harumi-io/harumi-cli) — install the skill first, and let it install the CLI for you.

Two skills ship here:

| Skill | Use when |
|---|---|
| `harumi-cli-setup` | The CLI isn't installed or logged in — install/upgrade, authenticate, pick the backend environment, set the org, diagnose a broken install. |
| `harumi-cli` | Actually using the CLI — running code, managing projects, repos, runs, datasources, schedules, secrets, and organizations. |

Both are model-invoked: your agent picks them up automatically from the task context. Once installed, just say something like *"install the harumi CLI"* — the `harumi-cli-setup` skill walks the whole path from nothing installed to a verified, logged-in CLI.

## Install

Pick whichever matches your agent. Every path below installs the **same two skills** from this repo.

### `npx skills` (Cursor, Claude Code, Codex, Copilot, and ~70 more)

```bash
npx skills add harumi-io/harumi-skills          # all agents it detects, project scope
npx skills add harumi-io/harumi-skills -g       # global scope (~/), available in every project
npx skills add harumi-io/harumi-skills --skill harumi-cli-setup   # just one skill
```

See [vercel-labs/skills](https://github.com/vercel-labs/skills) for the full flag reference (`-a/--agent`, `--copy`, etc).

### Claude Code

```text
/plugin marketplace add harumi-io/harumi-skills
/plugin install harumi-cli-plugin@harumi-skills
```

If the install summary says `Run /reload-plugins to activate.`, run that. To update later:

```text
/plugin marketplace update harumi-skills
```

For local development, register directly from a cloned copy:

```text
/plugin marketplace add /path/to/harumi-skills
/plugin install harumi-cli-plugin@harumi-skills
```

### Cursor

**Marketplace-independent (works today):** clone and symlink into Cursor's local plugin directory:

```bash
git clone https://github.com/harumi-io/harumi-skills.git
ln -s "$PWD/harumi-skills" ~/.cursor/plugins/local/harumi-cli-plugin
```

Then **Developer: Reload Window** (or restart Cursor). The plugin's `skills/` shows up under **Customize → Skills**.

Alternatively, manual copy without the plugin wrapper:

```bash
git clone https://github.com/harumi-io/harumi-skills.git
cp -R harumi-skills/skills/* ~/.cursor/skills/     # global
cp -R harumi-skills/skills/* .cursor/skills/        # project-only
```

### Install the CLI itself, then seed the skills from it

Every `harumi` install (pipx/uv/pip) bundles these same two skills and can write them to your agent's skills directory directly — no separate clone needed:

```bash
pipx install harumi
harumi skill install            # auto-detects installed agents, writes to their global skills dir
harumi skill install --project  # writes to ./.agents/skills instead
```

### Manual copy (any agent)

Any `SKILL.md` folder works with any agent that reads the Agent Skills format. Copy the two skill directories from this repo (or from `harumi skill path` if you already have the CLI) into whichever directory your agent reads, e.g.:

```bash
cp -R skills/harumi-cli skills/harumi-cli-setup ~/.claude/skills/
cp -R skills/harumi-cli skills/harumi-cli-setup ~/.codex/skills/
cp -R skills/harumi-cli skills/harumi-cli-setup ~/.agents/skills/
```

## Repository layout

```
harumi-skills/
├── .claude-plugin/
│   ├── marketplace.json          # Claude Code marketplace catalog
│   └── plugin.json               # Claude Code plugin manifest (source: "./")
├── .cursor-plugin/
│   └── plugin.json               # Cursor plugin manifest
└── skills/
    ├── harumi-cli/
    │   ├── SKILL.md
    │   └── references/
    │       ├── commands.md       # full flag-by-flag reference
    │       └── dashboard.md      # dashboard.toml widget reference
    └── harumi-cli-setup/
        └── SKILL.md
```

A root-level `skills/` directory is the layout every installer above auto-discovers (`npx skills`, the Claude Code plugin root, and the Cursor plugin `skills` field) — no manifest needs to list individual skill paths.

## Source of truth

**`skills/` in this repo is a generated mirror, not hand-edited.** The canonical copy lives at [`harumi-cli/.agents/skills/`](https://github.com/harumi-io/harumi-cli/tree/main/.agents/skills) — a GitHub Actions job in that repo's `release.yml` copies it here and bumps the manifest `version` fields on every CLI release. Edit the CLI repo, not this one; a hand-edit here will be overwritten on the next release.

## Local development

Load the plugin without installing it:

```bash
claude --plugin-dir .
```

Or add the marketplace from a local path:

```text
/plugin marketplace add ./
```

After editing a skill, run `/reload-plugins` to pick up the change without restarting.

Validate before pushing:

```bash
claude plugin validate .
npx skills add . --list   # confirms npx-skills discovery still resolves both skills
```

## Links

- [Harumi CLI source](https://github.com/harumi-io/harumi-cli) · [CLI docs](https://docs.harumi.io/docs/cli)
- [Claude Code plugins](https://code.claude.com/docs/en/plugins) · [plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) · [reference](https://code.claude.com/docs/en/plugins-reference)
- [Cursor plugins](https://cursor.com/docs/plugins) · [Cursor skills](https://cursor.com/docs/context/skills)
- [Agent Skills specification](https://agentskills.io) · [vercel-labs/skills](https://github.com/vercel-labs/skills) (`npx skills`)

## License

Apache-2.0. See [LICENSE](LICENSE).
