# x-cmd/action

> GitHub composite Action that bootstraps a CI runner with **x-cmd**, **git**, **docker** and **SSH** — so your workflow only contains the build step.

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[中文文档](./README.cn.md)

---

## Overview

`x-cmd/action` is a [composite GitHub Action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) from `x-cmd/action`. It collapses the repetitive "spin up a runner" steps — installing x-cmd, configuring git/docker/SSH, optionally cloning a workspace repo, and uploading artifacts — into a single reusable step.

```
┌──────────────┐    ┌──────────────┐    ┌────────────────────┐
│   init step  │ →  │   run step   │ →  │  upload-artifact   │
│ ssh / x-cmd  │    │ prehook →    │    │  (always last)     │
│ docker / git │    │ script →     │    │  ~/ws/.artifact    │
│              │    │ code →       │    │                    │
│              │    │ posthook     │    │                    │
└──────────────┘    └──────────────┘    └────────────────────┘
```

You just write a step. You don't worry about how x-cmd gets onto the runner.

---

## 1. Basic Usage

### 1.1 Hello world — one inline command

```yaml
# .github/workflows/hello.yml
name: Hello
on: [push]
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: x-cmd/action@main
        with:
          code: x cowsay "hello from x-cmd"
```

That's the whole thing. The action installs x-cmd, then `eval`s whatever you put in `code`.

### 1.2 A few one-liners to feel it out

```yaml
# inspect the runner
code: x sysinfo

# grab a binary via x eget
code: x eget use jq && jq --version

# fetch weather
code: x wttr 'Beijing'

# disk usage
code: x df
```

### 1.3 Convention script — drop a file, no `script:` needed

When `script:` is omitted, the action defaults to `.x-cmd/<github.job>` and `source`s it with bash. So a job named `build` automatically picks up `.x-cmd/build`:

```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: x-cmd/action@main
```

```bash
# .x-cmd/build
type x
x ws build
```

Override the path explicitly:

```yaml
- uses: x-cmd/action@main
  with:
    script: scripts/ci.sh
```

### 1.4 Running multiple commands — three patterns

Pick one based on how serious the script is.

#### Pattern A — multi-line `code:` (throwaway)

Quickest. Use when the commands are short and tied to the workflow file.

```yaml
- uses: x-cmd/action@main
  with:
    code: |
      x cowsay "step 1"
      x sysinfo | head
      x ws build
```

#### Pattern B — repeat the action (cleanest separation)

Each invocation is independent. Good when steps are conceptually unrelated.

```yaml
steps:
  - uses: x-cmd/action@main
    with:
      code: x ws build

  - uses: x-cmd/action@main
    with:
      code: x ws test

  - uses: x-cmd/action@main
    with:
      code: x ws publish
```

#### Pattern C — portable script (recommended)

Write the script **once**, run it **locally, in CI, anywhere**. The script bootstraps x-cmd itself so it never assumes the env.

```bash
# scripts/ci.sh — portable: works in any shell on any machine

# Bootstrap x-cmd if it isn't already loaded.
# (CI via x-cmd/action has it; local dev usually has it on PATH.)
case $- in *i*) ;; *) . "$HOME/.x-cmd.root/X" >/dev/null 2>&1 || {
    eval "$(curl -s https://get.x-cmd.com)" >/dev/null 2>&1 || true
} ;; esac

x cowsay "step 1"
x sysinfo | head
x ws build
```

```yaml
# .github/workflows/build.yml
- uses: x-cmd/action@main
  with:
    script: scripts/ci.sh
```

**Why this is the recommended pattern:**

- Same file runs `./scripts/ci.sh` on your laptop and inside GitHub Actions.
- The script is the single source of truth — no copy-paste between local dev and CI.
- The bootstrap line is a no-op when x-cmd is already loaded (interactive shell, or after the action ran init), so the cost is paid once.
- When the script grows, refactor freely without touching `action.yml`.

---

## 2. Advanced Usage

### 2.1 Pre/post hooks

Run shell code **before** and **after** the main script:

```yaml
- uses: x-cmd/action@main
  with:
    prehook: |
      x log :setup "preparing"
      mkdir -p build
    script: .x-cmd/build
    posthook: |
      x log :teardown "cleaning up"
      rm -rf build
```

Execution order: **`prehook` → `script` → `code` → `posthook`**.

### 2.2 Push back to GitHub over SSH

Set commit identity with `git_user` / `git_email`, and pass a private key with `ssh_key`. The action starts `ssh-agent`, writes `known_hosts` from [`x-cmd/knownhost`](https://github.com/x-cmd/knownhost), and `ssh-add`s your key — no manual plumbing.

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    git_user: ci-bot
    git_email: ci@example.com
    code: |
      git clone git@github.com:me/notes.git
      cd notes
      echo hello >> README.md
      git commit -am "ci: update"
      git push
```

### 2.3 Clone a workspace repo first

Set `ws_owner_repo` + `ws_repo_ref` and the action clones that repo, creates a `ws/` symlink, then `cd`s into it before running your script.

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    ws_owner_repo: x-cmd/ws
    ws_repo_ref: main
    script: .x-cmd/build
```

Defaults: `ws_owner_repo` → `${{ github.repository }}`, `ws_repo_ref` → `${{ github.head_ref || github.ref_name }}`.

Authenticated HTTPS clone when `github_token` is set.

### 2.4 Docker login + buildx

```yaml
- uses: x-cmd/action@main
  with:
    docker_username: ${{ secrets.DOCKERHUB_USERNAME }}
    docker_password: ${{ secrets.DOCKERHUB_TOKEN }}
    docker_buildx_init: 'true'
    code: docker buildx build --platform linux/amd64,linux/arm64 -t me/app .
```

### 2.5 Upload an artifact

The final step is always [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact). Anything under `~/ws/.artifact` is uploaded.

```yaml
- uses: x-cmd/action@main
  with:
    code: x ws art                     # populates ~/ws/.artifact
    artifact_name: release
    artifact_path: ~/ws/.artifact
    artifact_retention_days: 30
```

| Knob | Default | Notes |
| --- | --- | --- |
| `artifact_name` | `artifact` | |
| `artifact_path` | `~/ws/.artifact` | |
| `artifact_not_found` | `ignore` | `warn` / `error` / `ignore` |
| `artifact_retention_days` | `10` | 1–90 |

### 2.6 Matrix build across containers or OS

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        image: [xcmd/ubuntu-dev, xcmd/centos-dev]
    container:
      image: ${{ matrix.image }}
    steps:
      - uses: x-cmd/action@main
        with:
          code: x ws build ${{ matrix.image }}
```

### 2.7 Switch x-cmd release channel

By default the action pulls the stable installer (`index.html`). To test a canary/beta/dev build, set `___X_CMD_GHACTION_X`:

```yaml
- uses: x-cmd/action@main
  env:
    ___X_CMD_GHACTION_X: x1    # x0 / x1 / x2 → canary / beta / dev
  with:
    code: x --version
```

### 2.8 Fall back to env vars

Any input falls back to a matching environment variable if it isn't set explicitly. Useful for matrix jobs that share most knobs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
      git_user: ci-bot
      git_email: ci@example.com
    strategy:
      matrix: { os: [ubuntu-latest, macos-latest] }
    steps:
      - uses: x-cmd/action@main        # no `with:` needed; uses job-level env
        with:
          code: x ws build ${{ matrix.os }}
```

---

## 3. All Parameters

| Input | Default | Description |
| --- | --- | --- |
| `prehook` | — | Shell code evaluated **before** the main script. |
| `script` | `.x-cmd/<github.job>` | Script file (`source`-d with bash). |
| `code` | — | Inline shell code evaluated after the script. |
| `posthook` | — | Shell code evaluated **after** the main script. |
| `ws_owner_repo` | `${{ github.repository }}` | `<owner>/<repo>` cloned into `ws/`. |
| `ws_repo_ref` | `${{ github.head_ref \|\| github.ref_name }}` | Branch / tag / SHA. |
| `github_token` | `secrets.GITHUB_TOKEN` | Used for authenticated HTTPS clone. |
| `ssh_key` | — | Private SSH key (PEM) loaded into `ssh-agent`. |
| `git_user` | head-commit author | `git config user.name`. |
| `git_email` | head-commit author email | `git config user.email`. |
| `docker_username` | — | `docker login` username. |
| `docker_password` | — | `docker login` password. |
| `docker_buildx_init` | `false` | If truthy, runs `docker buildx create --use`. |
| `artifact_name` | `artifact` | Name passed to `upload-artifact`. |
| `artifact_path` | `~/ws/.artifact` | Path passed to `upload-artifact`. |
| `artifact_not_found` | `ignore` | `warn` / `error` / `ignore`. |
| `artifact_retention_days` | `10` | 1–90. |

`artifact_*` knobs are forwarded to the always-last `actions/upload-artifact@v4` step — the action always runs it.

---

## 4. How it Works

### 4.1 Layout

```
action.yml          composite-action metadata & steps
lib/index.sh        core init / run dispatcher (pulled via curl)
.github/workflows/  example workflows (art, build-docker, build-node, build-os, cowsay)
.x-cmd/             convention: scripts named after the job, e.g. .x-cmd/build
LICENSE             Apache 2.0
```

### 4.2 Why two `shell: bash` steps

Composite-action steps are independent shell processes — variables and functions don't carry over. The dispatcher script is downloaded to `~/xghaction` once, then re-sourced in each step so its functions come back.

### 4.3 Step 1 — `init`

```yaml
- run: |
    curl -s https://raw.githubusercontent.com/x-cmd/action/main/lib/index.sh > ~/xghaction
    . ~/xghaction init
```

`___x_cmd_ghaction_init` runs four sub-steps:

1. **SSH** — start `ssh-agent`, write `~/.ssh/known_hosts` from `x-cmd/knownhost`, `ssh-add` the key.
2. **x-cmd** — `eval "$(curl ... x-cmd/get/main/<channel>)"` installs to `~/.x-cmd.root/`.
3. **Docker** — optional `docker login` and `docker buildx create --use`.
4. **Git** — set `user.name` / `user.email`, optionally clone `ws_owner_repo` → `ws/`.

All wrapped in `set +o errexit` and `|| true`, so a sub-step failure doesn't kill the job.

### 4.4 Step 2 — `run`

```yaml
- run: . ~/xghaction run
```

`___x_cmd_ghaction_run`:

```bash
. "${___X_CMD_ROOT:-~/.x-cmd.root}/X"   # actually load x-cmd into PATH
cd ws
eval "$prehook"
source "$script"
eval "$code"
eval "$posthook"
```

Note that init is for **setup**; x-cmd is only loaded into the shell here, after step 1 has finished installing.

### 4.5 Step 3 — `actions/upload-artifact@v4`

Always runs. Forwards `artifact_*` inputs verbatim.

---

## 5. FAQ

### Do I need `actions/checkout` first?

No. The action pulls its own dispatcher via `curl` and clones the workspace repo via `git clone` only if you set `ws_owner_repo`. If you need the current repo checked out for your own scripts, add `actions/checkout@v4` separately.

### Will x-cmd installation fail break my job?

No. The init step runs with `errexit` disabled and the `eval` has `|| true`. Failures show up in logs but the workflow continues. To debug, look for `-------------------------------HOME[...]` and the curl output.

### Where is x-cmd installed?

`~/.x-cmd.root/`. It's per-runner, per-job — gone the moment the job ends.

### Can I pin a specific x-cmd version?

Indirectly — by pinning this action (`x-cmd/action@v1.2.3`) and switching `___X_CMD_GHACTION_X` to a known channel. There's no per-commit hash pinning for the x-cmd installer itself.

### Does it work on macOS / Windows runners?

`bash` + `curl` + `git` + `ssh-agent` + `docker` must be present. GitHub-hosted `ubuntu-latest` and `macos-latest` are tested. `windows-latest` may work but is not a primary target — wrap with `shell: bash` and verify.

### Can I run the action multiple times in one job?

Yes. Each invocation is independent — a fresh `init` then a fresh `run`. The first one pays the install cost; subsequent ones reuse the same `~/.x-cmd.root/` (it's idempotent).

### Why two separate bash steps instead of one big script?

Composite-action steps don't share shell state, and `init` (slow, installs x-cmd) and `run` (fast, executes user code) belong to different stages. Splitting them makes the action composable — you can `uses: x-cmd/action` for the run only, after doing your own setup.

### How is `ws/` created?

When `ws_owner_repo` and `ws_repo_ref` are both set, the action does `git clone --branch <ref> <url>` and then `ln -s $(pwd)/<repo> $(pwd)/ws`. The symlink is what your `script:` path resolves against after `cd ws`.

### What's `~/xghaction`?

A transient dispatcher file downloaded by the first step and re-sourced by the second. Safe to ignore; don't put your own file at that path.

### How do I run multiple x-cmd commands?

Three options, see [§1.4](#14-running-multiple-commands--three-patterns):

- **Multi-line `code:`** for throwaway inline sequences.
- **Repeat the action** for cleanly separated, unrelated steps.
- **Portable script** (recommended) — one file, self-bootstrapping, runs locally and in CI. The bootstrap line is:

  ```bash
  case $- in *i*) ;; *) . "$HOME/.x-cmd.root/X" >/dev/null 2>&1 || {
      eval "$(curl -s https://get.x-cmd.com)" >/dev/null 2>&1 || true
  } ;; esac
  ```

### Why does the portable-script bootstrap use `case $- in *i*) ;; *) ...`?

The `case` checks the shell's option flags (`$-`). The `*i*` matches when the shell is **interactive** — in which case x-cmd was likely loaded by your shell init (`.bashrc`, `.zshrc`, etc.) and is on PATH already, so we skip the bootstrap. In **non-interactive** shells (CI, `bash script.sh`, `sh -c "..."`), `$-` does not contain `i`, and we source `~/.x-cmd.root/X` (or curl-install if it's missing). The whole line is a no-op when x-cmd is already available.

### Why is my artifact not appearing?

Most common causes:

- `artifact_path` is wrong (default is `~/ws/.artifact`, not `./artifact`).
- `code:` ran but didn't write anything to the path.
- `artifact_not_found` is set to `error`, hiding the actual issue. Try `warn`.

---

## Related Links

- x-cmd project: <https://www.x-cmd.com/>
- x-cmd installer: <https://github.com/x-cmd/get>
- x-cmd known hosts: <https://github.com/x-cmd/knownhost>
- Composite Actions docs: <https://docs.github.com/en/actions/creating-actions/creating-a-composite-action>
- `actions/upload-artifact@v4`: <https://github.com/actions/upload-artifact>

## License

Apache License 2.0 — see [`LICENSE`](LICENSE).
