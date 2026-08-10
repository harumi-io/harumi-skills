---
name: harumi-cli
description: Drive the `harumi` CLI to run local optimization/solver code (Gurobi, OR-Tools, plain Python) on Harumi's infrastructure via the project's Harumi Git (Gitea) repo — and to create/manage projects, import a downloaded project export, browse and edit the project's repo, inspect/cancel runs and download outputs, manage datasources (including read-only SQL queries), cron schedules, secrets, organizations, and your profile. Use when the user wants to run, push, or debug a local script against a Harumi project, mentions `harumi run`, `harumi init`, `harumi import`, `harumi runs`, `harumi outputs`, `harumi repo`, `harumi projects`, `harumi specs`, `harumi datasources`, `harumi schedules`, `harumi secrets`, `harumi org`, `harumi profile`, `harumi env`, asks about Harumi kernel specs, scratch branches, promoting a version to live, querying a project datasource, scheduling recurring runs, or fetching results/files from a Harumi run or repo. If the CLI is not installed or not yet logged in, use `harumi-cli-setup` first.
---

# Harumi CLI

Drives the `harumi` CLI (PyPI package `harumi`, source repo `harumi-cli`). Every run is **git-ref based**: code lives in a per-project Harumi Git (Gitea) repo, and the CLI auto-manages a throwaway scratch branch so users can iterate without committing manually.

For the full flag-by-flag reference, API endpoints, config file layout, troubleshooting table, and the Python `Client` SDK, see [references/commands.md](references/commands.md).

For installing the CLI, first-time login, environment selection, and install-level troubleshooting, use the **`harumi-cli-setup`** skill instead.

## Preflight

1. `harumi whoami` — the single best health check, since it needs both the real binary and a valid session. If it prints an email and environment, you're good; skip to the workflow.
2. If it fails, hand off to **`harumi-cli-setup`** rather than debugging inline. Two distinct failures land here: `Not logged in. Run harumi login first.` (installed, no session) and anything suggesting the wrong binary — a version string that doesn't look like `harumi <x.y.z>`, or unrecognized subcommands, which usually means another tool named `harumi` is shadowing it on PATH. Don't try to automate `harumi login`; it emails a one-time code and needs the user at the keyboard.
3. `harumi env current` — worth checking before any write. `staging` is internal and VPN-only, so off-VPN work there fails with network errors; more importantly, it's easy to mutate the wrong backend when the saved default isn't what the user assumes.

Environment caveat: the git-first `run`/`repo` flow currently works on `staging`; production Gitea (`git.harumi.io`) is not live yet, so on `production` expect only the non-git commands (`projects`, `datasources`, `schedules`, `secrets`, `org`, `profile`, `specs`) to work. Verify with `harumi repo ls` rather than assuming.

## The always-bound-project invariant

Every `harumi run` — and every command that accepts `--project` — needs **either** an explicit `--project <ID>` **or** a `.harumi/config.json` binding in the working directory or a parent. Without either it exits with:

```
Provide --project or run from a directory with a .harumi binding (see `harumi init`).
```

The binding file is searched **upward** from cwd, so bound commands work from subdirectories.

## Global flags

These sit before the subcommand:

```bash
harumi --version
harumi --env staging run        # target an environment for this command only
```

Most subcommands additionally accept `--api-url`, `--git-url` (where relevant), and `--org` to override the resolved config for that one invocation.

## Workflow

### 1. Bind a directory to a project

Once per project directory:

```bash
harumi init --project <PROJECT_ID>
```

Fetches the repo metadata (`GET /projects/{id}/repo`), writes `.harumi/config.json`, and configures the `harumi` git remote for authenticated HTTPS pushes.

Find project ids with `harumi projects list` (or `harumi notebooks` for the legacy notebook-centric view).

**Create a new project** instead of binding an existing one:

```bash
harumi projects create "My Project" [--customer-id ID] [--template-id ID] [--no-bind]
```

Creates the project (which provisions its Gitea repo server-side), then binds the current directory the same way `init` does. If the backend didn't provision a repo, the project is still created and the CLI warns instead of failing.

**Import a downloaded project export** (from the web app's "Download project"):

```bash
harumi import [PATH] [--project-name NAME] [--from-git URL] [--no-bind]
```

`PATH` must be an **unzipped folder** (default: cwd) — unzip first, or it fails with `Not a directory: <path>`. Creates a project, pushes the whole folder as one initial commit, and binds the directory. `--from-git URL` also clones an old GitHub repo flat into the folder first (exported files win on collisions). Check for a `HARUMI_IMPORT.md` in the export for follow-ups like re-adding datasource credentials.

### 2. Run code

**`harumi run` takes no file path.** It executes whatever the project's *repo* is configured to run at the given git ref — the entrypoint comes from a `harumi.toml` `command` field committed in the repo, which the backend reads. The CLI never parses that file itself, so don't expect a local `harumi.toml` to change anything on its own; it has to be committed and pushed.

This trips people up constantly, because "run my script" naturally suggests `harumi run solver.py`. Two ways to actually target a specific file:

```bash
harumi run --command "python solver.py"                 # override for this run only
harumi run --command "python demos/project_demo/main.py"  # multi-file: cwd is the repo root, so relative imports work
```

Or commit a `harumi.toml` pointing `command` at it, which makes it the project default. If a run executes the wrong thing, check `harumi runs get <RUN_ID>` — it reports the `command` that actually ran.

Pick a kernel size if needed — `harumi specs` lists `name`, `display_name`, `cpu`, `memory`, `subscription_required`. The `name` is what `--kernel` takes.

**Default — scratch branch, for uncommitted/unpushed work:**

```bash
harumi run
```

The CLI detects a dirty or unpushed tree, commits the full working tree to a throwaway branch `harumi-scratch/<user>/<yyyymmdd-HHMMSS>` using a temporary git index (**the user's real index, HEAD, and branches are untouched**), pushes it, queues the run against that ref, and deletes the remote scratch branch when finished.

If the tree is clean and fully pushed, it runs the current branch directly — no scratch branch.

**Specific ref, command, or kernel:**

```bash
harumi run --branch feature/solver-v2
harumi run --commit abc123f
harumi run --command "python solver.py" --kernel gurobi_python_medium
```

`--branch` and `--commit` skip the scratch-branch path entirely and run the named ref as-is — so local edits are ignored, which is what you want for reproducing a past result and *not* what you want mid-iteration.

**Block until done and download artifacts:**

```bash
harumi run --watch --output-dir ./out
```

`--output-dir` requires `--watch`. Without `--watch`, the CLI prints the run id and returns immediately; `harumi run --watch` exits **non-zero** if the run ends in a failed status.

### 3. Inspect and manage runs

```bash
harumi runs list                # runs for the bound project, newest first
harumi runs get <RUN_ID>        # status, git ref, exit code, stdout/stderr/error
harumi runs cancel <RUN_ID>     # cancel an in-flight run
```

`harumi runs get` is the first thing to reach for when a run fails — it carries the captured stdout/stderr and error.

`harumi outputs` is a deprecated backwards-compatible wrapper (`--latest`, `--download <RUN_ID>`, `--output-dir`); prefer `harumi runs` and `harumi run --output-dir`.

## Manage the project's repo directly

`harumi repo` reads and writes the project's Gitea repo through harumi-api — no local clone needed for file-level edits. Every write lands as exactly **one commit** via the batch changes endpoint.

```bash
harumi repo ls [--ref REF]                              # every file, flat + recursive
harumi repo cat <path> [--ref REF] [--output FILE]      # print, or save bytes
harumi repo put <local_file> <repo_path> [-m MSG] [--branch B]
harumi repo rm <path> [-m MSG] [--branch B] [--yes]     # file, or everything under a folder
harumi repo mv <from> <to> [-m MSG] [--branch B]
harumi repo download -o out.zip [--path DIR] [--ref REF]
harumi repo branches                                    # versions; live branch flagged
harumi repo branch-create <name> [--from BRANCH]
harumi repo branch-rm <name> [--yes]
harumi repo promote <name> [--title T] [--delete-after] # merge a version into live
```

Notes:

- **Branches are "versions"** in Harumi's product vocabulary, and one is the **live** branch. `branch-rm` will not delete the live branch.
- `repo cat` refuses to print non-UTF-8 content — use `--output <path>` for binary files.
- `repo put` auto-detects create vs update by probing the file first.
- `repo promote` merges into live. On a merge conflict the CLI **fails with the conflict message** rather than reporting a fake success.
- `rm`, `branch-rm` prompt for confirmation unless `--yes`. Don't pass `--yes` on the user's behalf for destructive operations unless they asked for it.

## Manage datasources

Project-scoped database connections. **Credentials are always prompted interactively with hidden input** — never a flag, so they can't leak into shell history or process listings — and the server (which stores them in AWS SSM) never returns them.

```bash
harumi datasources list
harumi datasources get <name>                    # detail view, no credentials
harumi datasources add <name> --type postgresql --host H --port 5432 --database D --username U
harumi datasources update <name> --host newhost [--set-credentials] [--name NEW_NAME]
harumi datasources remove <name> [--yes]
harumi datasources test --type postgresql --host H --port 5432 --database D --username U
harumi datasources query <name> --sql "SELECT * FROM orders LIMIT 10" [--limit N] [--csv PATH]
```

- `--type` is one of `postgresql | mysql | sqlserver | oracle`.
- `add` is tested server-side before it persists, so bad credentials fail the `add`.
- `test` validates a connection **without persisting** — use it to sanity-check before `add`.
- `query` is the workhorse for iteration: a server-side read-only proxy that allows **`SELECT`/`WITH` only** (any destructive keyword → 403 naming the keyword) and caps rows (`--limit`, default 10000, server hard max 100000). Use it to validate SQL before hardcoding it into solver code. `--csv <path>` saves instead of printing.
- Proxy variants: `--use-proxy`, `--proxy-host`, `--proxy-port`, `--proxy-server-name` route traffic via the mTLS proxy.

## Schedule recurring runs

```bash
harumi schedules list
harumi schedules get <SCHEDULE_ID>
harumi schedules add --cron "0 9 * * *" --git-branch main [--start-at ISO] [--git-commit SHA] \
                     [--command C] [--kernel K] [--output-format F] [--email-to E]
harumi schedules update <SCHEDULE_ID> --cron "0 */6 * * *"
harumi schedules remove <SCHEDULE_ID> [--yes]
```

Key semantics:

- **`--cron` is a raw 5-field expression interpreted in UTC.** The CLI does no client-side validation; the server validates with `croniter` and returns a clear `400 Invalid cron expression`. Don't build a calendar UX — pass the string.
- **There is no pause/enable flag.** `harumi schedules remove` is the only way to stop a schedule firing.
- **There is no "run now" for a schedule** — immediate execution is `harumi run`.
- `--git-branch` defaults to `main`; `--start-at` defaults to now (UTC).
- `--email-to` takes `only-me` | `team` | `everyone` | a comma-separated list of addresses.

## Manage secrets

Project-scoped environment variables, stored as SSM SecureStrings and injected into kernels/apps at run time.

```bash
harumi secrets list          # names only — values are never printed
harumi secrets set <NAME>    # prompts for the value (hidden input)
harumi secrets rm <NAME> [--yes]
```

There is no update endpoint: `set` on an existing name **overwrites** it.

## Organizations and profile

```bash
harumi org list
harumi org create <BUSINESS_NAME>
harumi org rename <ORG_ID> <NEW_NAME>
harumi org delete <ORG_ID> [--yes]
harumi org members <ORG_ID>
harumi org invite <ORG_ID> --email a@b.com --role member
harumi org role <ORG_ID> <USER_ID> --role admin
harumi org remove <ORG_ID> <USER_ID> [--yes]

harumi profile show
harumi profile set --first-name Ana --last-name Silva --bio "..."
```

`--role` is one of `owner | admin | member | viewer`. `profile set` and every `update` command send only the flags you pass, and error locally with `No fields to update.` if you pass none.

`harumi config set-org <ORG_ID>` persists the org sent as `X-Organization` (scoped per environment); `--org` overrides it per command.

## Working safely

The CLI's design encodes most of these already — the useful thing is understanding *why*, so you make the right call in a situation this skill didn't anticipate.

**Secrets never travel as flags.** There is deliberately no `--password` or `--value` anywhere: `datasources add`, `datasources test`, `datasources update --set-credentials`, and `secrets set` all prompt with hidden input. Flags leak into shell history, `ps` output, and CI logs. So if you find yourself wanting a flag that doesn't exist, that's the design working — let the command prompt the user rather than routing the value some other way.

**Leave `harumi login` to the user.** It emails a one-time code, so there's nothing to automate; the same is true of any prompt above.

**Confirmations exist because these operations are unrecoverable.** `projects delete`, `org delete`, `repo rm`, `repo branch-rm`, `datasources remove`, `schedules remove`, and `secrets rm` all prompt, and `projects delete` deliberately makes you retype the project name. Passing `--yes` on the user's behalf converts a deliberate action into a silent one — only use it when they asked for exactly that deletion. `repo rm` on a folder path deletes *everything* beneath it, so confirm the path before running it, not after.

**Check the environment before writes.** Each environment has its own separate session and data, and the active one comes from a saved default the user may have forgotten. `harumi env current` costs nothing next to mutating the wrong backend.

**Show platform links, not Gitea URLs.** Report `https://platform.harumi.io/projects/<id>` (the CLI prints this itself) rather than a `git.harumi.io` URL — Gitea is internal plumbing, and a link into it isn't something the user can act on.

## Common failures

| Error | Fix |
|---|---|
| `Not logged in. Run harumi login first.` | Hand off to `harumi-cli-setup` |
| `harumi` runs but subcommands are unrecognized | Another tool named `harumi` is shadowing it — hand off to `harumi-cli-setup` |
| `Provide --project or run from a directory with a .harumi binding` | `harumi init --project <ID>`, or pass `--project` |
| `No Gitea token found. Run harumi login` | `harumi login` again to re-provision |
| `git push failed: ...` | VPN not connected, or stale token — check VPN, re-run `harumi login` |
| `Run ended with status: failed` | `harumi runs get <RUN_ID>` for stdout/stderr/error |
| `HTTP 403: Only SELECT queries are allowed ...` | Rewrite the `datasources query` SQL as read-only SELECT/WITH |
| `HTTP 400: Invalid cron expression` | Fix the 5-field cron string (minute hour day month weekday, UTC) |
| `<path> is not valid UTF-8 text.` | `repo cat --output <local_path>` for binary files |
| `Result was truncated at the server-side row cap` | Narrow the query or raise `--limit` |

The full troubleshooting table is in [references/commands.md](references/commands.md#troubleshooting).

## Scripting alternative

For anything loop-heavy or multi-step, the Python library is often cleaner than shelling out:

```python
from harumi import Client
from harumi.config import ProjectBinding

binding = ProjectBinding.load()   # reads .harumi/config.json (cwd or a parent)
client = Client()                 # loads the stored session

response = client.execute_project(binding.project_id, branch="main")
```

See [references/commands.md](references/commands.md#python-library-client-alternative) for polling, artifact download, repo edits, datasources, schedules, secrets, and orgs.

## When the docs and the CLI disagree

`harumi --help` and `harumi <group> <command> --help` are the authority. If real behavior contradicts this skill, follow the CLI and tell the user the skill is out of date.
