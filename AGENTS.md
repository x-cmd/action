# AGENTS.md

## docs/ 的范围

本 repo（`x-cmd/action`）的 `docs/` 目录用于介绍 **x-cmd-action/* 整个 GitHub Action 生态**——是跨 action 共享的**生态层文档**，而不是单个 action 自身的实现细节。

每个 action 自己的代码、README、SKILL.md 由该 action 的维护者维护，不在本 `docs/` 的覆盖范围内。

## 不要动

为保证现有 x-cmd workflow 稳定，以下内容**保持不变**：

- 现有代码（`action.yml`、`agent/`、`aireply/`、`lib/`、`.x-cmd/` 等）
- `README.md` 和 `README.cn.md`
- `SKILL.md`

可以新增：

- `docs/` 下的生态层文档
- 本 AGENTS.md（已存在即更新）