# x-cmd/action

> A GitHub composite Action that bootstraps a CI runner with **x-cmd**, **git**, **docker** and **SSH**.

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

[中文文档](./README.cn.md)

---

## 1. Quick start

The most basic use — run an **x-cmd** command on a GitHub-hosted runner:

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

That's it. The action installs x-cmd for you, then `eval`s whatever you put in `code`.

A few more one-liners to feel it out:

```yaml
# inspect the runner
code: x sysinfo

# pull a binary via x eget
code: x eget use jq && jq --version

# fetch weather
code: x wttr 'Beijing'
```

---

## 2. Script by convention

If `script` is not set, the action defaults to `.x-cmd/<github.job>` and `source`s it with bash. So a job named `build` will automatically pick up `.x-cmd/build`:

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

You can override the path:

```yaml
- uses: x-cmd/action@main
  with:
    script: scripts/ci.sh
```

---

## 3. Pre/post hooks

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

Execution order: `prehook` → `script` → `code` → `posthook`.

---

## 4. Push back to GitHub

Use `git_user` / `git_email` to set commit identity, plus `ssh_key` for pushes. The action loads `ssh-agent`, populates `~/.ssh/known_hosts` from [`x-cmd/knownhost`](https://github.com/x-cmd/knownhost), then `ssh-add`s your key:

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

---

## 5. Clone a workspace repo first

Set `ws_owner_repo` + `ws_repo_ref` and the action will clone that repo and `cd` into it before running your script. The clone becomes the symlink `ws/`:

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    ws_owner_repo: x-cmd/ws
    ws_repo_ref: main
    script: .x-cmd/build
```

Defaults: `ws_owner_repo` falls back to `${{ github.repository }}`, `ws_repo_ref` to `${{ github.head_ref || github.ref_name }}`.

---

## 6. Docker login & buildx

```yaml
- uses: x-cmd/action@main
  with:
    docker_username: ${{ secrets.DOCKERHUB_USERNAME }}
    docker_password: ${{ secrets.DOCKERHUB_TOKEN }}
    docker_buildx_init: 'true'
    code: docker buildx build --platform linux/amd64,linux/arm64 -t me/app .
```

---

## 7. Upload an artifact

The last step is always [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact). Anything under `~/ws/.artifact` is uploaded:

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

---

## 8. Matrix build across containers / OS

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

## 9. All inputs

| Input | Default | Description |
| --- | --- | --- |
| `prehook` | — | Shell code evaluated **before** the main script. |
| `script` | `.x-cmd/<github.job>` | Script file (`source`-d with bash). |
| `code` | — | Inline shell code evaluated after the script. |
| `posthook` | — | Shell code evaluated **after** the main script. |
| `ws_owner_repo` | `${{ github.repository }}` | `<owner>/<repo>` cloned into `ws/`. |
| `ws_repo_ref` | `${{ github.head_ref \|\| github.ref_name }}` | Branch / tag / SHA. |
| `github_token` | `secrets.GITHUB_TOKEN` | Authenticated HTTPS clone when set. |
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

> Any input falls back to the matching **environment variable** if it isn't set explicitly — handy for matrix jobs.

---

## 10. How it works

```
action.yml          composite-action metadata & steps
lib/index.sh        core init / run dispatcher (sourced from raw.githubusercontent.com)
.github/workflows/  example workflows (art, build-docker, build-node, build-os, cowsay)
.x-cmd/             convention: scripts named after the job, e.g. .x-cmd/build
LICENSE             Apache 2.0
```

Internally the action does six things, in order:

1. **SSH** — start `ssh-agent`, install `known_hosts`, `ssh-add` the key.
2. **x-cmd** — source the installer from [`x-cmd/get`](https://github.com/x-cmd/get) into `~/.x-cmd.root/`.
3. **Docker** — optional `docker login` + `docker buildx create --use`.
4. **Git** — set `user.name` / `user.email`, optionally clone a workspace repo.
5. **Run** — `prehook` → `script` → `code` → `posthook`.
6. **Artifact** — upload via `actions/upload-artifact@v4`.

## License

Apache License 2.0 — see [`LICENSE`](LICENSE).

## Related Links

- x-cmd project: <https://www.x-cmd.com/>
- x-cmd installer: <https://github.com/x-cmd/get>
- x-cmd known hosts: <https://github.com/x-cmd/knownhost>
- Composite Actions docs: <https://docs.github.com/en/actions/creating-actions/creating-a-composite-action>
- `actions/upload-artifact@v4`: <https://github.com/actions/upload-artifact>
