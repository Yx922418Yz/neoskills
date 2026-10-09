# neoskills 中文使用说明（README.zh-CN）

> 本文件由李亚轩（GitHub: Yx922418Yz）在 fork 仓库的 `add-chinese-readme` 分支上
> 亲自撰写，作为对 [neolaf2/neoskills](https://github.com/neolaf2/neoskills) 的中文贡献。
> 这不是机器翻译原文，而是我通读源码与 README 后，面向中文新手写的"上手指南 + 原理解读"。
> 英文原版见 [README.md](./README.md)。

---

## 一句话它是干什么的

**neoskills = 给 AI 编程 Agent 用的"Homebrew"。**

如果你给 Claude Code / OpenCode 等 Agent 装过一堆技能（skill），迟早会遇到三个痛点：
散落在各处、重复、不知道谁依赖谁。neoskills 用一个中央"技能库（bank）"统一管理它们，
用**符号链接（symlink）**把技能按需部署到不同 Agent，并在文件系统之上叠了一层**本体图谱（ontology）**来描述技能之间的关系。

- **输入**：一条命令（如 `neoskills install kstar-loop`）或一组 git 技能仓库（tap）。
- **输出**：被部署到你 Agent 技能目录里的可用技能，外加一张可查询、可可视化的技能关系图。

## 解决什么问题

| 痛点 | neoskills 怎么解 |
|------|------------------|
| 同一个技能在 Claude、OpenCode 里各存一份，改一处忘了另一处 | 中央 bank 存唯一真源，Agent 目录里只放符号链接，改 bank 处处生效 |
| 不知道自己装了哪些技能、哪些重复了 | `list` / `search` / `skill-dedup` 技能做盘点与去重 |
| 技能之间谁依赖谁、谁和谁冲突，全靠人脑记 | ontology 层把 `requires/extends/composes/conflicts` 存成 `ontology.yaml` 边 |
| 换台机器要重装一遍 | tap 是 git 仓库，`tap` + `install` 可一键还原 |

## 安装

要求 **Python 3.13+**、`git`（做 tap 操作）、推荐 `uv`。

```bash
# 推荐：PyPI
pip install neoskills
# 或
uv tool install neoskills

# 从源码
git clone https://github.com/neolaf2/neoskills
cd neoskills
uv sync --dev
uv run neoskills --help
```

## 30 秒上手

```bash
neoskills init                 # 1) 建 ~/.neoskills/ 工作区 + 默认 tap
neoskills tap https://github.com/你的组织/my-skills   # 2) 加一个技能仓库
neoskills list                 # 3) 看看有哪些技能
neoskills install kstar-loop   # 4) 装一个技能：拷进 tap + 软链到 Agent
neoskills create my-new -d "做什么" --type task   # 5) 脚手架式新建一个技能
neoskills doctor               # 6) 体检：软链/配置/tap 是否健康
```

执行完 `install` 后，`~/.claude/skills/kstar-loop` 会变成一个指向
`~/.neoskills/taps/<tap>/skills/kstar-loop` 的软链——**零拷贝、可秒删**。

## 一个技能长什么样

每个技能至少是一个带 `SKILL.md` 的目录：

```yaml
---
name: my-skill
description: "这个技能做什么"
author: "你的名字"
tags: [productivity, automation]
targets: [claude-code]
---
# 正文：Agent 会读的指令、prompt、文档
```

可选地，旁边放一个 `ontology.yaml` 记录图谱元数据（类型、领域、生命周期、边、版本）。

## 本体图谱（ontology）——本项目最有想法的部分

neoskills 没有引入外部数据库，而是**把属性图直接物化在文件系统上**：
节点 = 技能目录，边 = 每个目录里的 `ontology.yaml`。运行时再从磁盘"长"出整张图。

它做了**渐进式富化**设计，技能不必一开始就写全元数据：

| 级别 | 有什么 | 怎么到这一级 |
|------|--------|--------------|
| L0 裸 | 只有 SKILL.md | 所有老技能的默认态 |
| L1 打标 | + 类型/领域/标签 | `neoskills ontology enrich <id>` |
| L2 连接 | + 边（requires/extends/composes/conflicts） | 作者显式声明关系 |
| L3 治理 | + 生命周期、版本、能力清单 | 随时间维护 |

常用命令：

```bash
neoskills ontology discover --domain document-processing   # 按面搜索
neoskills ontology deps kstar-loop --transitive            # 它依赖谁
neoskills ontology graph kstar-loop --depth 2 --format mermaid   # 画关系图
neoskills ontology validate                                # 查坏边/环/冲突
```

边的类型：`requires` / `extends` / `composes_with` / `conflicts_with` / `supersedes` / `derived_from`。
技能生命周期状态机：`candidate → validated → operational → refined → deprecated → archived`。

## 仓库自带的两个示例技能

- **`bank-status`**：纯指令型技能。当用户问"我技能库里现在有啥"时触发，
  它读 `~/.neoskills/registry.yaml` / `state.yaml`，汇总技能总数、已部署目标、最近导入的 5 个技能。
- **`skill-dedup`**：带脚本的技能。`scripts/dedup_scan.py` 扫描 bank 与各 Agent 目录，
  把重复分三类——**完全重复**（SHA256 相同，可安全软链合并）、
  **分叉副本**（同 ID 内容不同，推荐"更富"的那份入库）、**名字相似组**（需人工判断）。
  支持 `--resolve exact|diverged|all --dry-run` 先预览再动手，并自动留 `.dedup-backup` 备份。

## 目录结构速览

```
neoskills/
├── src/neoskills/
│   ├── cli/        # Click 命令入口（tap/link/list/doctor/ontology 等）
│   ├── core/       # 工作区、配置、frontmatter、软链器、checksum、tap
│   ├── ontology/   # 属性图层：graph/loader/writer/engine/taxonomy/lifecycle/versioning...
│   ├── adapters/   # 不同 Agent 后端适配（claude-code / opencode / openclaw）
│   ├── runtime/    # Agent 运行时集成（如 Claude 的 MCP 插件）
│   └── meta/       # AI 增强（enhance audit/normalize/add-docs）
├── skills/         # 自带示例技能（bank-status、skill-dedup）
├── agents/         # 可自主调用的子 agent（dedup/deployer/importer/scanner）
├── commands/       # 斜杠命令（ns.md）
├── tests/          # 172 个测试（unit + integration）
└── docs/ontology-design.md   # 本体层完整设计文档
```

## 复现与开发

```bash
uv run pytest -v        # 跑全部测试
uv run ruff check src/  # lint
uv run neoskills --help
```

---

> 📝 本中文说明由李亚轩撰写于 `add-chinese-readme` 分支，欢迎指正。
> 英文原版与最新命令以 [README.md](./README.md) 与上游为准。
