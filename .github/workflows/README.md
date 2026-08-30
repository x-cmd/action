# `.github/workflows/` — Example Gallery

> **This directory is a demonstration gallery, not production CI for this repo.**
> `x-cmd/action` is a library: the action itself is `action.yml` + `lib/index.sh`, and there's nothing meaningful to test in CI on every push. The workflows here exist so you can read them, copy them, and (where `workflow_dispatch` is set) trigger them by hand to see the action in action.

## What's here

| Workflow | What it demonstrates |
| --- | --- |
| `cowsay.yml` | minimal `code:` — single `x cowsay` line |
| `art.yml` | artifact upload — populates `~/ws/.artifact`, action uploads it |
| `build-os.yml` | matrix across `ubuntu-latest` + `macos-latest` |
| `build-docker.yml` | matrix across container images (`xcmd/ubuntu-dev`, `xcmd/centos-dev`) |
| `build-node.yml` | matrix over `actions/setup-node` + `x-cmd/action` (showing both styles) |
| `test-git-push.yml` | push a commit back to a repo using `ssh_key` + `git_user` + `git_email` |
| `test-workspace-repo.yml` | clone a separate repo into `ws/` via `ws_owner_repo` + `ws_repo_ref` |
| `test-docker-buildx.yml` | `docker login` + `docker buildx create --use` + multi-arch build |
| `test-hooks.yml` | `prehook` → `script` → `code` → `posthook` execution order |
| `test-pipeline.yml` | multi-job workflow — build → test → publish with artifact handoff |
| `test-toolchain.yml` | `x env use node` / `python` instead of `actions/setup-*` |

## Why `test-` prefix

The `test-` prefix marks workflows whose **primary purpose is to exercise the action** — they exist to be triggered and observed. They double as documentation (you can read them) and as smoke tests (you can run them). The unprefixed files (`cowsay.yml`, `art.yml`, `build-*.yml`) predate this convention and remain as historical examples.

## Triggering by hand

Every workflow uses `workflow_dispatch`, so it doesn't auto-run on push. To try one:

1. Open the **Actions** tab of this repo.
2. Pick a workflow from the left sidebar.
3. Click **Run workflow**.

Some require secrets (`SSH_PRIVATE_KEY`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) — those will error out cleanly if the secret isn't set, which is itself a useful demo of how the action handles missing inputs.

## Adding a new example

Keep the pattern:

```yaml
name: Example — <one-line description>
on:
  workflow_dispatch:        # never auto-run on push

jobs:
  <job>:
    runs-on: ubuntu-latest
    steps:
      - uses: x-cmd/action@main
        with:
          <inputs>
          code: |
            <shell commands, multi-line OK>
```

Use the `test-` prefix when adding a new workflow. Use `code:` for inline shell — `script:` requires a file path on disk and won't work with multi-line inline content. For longer demos, drop a script in `.x-cmd/<job-name>` and the action will auto-source it (the convention from §1.3 of the top-level README).
