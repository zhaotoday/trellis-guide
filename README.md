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
14. [完整示例：用 Trellis 落地一个需求](#十四完整示例用-trellis-落地一个需求)
15. [进阶示例：一个复杂的跨层需求](#十五进阶示例一个复杂的跨层需求)

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

## 十四、完整示例：用 Trellis 落地一个需求

下面用一段**真实对话**演示从需求到归档的全过程。对话中 **🧑 我** 与 **🤖 AI** 来回交流，AI 在每一轮背后实际执行的动作（调用脚本、读写文件、跑子代理）以 `>` 引用块标注。

> **需求**：给一个博客网站的文章列表页增加「按标签筛选」功能——点击标签只显示带该标签的文章，URL 同步为 `?tag=xxx`，可分享/刷新保持。
>
> 假设这是一个 Next.js + TypeScript 项目，已执行过 `trellis init -u alice`。

---

🧑 **我**：文章列表页加一个按标签筛选的功能，点标签只显示该标签的文章，URL 要带上 `?tag=`，刷新后还在。

> 🤖 *AI 行为（会话开启时自动发生）*
> - Trellis 注入启动上下文：开发者身份、Git 状态、活动任务列表、`spec/**/index.md` 索引、工作区日志
> - 对本轮分流 → 判定为「完整 Trellis 任务」（多文件 + 路由 + 筛选逻辑）

🤖 **AI**：这个功能会改动列表页、路由参数和筛选逻辑，建议建一个 Trellis 任务并进入规划。是否允许我创建任务？（建任务 ≠ 开始写代码）

---

🧑 **我**：可以。

> 🤖 *AI 行为*
> - 创建任务：
>   ```bash
>   python3 ./.trellis/scripts/task.py create "文章列表按标签筛选" --slug filter-by-tag -p medium
>   ```
> - 生成 `.trellis/tasks/06-29-filter-by-tag/`，`task.json` 状态 = `planning`
> - 触发 `trellis-brainstorm`，逐条澄清后写出 `prd.md`

🤖 **AI**：任务已建好，PRD 草拟如下，请确认：

```markdown
# PRD：文章列表按标签筛选
## 需求
- 列表每篇文章展示其标签，标签可点击
- 点击标签 → 只显示含该标签的文章，URL 同步为 ?tag=<slug>
- 带 ?tag= 直接访问 / 刷新 → 筛选状态保持
- 提供「清除筛选」回到全部
## 验收标准
- 点击/直达均正确过滤，URL 更新；无匹配显示空状态；单测 + lint + build 通过
## 范围外
- 不做多标签组合；不做后端分页改造
```

---

🧑 **我**：可以，开始吧。

> 🤖 *AI 行为（实现前准备）*
> - 把要读的 **spec**（非待改源码）写进上下文清单：
>   ```bash
>   DIR=.trellis/tasks/06-29-filter-by-tag
>   python3 ./.trellis/scripts/task.py add-context "$DIR" ".trellis/spec/frontend/components.md" "组件与列表渲染约定"
>   python3 ./.trellis/scripts/task.py add-context "$DIR" ".trellis/spec/frontend/state.md" "URL 参数与状态同步约定"
>   ```
> - 激活任务：`task.py start "$DIR"` → 状态 `planning → in_progress`
> - 下一轮注入 `[workflow-state:in_progress]`，派生 `trellis-implement` 子代理（按 `jsonl → prd.md` 加载上下文，**不 commit**）写代码：
>   - `components/TagBadge.tsx`：可点击标签，跳转 `?tag=<slug>`
>   - `app/posts/page.tsx`：读 `searchParams.tag` 过滤列表 + 空状态 + 清除筛选
>   - `__tests__/filter-by-tag.test.tsx`：筛选逻辑与空状态单测
> - 派生 `trellis-check` 子代理，跑校验并自修复：
>   ```bash
>   npm test -- filter-by-tag && npm run lint && npm run build
>   ```

🤖 **AI**：功能已实现，单测 / lint / build 全部通过。实现中发现一个值得沉淀的规则——「列表筛选状态应走 URL searchParams，而不是组件内部 state」。我把它记进 spec 了：

```markdown
### 列表筛选状态约定（新增 → .trellis/spec/frontend/state.md）
列表/搜索类页面的筛选条件统一保存在 URL searchParams（如 ?tag=、?q=），
组件从 URL 读取而非内部 useState，保证可分享、可刷新、可前进后退。
```

接下来是提交计划，确认后我再 commit：

```bash
git add app/posts/page.tsx components/TagBadge.tsx __tests__/ .trellis/
git commit -m "feat(posts): 文章列表支持按标签筛选并同步 URL"
```

---

🧑 **我**：确认提交，然后收尾。

> 🤖 *AI 行为*
> - 执行上面的 `git add` / `git commit`
> - 运行 `/trellis-finish-work`：
>   - 归档任务到 `.trellis/tasks/archive/2026-06/06-29-filter-by-tag/`
>   - 把会话总结追加到 `.trellis/workspace/alice/journal-N.md`：
>     ```markdown
>     ## 2026-06-29 文章列表按标签筛选
>     - 列表支持点击标签筛选，状态同步到 URL ?tag=
>     - commit: feat(posts): 文章列表支持按标签筛选并同步 URL
>     - 沉淀：列表筛选状态一律走 URL searchParams
>     ```

🤖 **AI**：已提交并归档，工作区日志也更新好了。下次会话即使换个人、换个工具，也能从 spec 和 journal 里接着干。

---

### 接下来：如果要继续迭代这个需求

需求很少一次到位。下面演示**在上面这个任务基础上继续修改**，分两种情况。

#### 情况 A：任务还没收尾（仍是 `in_progress`）

直接继续对话即可，任务还是活动状态，每轮都在注入 `[workflow-state:in_progress]`。

🧑 **我**：再加一点，标签筛选要支持多选，URL 变成 `?tag=a,b`。

> 🤖 *AI 行为*
> - 当前任务仍是活动任务，无需新建
> - 比对原 PRD → 发现这命中了「范围外：不做多标签组合」，属于**范围变更**

🤖 **AI**：这超出了原 PRD 的范围（原本明确不做多标签组合）。你希望把多选扩进**当前任务**，还是**另开一个后续任务**？

🧑 **我**：就扩进当前任务。

> 🤖 *AI 行为*
> - **更新 `prd.md`**：把「多标签组合」从「范围外」挪到「需求」，并补验收标准
> - 让 `trellis-implement` 续写：URL 改为读写 `?tag=a,b`，组件支持多选切换
> - `trellis-check` 重跑校验

要点：**改的是 `prd.md` 这个唯一可信源**，不是口头追加；范围扩大时 AI 先提示、由你决定扩进本任务还是另开任务。

> 💡 中途关了会话、第二天想接着改：新开会话后运行 **`/trellis-continue`**，它会读 `task.json` + journal 把你带回任务当前状态，无需复述上下文。

#### 情况 B：任务已经 `finish-work` 归档了

任务已归档到 `.trellis/tasks/archive/...`，journal 也写过。继续修改走**后续任务**：

🧑 **我**：之前那个标签筛选，现在想再支持多选。

> 🤖 *AI 行为*
> - 识别为新需求，建后续任务：
>   ```bash
>   python3 ./.trellis/scripts/task.py create "标签筛选支持多选" --slug filter-multi-tag
>   ```
> - 新任务 `prd.md` 里引用归档任务作为背景
> - 可选：`task.py add-subtask <原任务> <新任务>` 建立父子关系
> - **关键**：上次沉淀进 `spec/frontend/state.md` 的「筛选状态走 URL searchParams」会**自动注入**，所以 AI 一开始就用 `?tag=a,b` 而非组件内部 state

> 为什么不「复活」旧任务：Trellis 的理念是**每个可独立验收的改动 = 一个任务**，归档任务是历史快照。新需求走新任务，历史清晰、可追溯。若只是修前一次的笔误/小 bug，则当作 inline 小改，可不建任务。

#### 情况 C：自测时发现 Bug，让 AI 继续修复

`trellis-check` 跑的是自动校验（test/lint/build）。但**你手动自测**时仍可能发现它没覆盖的问题——这时直接把现象告诉 AI 即可。

🧑 **我**：我点了标签 `react` 能正常筛选，但点「清除筛选」后 URL 回到了 `/posts`，列表却还是只显示 `react` 的文章，没刷新回全部。

> 🤖 *AI 行为（任务仍 `in_progress`，无需新建）*
> - 先**复现**：按你的路径定位到 `app/posts/page.tsx` 里清除筛选只改了 URL、没重新触发列表过滤
> - 让 `trellis-implement` 修复：清除时同步重置筛选状态 / 依赖 `searchParams` 重新渲染
> - **补一条回归单测**，覆盖「清除筛选 → 显示全部」这条路径（防止再次回归）
> - `trellis-check` 重跑 `npm test -- filter-by-tag && npm run lint && npm run build`

🤖 **AI**：复现到了，是清除筛选没重新读取 `searchParams` 导致的。已修复并补了回归单测，校验全部通过。你再自测一下这条路径？

🧑 **我**：好了，正常了。

> 🤖 *AI 行为（针对"棘手 bug"可选）*
> - 若这是个绕了一阵才定位的问题，触发 `trellis-break-loop` 做 5 维度根因 + 预防分析
> - 若分析出可复用的防错规则，经 `trellis-update-spec` 写进 spec，例如：
>   ```markdown
>   ### 筛选/清除一致性（新增 → .trellis/spec/frontend/state.md）
>   筛选与「清除筛选」必须共用同一份「从 URL 派生状态」的逻辑，
>   清除只改 URL、组件依赖 searchParams 重渲染，避免出现 URL 与列表不一致。
>   ```

几个要点：

- **自测发现的 Bug = 继续在当前任务里修**，不必新建任务（任务还没 finish 时）；
- AI 修 Bug 时应**顺手补回归测试**，把「人工自测才发现」的场景固化成自动校验；
- 真正棘手的 Bug 用 `trellis-break-loop` 复盘，把"为什么会犯""怎么预防"沉淀进 spec，下次同类问题自动规避；
- 若任务**已归档**后才自测出 Bug：小修当 inline 改；影响较大就建一个 `fix-xxx` 后续任务，流程同上。

#### 继续迭代决策表

| 你的情况 | 怎么做 |
| --- | --- |
| 任务还没收尾，继续改 | 直接对话；超原 PRD 范围则先更新 `prd.md` |
| 关了会话想接着改 | `/trellis-continue` |
| 自测发现 Bug（任务进行中） | 直接描述现象 → AI 复现修复 + 补回归测试；棘手则 `trellis-break-loop` 复盘 |
| 已归档，新增能力 | 建后续任务（可 `add-subtask` 关联原任务） |
| 已归档，只修小 bug | inline 小改，可不建任务 |

---

### 这个示例展示了什么

- **同意闸门**：建任务前、实现前、提交前各有一次人工确认；
- **结构化产物**：`prd.md`（必有）→ jsonl 上下文清单（复杂任务再加 `design.md` / `implement.md`）；
- **子代理分工**：research（只读）/ implement（不 commit）/ check（自修复）；
- **知识沉淀**：可复用规则上升到 `spec/`，会话痕迹留在 `workspace/`，下一次会话无需聊天记录即可接续。

---

## 十五、进阶示例：一个复杂的跨层需求

上一个例子是中等任务，PRD + jsonl 就够了。这一节演示一个**真正复杂的任务**——它会用到 Trellis 为复杂任务准备的全套产物：`research/`（调研）、`design.md`（技术设计与取舍）、`implement.md`（分阶段执行计划），以及 `trellis-research` 子代理和分阶段评审闸门。

> **需求**：给一个现有的 Next.js + TypeScript 应用从零加「用户认证」——邮箱注册/登录、JWT + 刷新令牌、受保护路由、登出，前后端打通，并要能平滑接入现有页面。
>
> 这类需求有**架构决策**（令牌存哪、刷新策略）、**多文件改动**、**安全考量**，属于必须先设计再动手的复杂任务。

---

🧑 **我**：给应用加一套用户认证，要支持邮箱注册登录、登出，登录态要能保持，部分页面要登录才能访问。

> 🤖 *AI 行为*
> - 启动上下文已注入；对本轮分流 → 判定「完整 Trellis 任务」且**复杂度高**（架构决策 + 安全 + 多文件）

🤖 **AI**：这是一个涉及架构决策和安全的复杂需求，我建议建任务并**先做调研和设计、确认后再动手**。可以吗？

---

🧑 **我**：可以，先调研和设计。

> 🤖 *AI 行为*
> - 建任务：
>   ```bash
>   python3 ./.trellis/scripts/task.py create "用户认证体系" --slug auth -p high
>   ```
> - 因为有不确定的技术选型，先派 **`trellis-research` 子代理（只读）**调研，产出 `research/auth-options.md`：
>   ```markdown
>   # 调研：认证方案选型
>   ## 候选
>   - A. 自建 JWT + 刷新令牌（access 存内存、refresh 存 httpOnly cookie）
>   - B. NextAuth.js / Auth.js（托管 session）
>   - C. 第三方（Clerk / Supabase Auth）
>   ## 现状约束
>   - 已有自建后端 API，团队希望令牌自控、不引第三方托管
>   ## 倾向
>   - 选 A：access token 放内存（防 XSS 持久化），refresh token 放 httpOnly + SameSite cookie（防 CSRF 配合）
>   ```
> - 触发 `trellis-brainstorm` 写 `prd.md`（需求 + 验收 + 范围外）
> - 因复杂度高，**额外写 `design.md` 与 `implement.md`**

🤖 **AI**：调研完成，倾向自建 JWT（不引第三方）。这是技术设计 `design.md`（节选），重点是令牌存储与刷新策略，请你确认：

```markdown
# 设计：用户认证体系

## 架构决策
- access token：存内存（JS 变量 / context），有效期 15min，不落 localStorage（防 XSS 持久窃取）
- refresh token：httpOnly + Secure + SameSite=Lax cookie，有效期 7d
- 刷新：access 过期 → 静默调用 /api/auth/refresh 换新 access；refresh 过期 → 跳登录

## 受保护路由
- 服务端在 middleware.ts 校验，未登录重定向 /login?from=

## 模块边界 / 数据流
- lib/auth/*（签发、校验、刷新）｜app/api/auth/*（register/login/logout/refresh）
- middleware.ts（路由守卫）｜components/AuthProvider.tsx（前端态）

## 兼容性 / 回滚
- 纯新增；回滚 = 摘掉 middleware 守卫与 AuthProvider，删 /api/auth/*
```

执行计划 `implement.md` 我拆成了 3 个阶段，每阶段之间设了评审点：

```markdown
# 执行计划：用户认证体系
## Phase 1 后端基础（可独立验证）
- [ ] lib/auth：密码哈希、JWT 签发/校验、refresh 轮换
- [ ] /api/auth/register、/login、/logout、/refresh
- [ ] 单测：签发/校验/过期/刷新
- 验证：npm test -- auth/api && npm run lint
## Phase 2 路由守卫 + 前端态
- [ ] middleware.ts 受保护路由重定向
- [ ] AuthProvider + useAuth；access 过期静默刷新
- 验证：npm test -- auth && npm run build
## Phase 3 接入现有页面
- [ ] 登录/注册页；把 /dashboard 等纳入保护
- [ ] 端到端手动自测清单
## 评审闸门：每个 Phase 结束后停下确认再进下一阶段
```

---

🧑 **我**：设计没问题，按这个分阶段来。

> 🤖 *AI 行为（填上下文清单 + 激活）*
> - 把设计/调研/相关 spec 写进清单：
>   ```bash
>   DIR=.trellis/tasks/06-29-auth
>   python3 ./.trellis/scripts/task.py add-context "$DIR" "$DIR/design.md" "认证架构与令牌策略"
>   python3 ./.trellis/scripts/task.py add-context "$DIR" "$DIR/research/auth-options.md" "选型依据"
>   python3 ./.trellis/scripts/task.py add-context "$DIR" ".trellis/spec/backend/api.md" "API 返回结构与错误码约定"
>   # check.jsonl 侧重安全/质量
>   python3 ./.trellis/scripts/task.py add-context "$DIR" ".trellis/spec/guides/security.md" "认证安全检查项" --target check
>   ```
> - `task.py start "$DIR"` → `in_progress`
> - **Phase 1**：派 `trellis-implement` 写 `lib/auth` + `/api/auth/*` + 单测（不 commit）；`trellis-check` 跑 `npm test -- auth/api && npm run lint`

🤖 **AI**：Phase 1（后端基础）完成，认证 API 与单测通过。按计划这里是评审点——要我继续 Phase 2（路由守卫 + 前端态）吗？

---

🧑 **我**：继续。

> 🤖 *AI 行为*
> - **Phase 2**：`middleware.ts` 守卫 + `AuthProvider/useAuth` + access 静默刷新；`trellis-check` 跑 `npm test -- auth && npm run build`
> - **Phase 3**：登录/注册页 + 把 `/dashboard` 纳入保护，并产出**手动自测清单**

🤖 **AI**：三个阶段都完成了。下面是给你的手动自测清单（自动测覆盖不到的交互路径）：

```text
□ 注册新账号 → 自动登录 → 进入 /dashboard
□ 登出 → 访问 /dashboard 被重定向到 /login?from=/dashboard
□ 登录后等待 access 过期 → 操作触发静默刷新，不掉登录
□ 清掉 refresh cookie → 下次刷新失败 → 跳登录
```

实现中沉淀了一条安全规则，我已记进 spec：

```markdown
### 令牌存储约定（新增 → .trellis/spec/guides/security.md）
access token 只存内存，绝不进 localStorage/sessionStorage；
refresh token 只用 httpOnly + Secure + SameSite cookie，前端 JS 不可读。
```

---

🧑 **我**：我自测过了，都正常。提交并收尾吧。

> 🤖 *AI 行为*
> - 打印提交计划，按阶段/逻辑分**多个 commit**（评审更友好）：
>   ```bash
>   git add lib/auth app/api/auth __tests__/auth && git commit -m "feat(auth): 后端认证 API 与 JWT 签发/刷新"
>   git add middleware.ts components/AuthProvider.tsx && git commit -m "feat(auth): 路由守卫与前端登录态"
>   git add app/login app/register app/dashboard .trellis/ && git commit -m "feat(auth): 登录/注册页与受保护页面接入"
>   ```
> - `/trellis-finish-work`：归档任务 + 写 journal

🤖 **AI**：已分三个提交合入并归档，安全规则也沉淀进 spec 了。

---

### 复杂示例比简单示例多了什么

| 维度 | 中等任务（示例十四） | 复杂任务（示例十五） |
| --- | --- | --- |
| 调研 | 一般不需要 | `trellis-research` 子代理 → `research/*.md` |
| 设计 | 跳过 | 必写 `design.md`（架构决策、取舍、回滚） |
| 计划 | 跳过 | 必写 `implement.md`（分阶段、带评审闸门） |
| 实现 | 一次实现 | **分 Phase**，每阶段独立验证 + 停下确认 |
| 提交 | 单个 commit | 按阶段/逻辑**多个 commit** |
| 沉淀 | 普通约定 | 安全等关键规则进 spec，check.jsonl 专挂安全检查 |

> 一句话：**任务越复杂，前期"设计/计划/调研"的产物越完整，实现越被拆成可独立验证、可随时停下复核的阶段。** 这正是 Trellis 控制复杂度、避免 AI 在大需求里跑偏的方式。

---

> 本指南依据 https://docs.trytrellis.app/ 官方文档整理。具体命令与字段以你本地安装的 Trellis 版本及 `.trellis/workflow.md`、`config.yaml` 为准。
