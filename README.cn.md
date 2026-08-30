# x-cmd/action

> 一个 GitHub 复合 Action，用来为 CI 运行环境预置 **x-cmd**、**git**、**docker** 与 **SSH**。

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

[English](./README.md)

---

## 1. 快速上手

最基础的用法 —— 在 GitHub 官方 runner 上跑一条 **x-cmd** 命令：

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

就这样。action 会帮你装好 x-cmd，然后 `eval` 你写在 `code` 里的任何命令。

再多看几个一行示例感受一下：

```yaml
# 查看 runner 信息
code: x sysinfo

# 用 x eget 拉一个二进制
code: x eget use jq && jq --version

# 查天气
code: x wttr 'Beijing'
```

---

## 2. 约定式脚本

如果不传 `script`，action 会默认 `source` `.x-cmd/<github.job>`。所以名为 `build` 的 job 会自动捡到 `.x-cmd/build`：

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

也可以显式指定路径：

```yaml
- uses: x-cmd/action@main
  with:
    script: scripts/ci.sh
```

---

## 3. 前置 / 后置 hook

在主脚本执行前后插入 shell 代码：

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

执行顺序：`prehook` → `script` → `code` → `posthook`。

---

## 4. 推送回 GitHub

用 `git_user` / `git_email` 设置提交身份，配合 `ssh_key` 完成推送。action 会启动 `ssh-agent`，从 [`x-cmd/knownhost`](https://github.com/x-cmd/knownhost) 写入 `known_hosts`，再 `ssh-add` 你的私钥：

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

## 5. 先克隆一个工作区仓库

设置 `ws_owner_repo` + `ws_repo_ref`，action 会先克隆该仓库、再 `cd` 进去跑你的脚本，克隆出来的目录会被符号链接为 `ws/`：

```yaml
- uses: x-cmd/action@main
  env:
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
  with:
    ws_owner_repo: x-cmd/ws
    ws_repo_ref: main
    script: .x-cmd/build
```

默认值：`ws_owner_repo` 回退到 `${{ github.repository }}`，`ws_repo_ref` 回退到 `${{ github.head_ref || github.ref_name }}`。

---

## 6. Docker 登录与 buildx

```yaml
- uses: x-cmd/action@main
  with:
    docker_username: ${{ secrets.DOCKERHUB_USERNAME }}
    docker_password: ${{ secrets.DOCKERHUB_TOKEN }}
    docker_buildx_init: 'true'
    code: docker buildx build --platform linux/amd64,linux/arm64 -t me/app .
```

---

## 7. 上传构建产物

最后一步固定是 [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact)。任何放在 `~/ws/.artifact` 下的内容都会被上传：

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

---

## 8. 跨容器 / 跨 OS 做矩阵构建

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

## 9. 全部输入参数

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

> 所有 input 都会回退到同名 **环境变量**，对 matrix job 特别方便。

---

## 10. 内部原理

```
action.yml          复合 action 元信息与 steps
lib/index.sh        核心 init / run 分发器（通过 raw.githubusercontent.com 拉取）
.github/workflows/  示例 workflow（art、build-docker、build-node、build-os、cowsay）
.x-cmd/             约定目录：用 job 名命名的脚本，例如 .x-cmd/build
LICENSE             Apache 2.0
```

action 在内部按顺序完成六件事：

1. **SSH** —— 启动 `ssh-agent`，写入 `known_hosts`，`ssh-add` 私钥。
2. **x-cmd** —— 从 [`x-cmd/get`](https://github.com/x-cmd/get) 拉取安装脚本，注入 `~/.x-cmd.root/`。
3. **Docker** —— 可选的 `docker login` 与 `docker buildx create --use`。
4. **Git** —— 设置 `user.name` / `user.email`，按需克隆工作区仓库。
5. **执行** —— `prehook` → `script` → `code` → `posthook`。
6. **产物** —— 通过 `actions/upload-artifact@v4` 上传。

## 许可证

Apache License 2.0 —— 详见 [`LICENSE`](LICENSE)。

## 相关链接

- x-cmd 项目：<https://www.x-cmd.com/>
- x-cmd 安装脚本：<https://github.com/x-cmd/get>
- x-cmd known hosts：<https://github.com/x-cmd/knownhost>
- 复合 Action 文档：<https://docs.github.com/zh/actions/creating-actions/creating-a-composite-action>
- `actions/upload-artifact@v4`：<https://github.com/actions/upload-artifact>
