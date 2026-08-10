---
name: harumi-cli-setup
description: >-
  Install, upgrade, authenticate, and verify the `harumi` CLI (the `harumi` PyPI
  package). Use when `harumi` is not installed or not on PATH ("command not found
  - harumi"), when the user asks to install/upgrade/reinstall/uninstall the Harumi
  CLI, install from source for development, log in for the first time (`harumi
  login`, OTP email code, `--signup`), provision Gitea credentials, choose or
  switch the backend environment (production vs staging), set the organization, or
  diagnose a broken install ("Not logged in", "No Gitea token found", VPN/network
  errors on api.dev.harumi.io, wrong Python/pip, pipx vs venv). Once the CLI is
  installed and authenticated, use the `harumi-cli` skill instead to actually run
  commands.
---

# Harumi CLI — Setup

Gets the `harumi` CLI installed, authenticated, and pointed at the right backend environment. For using the CLI once it works, switch to the `harumi-cli` skill.

The CLI is the `harumi` package on PyPI (`pip install harumi`). It installs one console script, `harumi`, plus the importable `harumi` Python package. Requires **Python ≥ 3.9** and **git** on PATH (git is used for `harumi run`'s scratch-branch push and `harumi import`).

## Order of operations

Run these in order and stop at the first failure — each step depends on the previous one.

1. [Install](#1-install) — get the `harumi` binary on PATH
2. [Pick the environment](#2-pick-the-environment) — production (default) or staging
3. [Log in](#3-log-in) — interactive OTP; **must be run by the user**
4. [Set the organization](#4-set-the-organization) — only if the user belongs to more than one
5. [Verify](#5-verify) — confirm the install end to end

## 1. Install

First check whether it's already there:

```bash
harumi --version
```

Expect exactly `harumi <x.y.z>` — then skip to step 2.

**Don't treat any zero-exit output as success.** `harumi` is a short, generic name, and other tools install a binary or shell shim by the same name. If the output names a different product, or `harumi --help` doesn't list subcommands like `projects` / `datasources` / `run`, you're looking at a **different program** and the real CLI is not installed. Confirm which one you have:

```bash
which -a harumi                                    # every harumi on PATH, in order
python3 -c "import harumi, sys; print(harumi.__version__)"   # the real package, if importable
python3 -m pip show harumi                         # installed? which location?
```

The Python import is the reliable signal: the real CLI ships the importable `harumi` package alongside the console script, so `ModuleNotFoundError: No module named 'harumi'` means it isn't installed no matter what `harumi --version` printed. See [name collisions](#name-collisions) if a foreign `harumi` is shadowing it.

**Recommended — pipx** (isolated venv, `harumi` on PATH, no dependency conflicts with the user's project):

```bash
pipx install harumi
```

**pip into the active environment** (fine inside a project venv/conda env):

```bash
pip install harumi
```

**From source** — for contributing to `harumi-cli` or picking up an unreleased fix. Run from a clone of the `harumi-cli` repo:

```bash
pip install -e .          # editable install
pip install -e ".[dev]"   # plus pytest / pytest-asyncio for running the test suite
```

**Upgrade / reinstall / uninstall:**

```bash
pipx upgrade harumi           # or: pip install --upgrade harumi
pipx reinstall harumi
pipx uninstall harumi         # or: pip uninstall harumi
```

Prefer `python -m pip install ...` when there is any doubt about which `pip` is on PATH — it guarantees the package lands in the same interpreter that will run.

If `pip install harumi` succeeds but `harumi --version` still says command not found, the script directory isn't on PATH. See [Install troubleshooting](#install-troubleshooting).

## 2. Pick the environment

The CLI ships two built-in environments. Each is backed by its **own Supabase**, so each has its own separate login.

| Env | API | Harumi Git | Access |
|---|---|---|---|
| `production` (default) | `https://api.harumi.io/api` | `https://git.harumi.io` | public |
| `staging` | `https://api.dev.harumi.io/api` | `https://git.dev.harumi.io` | internal, **VPN-only** |

```bash
harumi env list            # selectable envs, active one flagged (staging hidden by default)
harumi env list --all      # include internal/VPN-only envs (same as HARUMI_INTERNAL=1)
harumi env current         # active env + its endpoints
harumi env use staging     # persist as the default
harumi --env staging login # or override for a single command
```

Selection precedence: `--env` > `HARUMI_ENV` > saved default (`harumi env use`) > `production`.

Most users need nothing here — `production` is already the default. Only switch to `staging` if the user is an internal dev **on the VPN** with a staging account. Note that `staging` is only hidden from `env list` as a UX convenience; the real gate is the VPN plus a staging Supabase account.

## 3. Log in

```bash
harumi login              # existing account
harumi login --signup     # brand-new email — creates the account first
```

**Never try to automate this.** `harumi login` emails a one-time code and prompts for it interactively (and prompts for the email if `--email` is omitted). Ask the user to run it in their terminal and tell you when it's done. Suggest they type `! harumi login` in the Claude Code prompt so the output lands in the conversation.

Use `--signup` the **first** time a given email logs in. Plain `harumi login` on an unknown email fails with `HTTP 422: Signups not allowed for otp`; the CLI detects this case and tells the user to retry with `--signup`.

Logging in also, best-effort:

- provisions a per-user **Gitea token** (`POST /git/credentials`) needed by `harumi run` and `harumi init` for git-over-HTTPS, and
- resolves the organization (see step 4), and
- configures the `harumi` git remote if the cwd is already a bound project directory.

Log in once **per environment** — switching environments does not log you out of the other.

```bash
harumi logout             # clears the session for the active environment only
```

## 4. Set the organization

If the user belongs to exactly one org, `harumi login` stores it automatically and there is nothing to do. If they belong to several, login prints a table of org ids and you need to pick one:

```bash
harumi org list                    # see the orgs and your role in each
harumi config set-org <ORG_ID>     # persist it for the active environment
harumi --help                      # (org can also be overridden per-command with --org)
```

The resolved org is sent as the `X-Organization` header on every request. It is stored **per environment**.

## 5. Verify

```bash
harumi --version    # prints `harumi <x.y.z>` — not another tool (see step 1)
harumi whoami       # session is valid — prints email, user id, active environment
harumi env current  # confirms which backend you're talking to
harumi specs        # confirms real API reachability + lists kernel sizes
harumi projects list
```

`harumi whoami` and `harumi specs` are the checks that actually prove the install: they require both the real binary and a valid session, so they can't be faked by a same-named tool.

`harumi whoami` failing with `Not logged in. Run harumi login first.` means step 3 didn't complete (or the session expired). `harumi specs` failing with a network/timeout error on a `*.dev.harumi.io` host means the VPN isn't connected.

Setup is done once `whoami` and `specs` both succeed. Hand off to the `harumi-cli` skill for the actual work — the next step there is usually binding a directory to a project with `harumi init --project <PROJECT_ID>`.

## Where things are stored

| Path | Contents |
|---|---|
| `~/.harumi/config.json` | Global; stores only the selected `environment`. |
| `~/.harumi/environments/<env>/credentials.json` | Per-env `access_token`, `refresh_token`, `git_token`, `git_url`, `git_username`, `user_id`, `email`. Written mode `0600`. |
| `~/.harumi/environments/<env>/config.json` | Per-env `org_id`, plus any local `api_url` / `git_url` overrides. |
| `.harumi/config.json` (in the project dir) | `project_id` + repo metadata. Written by `harumi init` / `harumi projects create`; searched **upward** from cwd. |

Override the home directory with `HARUMI_HOME`. Upgrading from a pre-environments install migrates the old flat `~/.harumi/credentials.json` into `production` automatically on first run.

**Environment variables** (all optional, each overrides the corresponding config value):

| Var | Purpose |
|---|---|
| `HARUMI_ENV` | Environment to target (`production` \| `staging`). |
| `HARUMI_API_URL` | Override the harumi-api base URL (e.g. `http://localhost:8000/api` for local dev). |
| `HARUMI_GIT_URL` | Override the Harumi Git (Gitea) base URL. |
| `HARUMI_ORG` | Organization id sent as `X-Organization`. |
| `HARUMI_INTERNAL` | Set to `1` to reveal internal environments in `harumi env list`. |
| `HARUMI_HOME` | Directory for config/credentials (default `~/.harumi`). |
| `HARUMI_PLATFORM_URL` | Override the web-app URL used in printed project links. |

`--api-url` / `--git-url` override endpoints **without** changing which environment you are on — useful for pointing at a locally running harumi-api.

## Name collisions

`harumi` is a generic enough name that another tool may already own it on PATH — a real binary, or a one-line shell shim that execs something else (`exec hermes -p harumi "$@"` is one seen in the wild). This is worth ruling out early because every downstream symptom is confusing: the command exists, exits 0, prints a version, and yet no Harumi subcommand works.

Diagnose by comparing what's on PATH against what Python has:

```bash
which -a harumi          # every match, highest priority first
file "$(which harumi)"   # is it a shim? cat it if it's a short script
python3 -m pip show -f harumi | head    # the real package's console-script location
```

If a foreign `harumi` wins on PATH, pick one:

- **Call the real one by full path** — `python3 -m pip show harumi` reports its location; use that path directly, or `python3 -m harumi.cli ...` which bypasses PATH entirely.
- **Reorder PATH** so the real install's `bin` directory precedes the other tool's.
- **Rename or remove the other tool's shim**, if the user owns it and agrees. Ask first — it belongs to something they installed on purpose.

Tell the user which of these you're doing rather than silently working around it; a shadowed binary they don't know about will bite them again outside this session.

## Install troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `harumi --version` prints a *different* tool's name/version | Another `harumi` on PATH shadows the real one | See [name collisions](#name-collisions) |
| `harumi --version` works but every subcommand is unrecognized | Same as above — wrong `harumi` | See [name collisions](#name-collisions) |
| `ModuleNotFoundError: No module named 'harumi'` | The real package isn't installed (whatever `harumi --version` said) | Install it — step 1 |
| `command not found: harumi` after a successful `pip install` | Script dir not on PATH | `pipx ensurepath` (then restart the shell), or invoke via `python -m harumi.cli`, or add the reported `Scripts`/`bin` dir to PATH |
| `ERROR: Package 'harumi' requires a different Python` | Python < 3.9 | Install on 3.9+; with pipx: `pipx install --python python3.12 harumi` |
| Installed but an old version runs | Multiple installs (pip + pipx, or several venvs) | `which -a harumi` to see them all; uninstall the stale one |
| `error: externally-managed-environment` | System Python (PEP 668, common on Debian/Ubuntu/Homebrew) | Use `pipx install harumi`, or install inside a venv |
| Permission denied writing to site-packages | Installing into system Python | Use pipx or a venv — do **not** `sudo pip install` |
| `git not found` on `harumi run` / `harumi import` | git missing from PATH | Install git |
| `Not logged in. Run harumi login first.` | No session, or it expired, for the **active** environment | `harumi login` (check `harumi env current` — you may be logged in on the other env) |
| `harumi-api returned HTTP 422: ... Signups not allowed for otp` | First login for this email | `harumi login --signup` |
| `Could not provision a Gitea token: ...` (yellow, non-fatal) | Harumi Git unreachable or not configured on that backend | Non-git commands still work; re-run `harumi login` once reachable |
| `No Gitea token found. Run harumi login` | `git_token` missing from credentials | `harumi login` again |
| `No Gitea username on file` | Token saved by an older CLI version that predates the field | `harumi login` again to re-provision |
| Timeouts / connection refused on `api.dev.harumi.io` or `git.dev.harumi.io` | Staging endpoints are internal ALBs | Connect to the VPN, or switch back with `harumi env use production` |
| `Unknown environment 'x'. Known environments: production, staging.` | Typo in `--env` / `HARUMI_ENV` / `env use` | Use one of the two names |
| `You belong to multiple organizations.` | Ambiguous org after login | `harumi config set-org <ORG_ID>` (see step 4) |

## Notes for this plugin

Nothing in this plugin ships the CLI itself — the CLI is installed from PyPI, and the skills only document and drive it. If a command's real behavior ever contradicts these docs, trust `harumi --help` / `harumi <group> <cmd> --help` and say so.
