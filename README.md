# Trellis 完整指南

> 面向 AI 编程的「开箱即用」工程框架。一句话概括：**Trellis 是 AI 编程助手的辅助轮（training wheels）**——它把你的项目规范、任务上下文与历史记忆持久化进仓库，并在每次 AI 会话中自动注入，让 AI 按你的工程标准写代码，而不是每次都「即兴发挥」。

- 官网文档：https://docs.trytrellis.app/
- 开源仓库：https://github.com/mindfold-ai/Trellis （`mindfold-ai/Trellis`）
- 文档索引（给 LLM 用）：https://docs.trytrellis.app/llms.txt
- 本指南整理时间：2026-06-29，对应版本约 `v0.6.x`

---

## 目录

1. [它解决什么问题](#一它解决什么问题)
2. [核心理念与定位](#二核心理念与定位)
3. [与传统方案的对比](#三与传统方案的对比)
4. [核心概念速查](#四核心概念速查)
5. [目录结构](#五目录结构)
6. [安装与第一个任务](#六安装与第一个任务)
7. [完整工作流（运行时全流程）](#七完整工作流运行时全流程)
8. [命令、技能与子代理速查](#八命令技能与子代理速查)
9. [task.py 任务脚本](#九taskpy-任务脚本)
10. [如何写出 AI 真正会遵守的 Spec](#十如何写出-ai-真正会遵守的-spec)
11. [多平台与团队协作](#十一多平台与团队协作)
12. [典型使用场景](#十二典型使用场景)
13. [最佳实践与常见误区](#十三最佳实践与常见误区)
14. [结合本工作空间（MoreLogin）的落地建议](#十四结合本工作空间morelogin的落地建议)

---

## 一、它解决什么问题

AI 写代码很快，但**每次会话都从零开始**：

- 它不记得你的项目结构、命名约定、技术栈选择；
- 它不记得团队踩过的坑和达成的决策；
- 它不记得上一次会话做了什么、做到哪一步。

结果是 AI 不断「即兴发挥」，产出风格漂移、重复犯错、与团队规范脱节的代码。

Trellis 的思路：**把「项目知识」从聊天记录里搬进仓库文件**，再通过 hook / 前置注入机制，在每次会话、每个任务、每个子代理启动时，**精准地按需注入**相关上下文。这样无论换人、换工具、换会话，AI 都在同一套约定下工作。

> 官方比喻：AI 的能力像藤蔓一样疯长，生命力旺盛但四处蔓延；Trellis 就是那架「棚架」（trellis 本意即「棚架/格架」），引导藤蔓沿着你的约定生长。

---

## 二、核心理念与定位

官方把 Trellis 定位为 **「团队级的 Agent Harness（智能体编排外壳）+ 内置 LLM Wiki」**，由三层组成：

- **Agent Harness（编排外壳）**：工作流状态机、hooks、skills、sub-agents、平台适配器，控制 AI 编码工作如何流转。
- **内置 LLM Wiki**：spec / task / research / journal 全部以文件形式存进仓库，让 AI 会话能从文件重新加载项目知识。
- **团队级层**：工作流/规范/任务文件随 Git 版本管理，配合每位开发者的工作区记忆，让「多人 + 多 AI 工具」都对齐到同一套约定。

四大支柱能力：

| 能力 | 它改变了什么 |
| --- | --- |
| **Spec 自动注入** | 约定只在 `.trellis/spec/` 写一次，之后每个会话自动注入相关上下文，无需重复交代 |
| **以任务为中心的工作流** | PRD、实现上下文、评审上下文、任务状态都放进 `.trellis/tasks/`，让 AI 的工作结构化 |
| **项目记忆** | `.trellis/workspace/` 里的日志保留「上次做了什么」，每次新会话都有真实上下文 |
| **团队共享标准** | Spec 随仓库走，一个人沉淀的工作流/规则能惠及全团队 |
| **多平台一致性** | 同一套 Trellis 结构可用于 14+ AI 编码平台，无需为每个工具重建工作流 |

---

## 三、与传统方案的对比

| 维度 | `.cursorrules` | `CLAUDE.md` | Skills | **Trellis** |
| --- | --- | --- | --- | --- |
| Spec 注入方式 | 每次对话手动加载 | 自动加载但易被截断 | 用户主动触发 | **自动注入（支持 hook 的平台用 hook，其余用前置 prelude），按任务精准加载** |
| Spec 粒度 | 单个大文件 | 单个大文件 | 每个技能一份 | **模块化文件，按任务组合** |
| 跨会话记忆 | 无 | 无 | 无 | **Workspace 日志持久化** |
| 工作流强制 | 无 | 无 | 无 | **自动触发技能 + check 子代理校验闭环** |
| 团队共享 | 单用户 | 单用户 | 可分享但无标准 | **Git 版本化的 Spec 库** |
| 平台支持 | 仅 Cursor | 仅 Claude Code | 各平台单独配 | **14 个已配置平台 + 共享技能生态** |

支持平台：Claude Code、Cursor、OpenCode、Codex、Kiro、Kilo、Gemini CLI、Antigravity、Devin、Qoder、CodeBuddy、GitHub Copilot、Droid、Pi Agent，以及任何读取 `.agents/skills/` 标准的智能体（Amp、Cline、Deep Agents、Firebender、Kimi Code CLI、Warp 等）。

---

## 四、核心概念速查

| 概念 | 说明 | 位置 |
| --- | --- | --- |
| **Spec（规范）** | 你的编码标准，用 Markdown 写。AI 写代码前先读 spec | `.trellis/spec/` |
| **Workspace（工作区）** | 每位开发者的会话日志，让 AI 记得上次做了什么 | `.trellis/workspace/` |
| **Task（任务）** | 一个工作单元，含需求文档与上下文配置 | `.trellis/tasks/` |
| **Skill（技能）** | 自动触发的工作流模块：brainstorm / before-dev / check / update-spec / break-loop | 各平台技能目录 |
| **Sub-agent（子代理）** | 专用 AI 子进程：`trellis-research` / `trellis-implement` / `trellis-check` | 各平台代理目录 |
| **Command（命令）** | 显式会话边界：`finish-work`、`continue`，部分平台需手动 `start` | 各平台命令目录 |
| **Hook（钩子）** | 自动触发脚本，在会话开始、子代理启动等时机注入上下文（仅支持 hook 的平台） | `.claude/hooks/` 等 |
| **Journal（日志）** | 记录每次开发会话做了什么的会话日志文件 | `.trellis/workspace/{name}/journal-N.md` |

---

## 五、目录结构

一个 Trellis 项目 = 仓库 + `.trellis/` + 一个或多个平台目录（如 `.claude/`、`.cursor/`、`.codex/`、`.opencode/`、`.kiro/`、`.pi/`）。

```text
.trellis/
├── spec/                       # Spec 库：团队约定、分包/分层规则、思维指南
│   ├── frontend/
│   │   ├── index.md            # 各目录入口文件
│   │   ├── components.md
│   │   └── state.md
│   ├── backend/
│   │   ├── index.md
│   │   ├── api.md
│   │   └── database.md
│   └── guides/
│       └── index.md
├── tasks/                      # 任务知识：PRD、设计、实现计划、研究、JSONL 清单、归档
│   ├── <MM-DD-name>/
│   │   ├── task.json           # 任务元数据与状态
│   │   ├── prd.md              # 需求文档（必有）
│   │   ├── design.md           # 技术设计（复杂任务）
│   │   ├── implement.md        # 执行计划（复杂任务）
│   │   ├── research/*.md       # 研究产出
│   │   ├── implement.jsonl     # 实现阶段上下文清单
│   │   └── check.jsonl         # 评审阶段上下文清单
│   └── archive/YYYY-MM/        # 已归档任务
├── workspace/                  # 工作区记忆：每位开发者的日志与索引
│   └── <name>/
│       ├── index.md
│       └── journal-N.md
├── workflow.md                 # 工作流契约：各状态的「下一步」提示块
├── config.yaml                 # 配置
├── scripts/                    # task.py / get_context.py 等脚本
└── .runtime/sessions/          # 运行时：会话 → 活动任务指针
```

---

## 六、安装与第一个任务

### 1. 安装 CLI

```bash
npm install -g @mindfoldhq/trellis@latest
```

### 2. 在项目里初始化

```bash
trellis init -u your-name
```

`-u your-name` 用于设置开发者身份（写入 `.trellis/.developer`），日志与任务归属会用到。初始化会生成 `.trellis/` 结构以及你所用平台的适配目录（`.claude/`、`.cursor/` 等）。

### 3. 维护 CLI 与项目文件

```bash
trellis upgrade            # 升级全局安装的 Trellis CLI
trellis update             # 把当前项目的 Trellis 文件同步到当前 CLI 版本
trellis update --dry-run   # 仅预览将要变更的文件
trellis update --migrate   # 执行迁移
```

### 4. 开始第一个任务

最简单的方式：**直接用自然语言把需求告诉 AI**。当 AI 识别到这是一个需要结构化处理的需求时，会触发 `trellis-brainstorm` 技能，**先征求你「是否创建任务」的同意**，然后逐条澄清需求并写出 `prd.md`。

> 注意：同意「创建任务」并不等于同意「开始实现」。开始实现前还有一道独立的「规划评审」闸门。

---

## 七、完整工作流（运行时全流程）

下面是从一次全新 AI 会话到任务归档的标准流程（以支持 hook 的平台如 Claude Code / Cursor 为例）：

### 1）会话开启 — 注入启动上下文

在支持 SessionStart 的平台，Trellis 注入一份**紧凑的启动负载**（是「索引 + 状态报告」，不是把所有 spec/task 全量倒出来），通常包含：

- 开发者身份（`.trellis/.developer`）
- Git 状态（当前分支、改动文件、最近提交）
- 活动任务指针（`.trellis/.runtime/sessions/*.json`）
- 活动任务列表（`.trellis/tasks/*/task.json`）
- 工作流索引（`.trellis/workflow.md` 的精简 Phase Index）
- Spec 索引路径（`.trellis/spec/**/index.md`）
- 工作区记忆（`.trellis/workspace/<name>/index.md` 与近期日志）

不同平台投递方式不同：Claude Code / Cursor / OpenCode / Gemini CLI / Qoder / CodeBuddy / Droid / Pi 用 hook 或插件自动注入；Codex 走 `AGENTS.md` 自动加载 + `UserPromptSubmit` hook 提醒；Copilot / Kiro / Kilo / Antigravity / Devin 则通过技能或工作流入口显式读取。

### 2）每条 Prompt — 注入当前工作流状态

在支持 hook 的平台，每条用户消息都会触发一次轻量的「工作流状态注入」，作为每轮护栏。解析链路：

```text
cwd → 找到 .trellis/ → 解析 session key
   → 读 .trellis/.runtime/sessions/<session-key>.json
   → 读 .trellis/tasks/<task>/task.json
   → 读 task.json.status
```

然后在 `.trellis/workflow.md` 里匹配对应状态块 `[workflow-state:STATUS]` 并注入当前轮。无活动任务时伪状态为 `no_task`。**要改工作流状态行为，改 `workflow.md`，而不是改 hook 脚本。**

### 3）当前轮分流（triage）

无活动任务时，AI 先对本轮分类，并在创建任何任务前征求同意：

| 路由 | 适用场景 | AI 会问什么 |
| --- | --- | --- |
| 简单对话 | 不写仓库；问答、解释、查询、讨论 | 默认不建任务；若有帮助才问是否要建任务 |
| 内联小任务 | 当前轮即可理解并验证的小改动 | 仅问是否要建 Trellis 任务；拒绝则本轮内联处理 |
| 完整 Trellis 任务 | 多文件、工作流/规范/平台改动、模板生成、需持久计划 | 解释为何建任务，并问是否可创建任务并进入规划 |

### 4）创建任务 — 写入规划状态

用户同意后，主会话执行：

```bash
python3 ./.trellis/scripts/task.py create "<title>" --slug <name>
```

生成任务目录（`task.json` + `prd.md` + 可能预置的 `implement.jsonl` / `check.jsonl`）。初始 `task.json` 状态为 `planning`，并尽力把当前会话的活动任务指针指向它。

### 5）规划 — 写出正确的产物

| 产物 | 何时需要 | 用途 |
| --- | --- | --- |
| `prd.md` | 每个任务 | 需求、约束、验收标准、范围外项 |
| `design.md` | 复杂任务 | 技术设计：边界、契约、数据流、兼容性、取舍、回滚 |
| `implement.md` | 复杂任务 | 执行计划：有序清单、验证命令、评审闸门、回滚点 |
| `research/*.md` | 需要调研时 | 规划期发现的持久事实 |
| `implement.jsonl` | 需要上下文清单时 | 实现阶段要读的 spec/research 文件 |
| `check.jsonl` | 需要上下文清单时 | 评审/验证阶段要读的 spec/research 文件 |

> 轻量任务可只有 PRD；复杂任务必须先有 `prd.md` + `design.md` + `implement.md` 才能开始。

### 6）上下文清单保持「窄而准」

`implement.jsonl` / `check.jsonl` 列出实现/评审前要读的稳定上下文文件：

```jsonl
{"file": ".trellis/spec/docs-site/docs/style-guide.md", "reason": "文档写作风格"}
{"file": ".trellis/tasks/04-30-example/research/platforms.md", "reason": "平台行为研究"}
```

规则：只列 spec 文件和任务研究文件；**不要列即将被修改的源码文件**；实现上下文放 `implement.jsonl`，验证/质量上下文放 `check.jsonl`。

### 7）激活 — 进入实现

```bash
python3 ./.trellis/scripts/task.py start <task-dir>
```

状态从 `planning` 变为 `in_progress`，下一轮会注入 `[workflow-state:in_progress]` 块。

### 8）实现 — 读任务产物与 spec

统一的上下文加载顺序：

```text
jsonl 条目 → prd.md → design.md（若有）→ implement.md（若有）
```

实现工作由 `trellis-implement` 子代理完成（**不做 git commit**）。Codex 有 `inline` 与 `sub-agent` 两种模式。

### 9）检查 — 评审并自修复

实现后运行 `trellis-check`：读 PRD / 设计 / 计划 / `check.jsonl` / 相关 spec / 改动文件，并跑本地 test、lint、type-check、format。它不仅报告问题，还**允许直接修复并重跑检查**。

### 10）收尾 — 沉淀可复用知识

检查通过后加载 `trellis-update-spec`：询问本次任务是否产生了可复用规则，如有则写进 `.trellis/spec/`。任务局部事实留在任务目录，稳定团队规则上升到 spec。

### 11）主会话驱动提交（commit）

提交边界与实现、与 `finish-work` 都是分开的。主会话会：读 `git status` → 区分本任务文件与无关脏文件 → 分组为逻辑提交 → 打印提交计划 → **等一次用户确认** → 执行 `git add` / `git commit`。

### 12）`/trellis:finish-work` — 归档与写日志

只有在 work commit 完成后才运行。它做账务性工作：若本任务仍有未提交改动则停止；把任务归档到 `.trellis/tasks/archive/YYYY-MM/`；把会话总结追加到 `journal-N.md`；更新工作区索引。**它不负责提交功能代码。**

### 会话之后留存下来的东西

| 存储位置 | 留存内容 |
| --- | --- |
| `.trellis/tasks/<task>/` | PRD、设计、实现计划、研究、上下文清单、元数据、归档历史 |
| `.trellis/spec/` | 团队约定与可复用经验 |
| `.trellis/workspace/<name>/` | 开发者日志与跨会话笔记 |
| Git 提交 | 可评审的代码/文档/spec/归档/日志变更单元 |

> 下一次 AI 会话重新读取仓库状态即可，**无需上一段聊天记录**就能知道任务是什么、适用哪些 spec、下一步该做什么。

---

## 八、命令、技能与子代理速查

Trellis 只暴露很小的「会话边界」命令面，其余都是自动触发技能或子代理——这是刻意设计：命令用于显式的用户边界，其余工作流自行运转。

### Slash 命令（按平台）

| 平台 | Start | Finish | Continue |
| --- | --- | --- | --- |
| Claude Code / OpenCode / Gemini CLI / CodeBuddy / Droid | 自动 SessionStart hook | `/trellis:finish-work` | `/trellis:continue` |
| Cursor / Pi Agent / Qoder | 自动 hook 或扩展 | `/trellis-finish-work` | `/trellis-continue` |
| Kiro | `@trellis:start` | `@trellis:finish-work` | `@trellis:continue` |
| Kilo | `/start.md` | `/finish-work.md` | `/continue.md` |
| Codex | 自动 `AGENTS.md` prelude（可选 hook） | 技能/prompt 入口 | 技能/prompt 入口 |

### 自动触发技能（按用户意图匹配，无需显式调用）

| 技能 | 触发时机 | 作用 |
| --- | --- | --- |
| `trellis-brainstorm` | 用户描述功能/Bug/模糊需求 | 产出任务 + `prd.md`，按需派生研究 |
| `trellis-before-dev` | 任务 in_progress 且即将写代码 | 读取该包相关 spec 文件 |
| `trellis-check` | 实现阶段结束 | diff 评审、lint/typecheck/test、自修复闭环 |
| `trellis-update-spec` | 出现值得沉淀的经验/决策/坑 | 在合适的 spec 文件追加条目 |
| `trellis-break-loop` | 刚解决一个棘手 Bug | 5 维度根因 + 预防分析 |

### 子代理（由主会话通过平台的 sub-agent / Task 原语派生）

| 子代理 | 角色 | 限制 |
| --- | --- | --- |
| `trellis-research` | 代码库/文档搜索 | 只读 |
| `trellis-implement` | 写代码 | 不做 `git commit` |
| `trellis-check` | 验证 + 自修复 | 自带重试闭环 |

> Kilo / Antigravity / Devin 没有真正子代理，相同工作在主会话内联完成。

---

## 九、task.py 任务脚本

`./.trellis/scripts/task.py` 是任务生命周期的核心脚本：

| 子命令 | 用途 |
| --- | --- |
| `create "title" [--slug name] [-a assignee] [-p priority]` | 创建任务（子代理平台会顺带预置 jsonl） |
| `add-context "$DIR" "<file>" "<reason>"` | 添加上下文条目（create 后填充 jsonl 的主要方式） |
| `validate "$DIR"` | 校验 JSONL |
| `list-context "$DIR"` | 查看所有上下文条目 |
| `start "$DIR"` | 设为当前 AI 会话/窗口的活动任务 |
| `finish` | 清除当前会话/窗口的活动任务 |
| `set-branch "$DIR" "feature/xxx"` | 设置分支名 |
| `set-base-branch "$DIR" "main"` | 设置 PR 目标分支 |
| `set-scope "$DIR" "auth"` | 设置范围 |
| `add-subtask <parent> <child>` / `remove-subtask` | 关联/解除子任务 |
| `archive <DIR>` | 归档任务 |
| `list [--mine] [--status <s>]` / `list-archive [YYYY-MM]` | 列出活动/归档任务 |

其他常用脚本：

```bash
# 上下文
./.trellis/scripts/get_context.py                 # 完整上下文
./.trellis/scripts/get_context.py --json          # JSON 输出
./.trellis/scripts/get_context.py --mode packages # 按包分层 spec（monorepo）
./.trellis/scripts/get_context.py --mode record   # 供 finish-work 使用

# 会话
./.trellis/scripts/add_session.py --title "..." --commit "..." --summary "..."

# Spec 引导（首次）
./.trellis/scripts/create_bootstrap.py
```

---

## 十、如何写出 AI 真正会遵守的 Spec

Spec 只有被 AI 遵守才有用。**模糊的指南会被忽略，具体的规则才会被执行。**

### 组织方式

放在 `.trellis/spec/`，按领域分目录，每个目录的 `index.md` 是入口（保持精简，细节链到其他文件）：

```text
.trellis/spec/
├── frontend/{index.md, components.md, state.md}
├── backend/{index.md, api.md, database.md}
└── guides/index.md
```

### 写「站得住」的规则

- **写文件路径**：不要说「错误处理工具」，要说 `src/lib/errors.ts`，AI 才能去读那个文件理解模式。
- **贴真实代码**：从你的项目里复制真实代码，AI 从真实示例学到的模式远胜于文字描述。
- **要有指令性**：说「该怎么做」，而不是「可以怎么做」；AI 需要方向，不是选项。
- **包含反模式**：明确写出「不要这样做」，防止常见错误。

### 测试与更新你的 Spec

写完后用一个相关任务测试 AI 是否遵守；不遵守就说明 spec 不够具体，补更多示例、路径、具体规则。

**不要在会话中途「口头告诉」AI 变化——去更新 spec 文件，那才是唯一可信源。** 发现新模式时，让 AI 运行 `trellis-update-spec` 技能捕获它；或在 `/trellis:finish-work` 收尾时，AI 会问你是否有新知识要加进 spec。

> 官方还提供可直接套用的技术栈模板：Electron + React + TypeScript、Next.js + oRPC + PostgreSQL、Cloudflare Workers + Hono + Turso 等。

---

## 十一、多平台与团队协作

- **团队共享**：`.trellis/spec/`、`.trellis/tasks/`、`.trellis/workflow.md` 等随 Git 版本管理，全团队共用同一套约定与任务知识。
- **个人记忆**：`.trellis/workspace/<name>/` 是每位开发者的私有日志区，互不干扰。
- **多工具一致**：同一套 `.trellis/` 结构 + 各平台适配目录，让 Claude Code、Cursor、Codex 等工具在同一约定下协作。
- **并行执行**：工作流对 git worktree 友好，多个任务可并行而减少分支混乱。

---

## 十二、典型使用场景

- **新产品/绿地项目**：用 spec 模板快速立约定，brainstorm 出第一批任务。
- **存量/棕地仓库**：用 `create_bootstrap.py` 从真实代码库引导出项目专属 spec。
- **重构**：把重构拆成多个任务，design.md 记录边界与回滚点，check 子代理护栏。
- **修 Bug**：解决后用 `trellis-break-loop` 做根因 + 预防分析，沉淀进 spec。
- **团队推广**：先在一个仓库跑通工作流，再把 spec 库共享给团队。

---

## 十三、最佳实践与常见误区

**最佳实践**

- Spec 写「路径 + 真实代码 + 指令 + 反模式」，并持续用 `trellis-update-spec` 更新。
- 让任务结构化：复杂任务先补齐 `prd.md` / `design.md` / `implement.md` 再开工。
- 上下文清单保持窄而准，不要把待改源码塞进 jsonl。
- 提交前认真看「提交计划」，区分本任务改动与无关脏文件。

**常见误区**

- ❌ 在会话里口头交代规则 → ✅ 改 spec 文件（唯一可信源）。
- ❌ 以为「同意建任务」=「同意开始实现」→ 实现前还有独立评审闸门。
- ❌ 把 `/trellis:finish-work` 当成提交命令 → 它只做归档/写日志，功能代码要先单独 commit。
- ❌ 改 hook 脚本来改工作流文案 → 应改 `.trellis/workflow.md`。

---

## 十四、结合本工作空间（MoreLogin）的落地建议

本工作空间（`AGENTS.md`）已说明：各仓库已引入 Trellis，`client` 的 `.trellis/spec` 已较完整地按真实栈（Yarn 1 + Vite + Umi Max + vitest + `window.CoreBrowser`）校准；但 harness 编排技能与 `selftest` / `sync:local` 等可运行脚本尚未搭建。结合 Trellis 工作流，建议：

1. **优先补齐各仓库 `.trellis/spec` 的「真实代码 + 路径」**：尤其 `kxc-player`、`rpa-editor`、`rpa-executor` 等复用组件，spec 越具体，跨产品复用时 AI 越不会跑偏。
2. **用 task 结构承接 PRD**：把 `prd-morelogin` 里的需求经 `trellis-brainstorm` 落成任务的 `prd.md`，复杂的跨仓库需求再补 `design.md`（标注影响面：client / client-cli / morelogin-mcp / 组件）。
3. **对齐工作空间约定的三个复核节点**：`AGENTS.md` 要求在 ①PRD 确认 ②组件同步前 ③Local API 改动前停下确认——这与 Trellis「建任务同意 / 规划评审闸门 / 提交确认」天然契合，可写进 `workflow.md` 的状态块。
4. **自测门槛对齐**：把「单测通过 + lint + 相关构建通过」写进 `check.jsonl` 与 `trellis-check` 流程，作为统一自测标准。

---

> 本指南依据 https://docs.trytrellis.app/ 官方文档整理。具体命令与字段以你本地安装的 Trellis 版本及 `.trellis/workflow.md`、`config.yaml` 为准。
