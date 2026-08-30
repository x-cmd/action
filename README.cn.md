# x-cmd/action

> 一个 GitHub 复合 Action，用来为 CI 运行环境预置 **x-cmd**、**git**、**docker** 与 **SSH** —— 让 workflow 只留下真正要执行的构建步骤。

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[English](./README.md)

---

## 简介

`x-cmd/action` 是 [GitHub 复合 Action](https://docs.github.com/zh/actions/creating-actions/creating-a-composite-action)，来自 `x-cmd/action` 仓库。它把 CI 启动时那些重复劳动 —— 装 x-cmd、配 git / docker / SSH、按需克隆工作区仓库、上传产物 —— 收拢成一个可复用的 step。

```
┌──────────────┐    ┌──────────────┐    ┌────────────────────┐
│   init step  │ →  │   run step   │ →  │  upload-artifact   │
│ ssh / x-cmd  │    │ prehook →    │    │  (固定最后一步)    │
│ docker / git │    │ script →     │    │  ~/ws/.artifact    │
│              │    │ code →       │    │                    │
│              │    │ posthook     │    │                    │
└──────────────┘    └──────────────┘    └────────────────────┘
```

你只管写业务 step，x-cmd 是怎么装到 runner 上的，不必操心。

---

## 设计思想 —— 可移植的 Action

大多数 GitHub Action 都和 GitHub 的运行时深度耦合：`actions/checkout` 用 GitHub 特有的方式拉代码，`actions/setup-node` 用 GitHub 特有的方式装 Node。一旦你拿这些 action 把 CI 拼起来，你的脚本**就只能在 GitHub 上跑** —— 而且往往只能在 GitHub 官方 runner 上跑。

`x-cmd/action` 反过来：**用 POSIX shell 写脚本，到哪里都能跑。**

整个 action 只做三件事：

1. **装 x-cmd** —— 一个兼容 POSIX 的 shell 库（`dash`、`ash`、`bash`、`zsh` 都能跑），装到 `~/.x-cmd.root/`。装完以后 `x cowsay`、`x ws build`、`x eget` 就是 `PATH` 上的普通命令。
2. **跑你的脚本** —— `source` 它（或者 `eval` 内联的 `code:`），x-cmd 已经加载好了。这个脚本就是你在本地 `./scripts/ci.sh` 会跑的**同一个文件**。不用 `${{ github.* }}` 插值、不用 `actions/checkout` 套壳、也不用和 YAML 引号搏斗。
3. **上传产物** —— 唯一真正和 GitHub 强相关的步骤，因为产物存储是 GitHub 自己的服务。任何写到 `~/ws/.artifact` 下的东西都会被上传。

就这三件事。`git` 就是 `git`，`docker` 就是 `docker`，`ssh` 就是 `ssh`。action 只是装了你本来就会用的 shell 库。

**为什么这很重要：**

- **CI 脚本就是开发脚本。** 同一个文件，同一种语法，同一组命令。本地 `./scripts/ci.sh` 就能调，不用 `act`、不用 Docker、不用提了 PR 再看日志。
- **CI 提供方无关。** 从 GitHub Actions 切到别的平台，只改一行（`- uses: x-cmd/action@main` 换成新平台的等价物）。脚本正文一行都不用动。
- **Artifact 是本地与 CI 之间的桥。** 本地开发时状态留在磁盘上，CI runner 是临时的，所以 action 自动把 `~/ws/.artifact` 上传 —— 你拿到的产物就像自己亲手跑过这个脚本一样。

一句话：**CI = `装 x-cmd` + `跑脚本` + `传 artifact`。** 就这些。

---

## 1. 基本使用

### 1.1 Hello world —— 一行内联命令

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

就这样。action 帮你装好 x-cmd，然后 `eval` 你写在 `code` 里的命令。

### 1.2 几个一行示例感受一下

```yaml
# 查看 runner 信息
code: x sysinfo

# 用 x eget 拉一个二进制
code: x eget use jq && jq --version

# 查天气
code: x wttr 'Beijing'

# 看磁盘
code: x df
```

### 1.3 约定式脚本 —— 不用写 `script:`

如果不传 `script`，action 会默认 `source` `.x-cmd/<github.job>`。所以名为 `build` 的 job 自动捡到 `.x-cmd/build`：

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

需要的话也可以显式指定：

```yaml
- uses: x-cmd/action@main
  with:
    script: scripts/ci.sh
```

### 1.4 跑多条命令 —— 三种姿势

按脚本的"严肃程度"挑一种。

#### 姿势 A —— 多行 `code:`（一次性）

最快。命令又短、又只跟这个 workflow 有关时用。

```yaml
- uses: x-cmd/action@main
  with:
    code: |
      x cowsay "step 1"
      x sysinfo | head
      x ws build
```

#### 姿势 B —— 重复调用 action（最干净）

每次调用互相独立。步骤之间没有强关联时用这个最清爽。

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

#### 姿势 C —— 可移植脚本（推荐）

**一份脚本**，本地、CI、任何环境都能跑。脚本自己负责把 x-cmd 加载好，永远不假设环境。

```bash
# scripts/ci.sh —— 可移植：任何 shell、任何机器都能直接跑

# 如果 x-cmd 还没加载，先把它加载进来。
# （经 x-cmd/action 走 CI 时已经在；本地开发通常也已经在 PATH 里。）
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

**为什么这是推荐姿势：**

- 同一个文件，`./scripts/ci.sh` 在本地能跑，进了 GitHub Actions 也能跑。
- 脚本是唯一的事实来源 —— 本地开发与 CI 之间不用复制粘贴。
- 当 x-cmd 已经加载时（交互式 shell、或 action 已经走完 init），bootstrap 这一行就是个空操作，开销只付一次。
- 脚本长大了随便重构，不用动 `action.yml`。

---

## 2. 高级使用

### 2.1 前置 / 后置 hook

在主脚本前后插入 shell 代码：

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

执行顺序：**`prehook` → `script` → `code` → `posthook`**。

### 2.2 通过 SSH 推回 GitHub

`git_user` / `git_email` 设提交身份，`ssh_key` 传私钥。action 会启动 `ssh-agent`，从 [`x-cmd/knownhost`](https://github.com/x-cmd/knownhost) 写入 `known_hosts`，再 `ssh-add` —— 不用自己接。

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

### 2.3 先克隆一个工作区仓库

设 `ws_owner_repo` + `ws_repo_ref`，action 会克隆该仓库、生成 `ws/` 符号链接，再 `cd` 进去跑你的脚本。

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    ws_owner_repo: x-cmd/ws
    ws_repo_ref: main
    script: .x-cmd/build
```

默认值：`ws_owner_repo` → `${{ github.repository }}`，`ws_repo_ref` → `${{ github.head_ref || github.ref_name }}`。

设了 `github_token` 就会走带认证的 HTTPS clone。

### 2.4 Docker 登录与 buildx

```yaml
- uses: x-cmd/action@main
  with:
    docker_username: ${{ secrets.DOCKERHUB_USERNAME }}
    docker_password: ${{ secrets.DOCKERHUB_TOKEN }}
    docker_buildx_init: 'true'
    code: docker buildx build --platform linux/amd64,linux/arm64 -t me/app .
```

### 2.5 上传构建产物

最后一步固定是 [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact)。任何放在 `~/ws/.artifact` 下的内容都会被上传。

```yaml
- uses: x-cmd/action@main
  with:
    code: x ws art                     # 负责填充 ~/ws/.artifact
    artifact_name: release
    artifact_path: ~/ws/.artifact
    artifact_retention_days: 30
```

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `artifact_name` | `artifact` | |
| `artifact_path` | `~/ws/.artifact` | |
| `artifact_not_found` | `ignore` | `warn` / `error` / `ignore` |
| `artifact_retention_days` | `10` | 1–90 |

### 2.6 跨容器 / 跨 OS 做矩阵构建

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

### 2.7 切换 x-cmd 发布通道

默认拉稳定版安装器（`index.html`）。想试 canary / beta / dev：

```yaml
- uses: x-cmd/action@main
  env:
    ___X_CMD_GHACTION_X: x1    # x0 / x1 / x2 → canary / beta / dev
  with:
    code: x --version
```

### 2.8 回退到环境变量

任何 input 没显式传时都会回退到同名环境变量。matrix job 共享配置时特别好用：

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
      - uses: x-cmd/action@main        # 不写 with:，直接吃 job 级 env
        with:
          code: x ws build ${{ matrix.os }}
```

---

## 3. 全部参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `prehook` | — | 主脚本前执行的 shell 代码。 |
| `script` | `.x-cmd/<github.job>` | 用 bash `source` 的脚本文件。 |
| `code` | — | 脚本文件之后 `eval` 的内联 shell 代码。 |
| `posthook` | — | 主脚本后执行的 shell 代码。 |
| `ws_owner_repo` | `${{ github.repository }}` | 要克隆到 `ws/` 的 `<owner>/<repo>`。 |
| `ws_repo_ref` | `${{ github.head_ref \|\| github.ref_name }}` | 检出的 branch / tag / SHA。 |
| `github_token` | `secrets.GITHUB_TOKEN` | 用于带认证的 HTTPS 克隆。 |
| `ssh_key` | — | 安装到 `ssh-agent` 的私钥（PEM）。 |
| `git_user` | head-commit 作者 | `git config user.name`。 |
| `git_email` | head-commit 作者邮箱 | `git config user.email`。 |
| `docker_username` | — | `docker login` 用户名。 |
| `docker_password` | — | `docker login` 密码。 |
| `docker_buildx_init` | `false` | 真值时执行 `docker buildx create --use`。 |
| `artifact_name` | `artifact` | 传给 `upload-artifact` 的名称。 |
| `artifact_path` | `~/ws/.artifact` | 传给 `upload-artifact` 的路径。 |
| `artifact_not_found` | `ignore` | `warn` / `error` / `ignore`。 |
| `artifact_retention_days` | `10` | 1–90。 |

`artifact_*` 这一组参数原样转发给固定最后一步的 `actions/upload-artifact@v4`，整个 action 总会运行它。

---

## 4. 原理分析

从用户视角看，action 就是三步：**装 x-cmd → 跑脚本 → 传 artifact。** 本节展开这三步在工程上是怎么接起来的。

### 4.1 仓库结构

```
action.yml          复合 action 元信息与 steps
lib/index.sh        核心 init / run 分发器（通过 curl 拉取）
.github/workflows/  示例 workflow（art、build-docker、build-node、build-os、cowsay）
.x-cmd/             约定目录：用 job 名命名的脚本，例如 .x-cmd/build
LICENSE             Apache 2.0
```

### 4.2 为什么要拆多个 `shell: bash` step

用户视角的三步，对应到 composite action 的三个 step：init 负责装、run 负责跑、artifact 上传独立一步。composite action 的每个 step 都是**独立的 bash 进程**，变量与函数不共享 —— dispatcher 脚本先被下载到 `~/xghaction`，之后每个 step 都重新 `source` 它把函数找回来。

### 4.3 第一步 —— `init`

```yaml
- run: |
    curl -s https://raw.githubusercontent.com/x-cmd/action/main/lib/index.sh > ~/xghaction
    . ~/xghaction init
```

`___x_cmd_ghaction_init` 串起四个子步骤：

1. **SSH** —— 启动 `ssh-agent`，从 `x-cmd/knownhost` 写 `~/.ssh/known_hosts`，`ssh-add` 私钥。
2. **x-cmd** —— `eval "$(curl ... x-cmd/get/main/<channel>)"` 装到 `~/.x-cmd.root/`。
3. **Docker** —— 可选的 `docker login` 与 `docker buildx create --use`。
4. **Git** —— 设 `user.name` / `user.email`，按需克隆 `ws_owner_repo` → `ws/`。

整个 init 用 `set +o errexit` + `|| true` 包好，任意子步骤失败不会拖垮 job。

### 4.4 第二步 —— `run`

```yaml
- run: . ~/xghaction run
```

`___x_cmd_ghaction_run`：

```bash
. "${___X_CMD_ROOT:-~/.x-cmd.root}/X"   # 真正把 x-cmd 加载进 PATH
cd ws
eval "$prehook"
source "$script"
eval "$code"
eval "$posthook"
```

注意 init 只负责**装**，x-cmd 命令真正进入 shell 是在这一步完成之后。

### 4.5 第三步 —— `actions/upload-artifact@v4`

固定运行，把 `artifact_*` 几个 input 原样透传。

---

## 5. FAQ

### 要不要先 `actions/checkout`？

不需要。action 自己用 `curl` 抓 dispatcher，只有在设了 `ws_owner_repo` 时才会 `git clone`。如果你自己的脚本需要当前 repo 的代码，另加一个 `actions/checkout@v4`。

### x-cmd 装不上会炸吗？

不会。init step 是关掉 `errexit` 的，`eval` 还带 `|| true`。失败会出现在日志里，但 workflow 继续往下跑。想排查时找 `-------------------------------HOME[...]` 这一行以及随后的 curl 输出。

### x-cmd 装在哪里？

`~/.x-cmd.root/`。每个 runner、每个 job 独立，job 一结束就没了。

### 能锁定特定版本的 x-cmd 吗？

间接可以：锁这个 action 的版本（如 `x-cmd/action@v1.2.3`），再配 `___X_CMD_GHACTION_X` 选一个固定通道。安装器本身不支持按 commit hash 锁定。

### macOS / Windows runner 能用吗？

需要 `bash` + `curl` + `git` + `ssh-agent` + `docker` 都齐。GitHub 官方的 `ubuntu-latest` 和 `macos-latest` 是验证过的。`windows-latest` 理论能跑但不是主要目标，要 `shell: bash` 包一下并自己测。

### 一个 job 里能多次调用吗？

能。每次调用都自走一遍 init → run。第一个付安装代价，后面会复用同一个 `~/.x-cmd.root/`（安装器是幂等的）。

### 为什么拆两个 bash step，不写一个脚本？

composite-action 的 step 不共享 shell 状态，而 init（慢、装 x-cmd）和 run（快、跑业务）是两个阶段。拆开之后这个 action 才**可组合**——你可以先做自己的 setup，再 `uses: x-cmd/action` 只跑 run 阶段。

### `ws/` 是怎么生成的？

`ws_owner_repo` 和 `ws_repo_ref` 都设了的时候，action 会 `git clone --branch <ref> <url>`，再 `ln -s $(pwd)/<repo> $(pwd)/ws`。进 `ws/` 之后 `script:` 的相对路径就是相对于这个仓库的根。

### `~/xghaction` 是什么？

第一步下载下来的临时 dispatcher 文件，第二步会重新 `source` 它。别自己往这个路径写东西。

### 怎么跑多条 x-cmd 命令？

三种姿势，见 [§1.4](#14-跑多条命令--三种姿势)：

- **多行 `code:`** —— 一次性内联序列。
- **重复调用 action** —— 步骤之间彼此独立时最干净。
- **可移植脚本（推荐）** —— 一份文件、自带 bootstrap，本地与 CI 都能跑。Bootstrap 那一行长这样：

  ```bash
  case $- in *i*) ;; *) . "$HOME/.x-cmd.root/X" >/dev/null 2>&1 || {
      eval "$(curl -s https://get.x-cmd.com)" >/dev/null 2>&1 || true
  } ;; esac
  ```

### 为什么 bootstrap 用 `case $- in *i*) ;; *) ...`？

`case` 查的是 shell 的选项标志 `$-`。`*i*` 命中代表当前 shell 是**交互式** —— 这种情况下 x-cmd 通常已经被 `.bashrc` / `.zshrc` 加载到 PATH 了，bootstrap 直接跳过。**非交互式** shell（CI、`bash script.sh`、`sh -c "..."`）里 `$-` 不含 `i`，于是走 `~/.x-cmd.root/X` source（或者缺失时 curl 装）。当 x-cmd 已可用时整行就是空操作。

### 我的 artifact 没出现？

最常见的原因：

- `artifact_path` 写错了（默认是 `~/ws/.artifact`，不是 `./artifact`）。
- `code:` 跑完了但没往那个路径写东西。
- `artifact_not_found` 设成了 `error`，把真实问题藏起来了。先改成 `warn` 看一眼。

---

## 相关链接

- x-cmd 项目：<https://www.x-cmd.com/>
- x-cmd 安装脚本：<https://github.com/x-cmd/get>
- x-cmd known hosts：<https://github.com/x-cmd/knownhost>
- 复合 Action 文档：<https://docs.github.com/zh/actions/creating-actions/creating-a-composite-action>
- `actions/upload-artifact@v4`：<https://github.com/actions/upload-artifact>

## 许可证

Apache License 2.0 —— 详见 [`LICENSE`](LICENSE)。
