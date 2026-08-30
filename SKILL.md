---
name: x-cmd-action
description: Author GitHub Actions workflows that use x-cmd/action — the portable-action pattern for shell-first CI. Use when the user asks to write, fix, or review a GitHub Actions workflow that should run x-cmd commands or be portable between local and CI.
metadata:
  type: reference
  target-tool: x-cmd/action
  domain: github-actions
---

# x-cmd/action — Skill

[中文 README](./README.cn.md) · [English README](./README.md)

This skill teaches an agent how to author GitHub Actions workflows that use `x-cmd/action`. For background reading (full input tables, internal mechanics, FAQ) see the README; this file is the operating manual.

---

## 1. When to invoke

Trigger this skill when the user asks to:

- Write a new GitHub Actions workflow that runs `x ...` commands.
- Replace an existing `actions/checkout` + `actions/setup-*` chain with a portable shell script.
- Add a CI step that pushes back to GitHub, builds a Docker image, or uploads artifacts.
- Review or fix an existing workflow that uses (or should use) `x-cmd/action`.

**Do not invoke** if the user is on a non-GitHub CI provider — `x-cmd/action` is GitHub-specific. For other CI, point them to x-cmd's generic install pattern (README §A).

---

## 2. Core thesis (memorize this)

`x-cmd/action` does exactly three things:

1. **Installs x-cmd** into `~/.x-cmd.root/`.
2. **Runs a shell script** (provided as `script:`, `code:`, or a convention file).
3. **Uploads an artifact** from `~/ws/.artifact` via `actions/upload-artifact@v4`.

Everything else is the user's responsibility — by design. The action's job is *infrastructure* friction (ssh-agent, docker login, git identity). *Toolchain* friction (node, python, rust) is x-cmd's job: `x env use node`, `x npm`, etc.

**Therefore:** the workflow's job is to write a **plain POSIX shell script**. Do not reach for `actions/checkout` or `actions/setup-*` unless absolutely necessary. The script should be runnable locally with `./scripts/ci.sh` and inside GitHub Actions with no changes.

---

## 3. Decision tree — which input to use

```
"How should the user provide the script body?"
│
├─ Single short command, throwaway, tied to this workflow only
│   └─→ with: { code: "x cowsay ..." }
│
├─ Sequence of commands, but still throwaway
│   └─→ with: { code: | 
│                  x sysinfo
│                  x ws build
│              }
│
├─ Script that will grow, or one the user will also run locally
│   └─→ write scripts/ci.sh with . "$HOME/.x-cmd.root/X" at the top
│       with: { script: scripts/ci.sh }
│       (Or rely on convention: job name "build" → .x-cmd/build auto-loaded.)
│
└─ Steps that are conceptually unrelated
    └─→ repeat the action, one `with: { code: ... }` per step
```

**Default to portable script** unless the user explicitly wants inline. Ask only if the choice is unclear from context.

---

## 4. Recipes

### 4.1 Minimal — single command

```yaml
- uses: x-cmd/action@main
  with:
    code: x cowsay "hello"
```

### 4.2 Portable script (preferred)

`scripts/ci.sh`:

```bash
#!/usr/bin/env bash
. "$HOME/.x-cmd.root/X"

x sysinfo | head
x ws build
```

`workflow.yml`:

```yaml
- uses: x-cmd/action@main
  with:
    script: scripts/ci.sh
```

The script must be **runnable locally** without changes. If the user has x-cmd installed (`eval "$(curl -s https://get.x-cmd.com)"` once), `. "$HOME/.x-cmd.root/X"` works there too.

### 4.3 Convention file

If the user doesn't want a separate script and is OK with a `.x-cmd/<job-name>` convention file:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: x-cmd/action@main
        # No `script:` — defaults to .x-cmd/build
```

`.x-cmd/build`:

```bash
. "$HOME/.x-cmd.root/X"
x ws build
```

### 4.4 Git push (ssh-agent + key + git identity)

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    git_user: ci-bot
    git_email: ci@example.com
    code: |
      git clone git@github.com:me/repo.git
      cd repo
      echo updated >> README.md
      git commit -am "ci"
      git push
```

`known_hosts` is preloaded from `x-cmd/knownhost` — no manual `github.com ssh-rsa` line.

### 4.5 Clone a workspace repo

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    ws_owner_repo: owner/ws
    ws_repo_ref: main
    script: .x-cmd/build
```

The clone becomes the `ws/` symlink; the script's working directory is `ws/`. Use this when the build logic lives in a separate repo from the workflow.

### 4.6 Docker

```yaml
- uses: x-cmd/action@main
  with:
    docker_username: ${{ secrets.DOCKERHUB_USERNAME }}
    docker_password: ${{ secrets.DOCKERHUB_TOKEN }}
    docker_buildx_init: 'true'
    code: docker buildx build --platform linux/amd64,linux/arm64 -t me/app .
```

### 4.7 Matrix across containers

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

### 4.8 Artifact upload

Anything under `~/ws/.artifact` is uploaded automatically:

```yaml
- uses: x-cmd/action@main
  with:
    code: x ws art                # populates ~/ws/.artifact
    artifact_name: release
    artifact_path: ~/ws/.artifact
    artifact_retention_days: 30
```

### 4.9 Switch x-cmd stream

```yaml
- uses: x-cmd/action@main
  env:
    ___X_CMD_GHACTION_X: x7       # x0 (community-dev) / x1-x6 (experimental) / x7 (alpha)
  with:
    code: x --version
```

---

## 5. Input reference (compact)

| Input | Default | When to set |
| --- | --- | --- |
| `prehook` | — | Shell before main script |
| `script` | `.x-cmd/<job-name>` | Path to a script file |
| `code` | — | Inline shell code |
| `posthook` | — | Shell after main script |
| `ws_owner_repo` | `github.repository` | Clone a workspace repo |
| `ws_repo_ref` | `github.head_ref \|\| github.ref_name` | Branch/tag/SHA |
| `github_token` | `secrets.GITHUB_TOKEN` | Authenticated HTTPS clone |
| `ssh_key` | — | SSH private key (PEM) |
| `git_user` | head-commit author | `git config user.name` |
| `git_email` | head-commit author email | `git config user.email` |
| `docker_username` | — | `docker login` user |
| `docker_password` | — | `docker login` password |
| `docker_buildx_init` | `false` | `docker buildx create --use` |
| `artifact_name` | `artifact` | Upload name |
| `artifact_path` | `~/ws/.artifact` | Upload path |
| `artifact_not_found` | `ignore` | `warn` / `error` / `ignore` |
| `artifact_retention_days` | `10` | 1–90 |

Execution order when multiple are set: `prehook` → `script` → `code` → `posthook`.

---

## 6. Anti-patterns (do NOT do these)

- ❌ **Adding `actions/checkout` before `x-cmd/action`** — only needed if your script needs the current repo's files. The action pulls its own dispatcher via `curl` and only clones `ws_owner_repo` if you ask.
- ❌ **Adding `actions/setup-node` / `setup-python` / `setup-go`** — x-cmd provides `x env use node`, `x env use python`, `x env use go`. Prefer those.
- ❌ **Putting secrets in `code:`** — pass via `env:` and reference as `$VAR` in `code:`. The action's dedicated inputs (`ssh_key`, `docker_password`, etc.) are the proper channels.
- ❌ **Relying on `x` being on PATH without loading** — first line of any portable script is `. "$HOME/.x-cmd.root/X"`. The `X` file is idempotent.
- ❌ **Expecting `jq` / `node` / etc. on PATH after `x eget use` without loading x-cmd** — `x eget use` drops the binary into `$HOME/.local/bin/`, and `x env use` lands it under `~/.x-cmd.root/local/data/pkg/sphere/.../bin/`. Neither path is on `PATH` by default. Sourcing `X` adds them; otherwise `export PATH="$HOME/.local/bin:$PATH"` manually. See FAQ in README.
- ❌ **Hard-coding `~/ws/.artifact`** in the script — write the script to honor `$ARTIFACT_PATH` if you make it configurable; otherwise rely on the default.
- ❌ **Writing a workflow that only works on GitHub** — if the script needs `${{ github.* }}` interpolations, it's not portable. Hoist those into env vars and read with `${VAR:-}`.
- ❌ **Wrapping the action in `run: |` blocks for "extra safety"** — `with: { code: | ... }` already supports multi-line strings.

---

## 7. Verification checklist

Before declaring a workflow done:

- [ ] The script body runs locally with `./scripts/ci.sh` (assuming x-cmd is installed).
- [ ] No `actions/checkout` unless the script genuinely needs repo files.
- [ ] No `actions/setup-*` unless x-cmd doesn't cover that toolchain (rare).
- [ ] Secrets go through `env:` + dedicated inputs, never literal in `code:`.
- [ ] If pushing back to GitHub: `ssh_key` + `git_user` + `git_email` are all set.
- [ ] If uploading artifacts: verify `artifact_path` actually exists after `code:` runs.
- [ ] Matrix jobs use job-level `env:` for shared inputs (the action falls back to env vars).
- [ ] Pinned action version (e.g. `@v1.2.3`) for production, `@main` only for dev.

---

## 8. Template outputs

### 8.1 Build + test + publish (single job)

```yaml
name: CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
      git_user: ci-bot
      git_email: ci@example.com
    steps:
      - uses: x-cmd/action@main
        with:
          script: scripts/ci.sh
          artifact_name: build-output
          artifact_path: ~/ws/.artifact
```

### 8.2 Matrix across OS

```yaml
name: Build
on: [push]
jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
    steps:
      - uses: x-cmd/action@main
        with:
          code: x ws build ${{ matrix.os }}
```

### 8.3 Container matrix

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

---

## 9. Related skills

- `x-cmd-dev` — broader x-cmd module authoring.
- `repo` (under `x-cmd skill0`) — sharing repos across agents to avoid duplicate downloads.
- `verify` — running workflows locally before pushing.

## 10. Failure recovery

If a workflow fails after following these recipes:

1. Check the **init step logs** — they show `HOME[...]` and the curl output. If x-cmd install failed, the rest will too.
2. Check that the script runs locally first (`./scripts/ci.sh`).
3. Verify the workspace repo + ref are reachable if `ws_owner_repo` is set.
4. Confirm secrets are present at the job level if not set per-step.
5. Look at the action's [README FAQ](./README.md#5-faq) for the specific failure mode.
