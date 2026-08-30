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

## Design Philosophy — a portable Action

Most GitHub Actions are tightly coupled to GitHub's runtime. `actions/checkout` checks out a repo in a GitHub-specific way. `actions/setup-node` downloads Node in a GitHub-specific way. The moment you stitch your CI together out of these, your script **only runs on GitHub** — and often only on GitHub's hosted runners.

`x-cmd/action` takes the opposite approach: **write your script in plain POSIX shell, run it anywhere.**

The action does exactly three things:

1. **Install x-cmd** — a POSIX-compatible shell library (works in `dash`, `ash`, `bash`, `zsh`) into `~/.x-cmd.root/`. Once installed, `x cowsay`, `x ws build`, `x eget`, etc. are just shell commands on `PATH`.
2. **Run your script** — `source` it (or `eval` the inline `code:`), with x-cmd already loaded. This is the **same file** you'd run locally with `./scripts/ci.sh`. No `${{ github.* }}` interpolation, no `actions/checkout` shim, no YAML quoting gymnastics.
3. **Upload the artifact** — the only step that's truly GitHub-specific, because artifact storage is GitHub's service. Anything you write to `~/ws/.artifact` gets uploaded.

That's the whole design. `git` is just `git`, `docker` is just `docker`, `ssh` is just `ssh`. The action installs the shell library you already know how to use.

**Why this matters:**

- **Your CI script *is* your dev script.** Same file, same syntax, same commands. Debug locally with `./scripts/ci.sh` — no `act`, no Docker, no PR-then-watch-logs loop.
- **CI-provider-agnostic.** Switching from GitHub Actions to another provider means changing one line (`- uses: x-cmd/action@main` → the new provider's equivalent). The script body is untouched.
- **The artifact is the bridge between local and CI.** Locally you keep state on disk; in CI the runner is ephemeral, so the action uploads `~/ws/.artifact` automatically — you get back the same files you would have had if you'd run the script yourself.

In other words: **CI = `install x-cmd` + `run my script` + `upload artifact`.** That's it.

---

# Part A — Using x-cmd Without This Action

> Skip everything below if you came here just to plug `x-cmd/action` into a workflow. This part is for users who **don't** need GitHub Actions, or want to understand what the action is doing under the hood.

## A.1 Load x-cmd into a shell

The same one-liner works in **both** a script body and a shell init file:

```bash
if [ -f "$HOME/.x-cmd.root/X" ]; then
    . "$HOME/.x-cmd.root/X"
else
    eval "$(curl -s https://get.x-cmd.com)"
fi
```

- **In a script** — paste at the top. Idempotent: re-runs fine because `X` itself guards against double-loading.
- **Globally (per user)** — paste into `~/.bashrc` / `~/.zshrc`. After login, `x <module>` works in every interactive shell.

## A.2 Calling x-cmd — two distinct modes

The two invocations styles look interchangeable but are **not equivalent**. Pick deliberately based on whether you need shell state to survive across calls.

```bash
# Mode 1: as an external command — fresh subshell each call
x-cmd cowsay "hello"

# Mode 2: load x-cmd into the current shell, then call x as a function
. "$HOME/.x-cmd.root/X"
x cowsay "hello"
x ws build
x sysinfo
```

| | Mode 1: `x-cmd <module>` | Mode 2: `. X` then `x <module>` |
| --- | --- | --- |
| What `x` is | an external program on `PATH` | a shell function defined by sourcing `X` |
| Setup needed | none | must `. X` first |
| Per-call cost | fork + exec | direct function call |
| Sees shell-local vars / functions / aliases | no (fresh shell each call) | yes |
| Can mutate the calling shell (set vars, define aliases, modify `PATH`) | no | yes |
| Available in shells that haven't sourced `X` | yes, if `x-cmd` is on `PATH` | no |
| Typical use | cron jobs, `sh -c "..."`, before login scripts, one-off CLI | interactive shell, scripts that call `x` many times |

**The non-equivalence in one sentence:** Mode 2 lets `x` participate in your shell's state; Mode 1 cannot. If your script needs `x` to see (or change) variables defined outside the call, you need Mode 2.

Note also that the two are not always both available — Mode 1 requires `x-cmd` to have been installed as an external command on `PATH`; Mode 2 only needs the `X` source file to exist. The `if [ -f "$HOME/.x-cmd.root/X" ]` pattern in §A.1 covers Mode 2's bootstrap regardless of whether Mode 1 is wired up.

## A.3 What `x-cmd/action` does, in those terms

`x-cmd/action`'s init step is just **A.1** in disguise: it ensures `~/.x-cmd.root/X` exists (by `eval`-installing via `x-cmd/get`), then the run step `. ${___X_CMD_ROOT}/X` switches you into **Mode 2**. The artifact upload at the end is the only piece that's GitHub-specific. Everything else is plain x-cmd.

---

# Part B — Using x-cmd Inside GitHub Actions

> The rest of this README documents `x-cmd/action` itself.

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

Write the script **once**, run it **locally, in CI, anywhere**. One line at the top loads x-cmd into the current shell.

```bash
# scripts/ci.sh — portable: works in any shell on any machine

. "$HOME/.x-cmd.root/X"

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
- When x-cmd is missing, the script fails loudly with a clear error — which is what you want.
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

At the user level, the action is just three steps: **install x-cmd → run your script → upload artifact.** The rest of this section is the mechanical view of how those three steps are wired up.

### 4.1 Layout

```
action.yml          composite-action metadata & steps
lib/index.sh        core init / run dispatcher (pulled via curl)
.github/workflows/  example workflows (art, build-docker, build-node, build-os, cowsay)
.x-cmd/             convention: scripts named after the job, e.g. .x-cmd/build
LICENSE             Apache 2.0
```

### 4.2 Why two `shell: bash` steps

The three user-facing steps map onto three composite steps: `init` runs the install, `run` executes the script, and the artifact upload is a third. Composite-action steps are independent shell processes — variables and functions don't carry over — so the dispatcher script is downloaded to `~/xghaction` once, then re-sourced in each step so its functions come back.

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

**Conditional execution.** Every sub-step is gated by its inputs — without the matching input (or env var), the sub-step is skipped entirely. So a minimal workflow using only `code:` only pays for the x-cmd install:

| Sub-step | Runs when |
| --- | --- |
| x-cmd install | always |
| SSH (known_hosts, ssh-agent, ssh-add) | `ssh_key` is set |
| Docker login | `docker_username` **and** `docker_password` are both set |
| Docker buildx init | `docker_buildx_init` is truthy |
| Git `user.name` / `user.email` | `git_user` **or** `git_email` is set |
| Clone workspace repo | `ws_owner_repo` **and** `ws_repo_ref` are both set |

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

### 4.6 Scope — what's first-class, what's delegated

`x-cmd/action` only handles a small set of "first-class" infrastructure: **SSH, Docker, AI credentials, and the auto-available `GITHUB_TOKEN`**. Everything else — language toolchains, package managers, linters, anything you'd `apt install` or `brew install` — is deliberately **not** the action's job.

**Why this split:**

- **SSH, Docker, AI, and `gh`-token are both essential and security-sensitive.** They each get a dedicated input so secrets don't have to be pasted into a `code:` string, and so the action can wire them into the system correctly (loading `ssh-agent`, writing `~/.docker/config.json`, etc.) rather than leaving the script to do it ad-hoc.
- **Other tools can be installed on-demand from inside the script.** x-cmd itself ships modules for most ecosystems; if the action tried to provide all of them, every invocation would re-download dozens of toolchains. Better to let the script pull what it needs, when it needs it.
- **Goal: parity with local dev, with minimum config.** When you sit down at your laptop you already have `git`, `ssh`, `docker`, plus whatever languages you happen to use (`node`, `python`, etc.). `x-cmd/action` replicates the first half out of the box; the second half is one line in the script — `x env use node`, `x env use python`, etc.
- **Future direction.** x-cmd's hub will surface `npm`, `crates.io`, `pip`, `jsr` as first-class x-cmd modules. Once that lands, no action input is needed for them — `x npm`, `x cargo`, etc. will work the same way inside the action as they do locally.

**In short:** the action's job is to remove *infrastructure* friction. *Toolchain* friction is x-cmd's job.

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

### How does `x-cmd/action`'s install differ from `eval "$(curl -s https://get.x-cmd.com)"`?

**Functionally identical** — both end up with the same x-cmd installed under `~/.x-cmd.root/`. The action just knows it's inside GitHub Actions, so it pins the install to a specific URL.

From `lib/index.sh`, the action does:

```bash
eval "$(curl "https://raw.githubusercontent.com/x-cmd/get/main/$___X_CMD_GHACTION_X")" || true
```

…where `___X_CMD_GHACTION_X` defaults to `index.html` (stable), with `x0` / `x1` / `x2` available for canary / beta / dev.

The generic `eval "$(curl -s https://get.x-cmd.com)"` is the same content served via a different endpoint — a CDN-fronted domain that ultimately serves `x-cmd/get`'s `index.html`. In the action's git history you can see the older versions of this line used `x-bash/get` and `get.x-cmd.com` before settling on the direct raw URL.

**Why the action uses a direct GitHub URL:**

For **GitHub-hosted runners** (the default — `ubuntu-latest`, `macos-latest`, etc.), the runner runs inside GitHub's own data centers. `raw.githubusercontent.com` is also GitHub's infrastructure, so the `curl` is effectively an **internal fetch** — same network, no external hop, no cross-ISP routing, no third-party DNS. By contrast, `get.x-cmd.com` is an external CDN: the request leaves GitHub's network, hits the public DNS, traverses the public internet, and lands at a third-party edge. Same content either way; on hosted runners the internal path is shorter and more controlled.

For **self-hosted runners**, this assumption doesn't hold — your runner is wherever you put it. Connectivity to `raw.githubusercontent.com` and to `get.x-cmd.com` will depend on your own network. In that case the CDN URL may actually be faster, but you can still use the action and let it pick the raw URL; just don't assume the topology benefit applies.

**One important counterpoint.** `get.x-cmd.com` carries a regional routing layer that `raw.githubusercontent.com` does not — explicit support for connectivity from China is part of why the CDN exists. So:

- For **hosted runners** (mostly in US/EU data centers), `raw.githubusercontent.com` wins on the internal-fetch argument.
- For **self-hosted runners in China**, or for **users in China installing x-cmd locally**, `get.x-cmd.com` is often the more reliable choice — its routing is tuned for that network.

The action's hard-coded raw URL is the right default for the common case (hosted runners worldwide), but it's worth knowing the trade-off if you're running x-cmd from inside China.

**The two real differences in summary:**

| | `x-cmd/action` | `eval "$(curl ... get.x-cmd.com)"` |
| --- | --- | --- |
| Install URL | direct raw: `raw.githubusercontent.com/x-cmd/get/main/<channel>` | CDN: `get.x-cmd.com` |
| Network path (on GitHub-hosted runners) | internal to GitHub's data center | leaves GitHub's network → public internet → CDN edge |
| Network path (on self-hosted runners) | depends on your network | depends on your network |
| Channel selection | explicit via `___X_CMD_GHACTION_X` env var (`index.html` / `x0` / `x1` / `x2`) | whatever the CDN serves at request time |
| Failure handling | `\|\| true` + init context has `errexit` off — install failure doesn't kill the job | up to the caller |

Use `x-cmd/action` inside GitHub Actions; use the generic install (§A.1 pattern) everywhere else.

### Why can't my shell find binaries installed by `x eget use` or `x env use`?

`x eget use` drops the binary into `$HOME/.local/bin/`. `x env use` (and other x-cmd package installs) drop into a per-package path under `~/.x-cmd.root/local/data/pkg/sphere/.../bin/`. Neither directory is on `PATH` by default in a fresh shell — installing a tool is not the same as putting it on `PATH`.

**Fix:** either configure `PATH` explicitly, or load x-cmd into the current shell — `. "$HOME/.x-cmd.root/X"` does the `PATH` setup for you.

```bash
# Option 1: source x-cmd first — PATH is wired up
. "$HOME/.x-cmd.root/X"
x eget use jq
jq --version                # works

# Option 2: prepend the install paths yourself
export PATH="$HOME/.local/bin:$PATH"
```

**Inside `x-cmd/action`:** the run step already does `. $___X_CMD_ROOT/X`, so binaries installed by `x eget use` / `x env use` inside `code:` are callable from the same `code:` block (and from the same step's subsequent invocations):

```yaml
- uses: x-cmd/action@main
  with:
    code: |
      x eget use jq
      jq --version            # works — PATH was set up by the action's . X
```

**Across separate action steps:** each step is an independent bash process, so `PATH` does **not** carry over. If step A does `x eget use jq` and step B wants to call `jq`, step B must load x-cmd again (which the action does automatically) — and since the install in `$HOME/.local/bin` survives between steps, `jq` will be found as long as x-cmd is loaded in step B.

### What's `~/xghaction`?

A transient dispatcher file downloaded by the first step and re-sourced by the second. Safe to ignore; don't put your own file at that path.

### How do I run multiple x-cmd commands?

Three options, see [§1.4](#14-running-multiple-commands--three-patterns):

- **Multi-line `code:`** for throwaway inline sequences.
- **Repeat the action** for cleanly separated, unrelated steps.
- **Portable script** (recommended) — one file, runs locally and in CI. Add this line at the top:

  ```bash
  . "$HOME/.x-cmd.root/X"
  ```

  In CI via `x-cmd/action`, x-cmd is installed by the `init` step before the script runs, so this `source` succeeds. Locally, run `eval "$(curl -s https://get.x-cmd.com)"` once to install, then the same line works thereafter.

  `X` is **idempotent** — sourcing it again on a shell where x-cmd is already loaded is a no-op, so it's safe to leave the line at the top of every script (and safe to source the script itself multiple times).

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

## Example Workflows

The `.github/workflows/` directory is a **demonstration gallery**, not production CI for this repo (the action itself is just `action.yml` + `lib/index.sh`). Each workflow is `workflow_dispatch`-only — never auto-runs on push — and you can trigger it by hand from the Actions tab to see the pattern working end-to-end.

See [`.github/workflows/README.md`](.github/workflows/README.md) for the full list. Highlights:

- **`test-git-push.yml`** — push a commit back using `ssh_key` + `git_user` + `git_email`.
- **`test-workspace-repo.yml`** — clone a separate repo into `ws/` via `ws_owner_repo`.
- **`test-docker-buildx.yml`** — Docker login + buildx init + multi-arch build.
- **`test-hooks.yml`** — trace `prehook` → `script` → `code` → `posthook` order.
- **`test-pipeline.yml`** — multi-job `build → test → publish` with artifact handoff.
- **`test-toolchain.yml`** — `x env use node` / `python` instead of `actions/setup-*`.

## License

Apache License 2.0 — see [`LICENSE`](LICENSE).
