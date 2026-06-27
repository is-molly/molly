---
order: 6
date: 2026-06-27
---

# 工具链 · 元 Skill 与安装器

> **章节类型：工具链**（用来装、造、发布 Skill 的工具，不是业务工作流）。含 skill-creator、npx skills、valorisa。

"元 Skill"是指那些用来管理、发现和创建其他 Skill 的 Skill。它们是 Skill 生态的基石工具。

## skill-creator（Anthropic 官方 Plugin）

### 项目概述

- **来源**：[Claude Code 官方 Plugin Marketplace](https://claude.com/plugins/skill-creator)
- **定位**：创建、评估、优化、基准测试 Skill 的完整工具链
- **CC Switch**：不能代装（Plugin + 子 Agent，须 Claude Code `/plugin`）
- **Cursor**：可用内置 `/create-skill` 代替 Create 流程，不必装本 Plugin

2026 版 skill-creator 提供 **四模式**：

| 模式 | 用途 |
|------|------|
| **Create** | 交互式创建 Skill（需求引导 → 结构 → 生成 → 验证 → 打包） |
| **Eval** | 生成测试用例，验证 Skill 是否在正确场景触发 |
| **Improve** | 基于 eval 结果优化 description 和指令 |
| **Benchmark** | 多轮 A/B 对比 Skill vs 无 Skill 基线，含方差分析 |

底层有四个专用子 Agent：Executor（执行）、Grader（评分）、Comparator（盲测 A/B）、Analyzer（改进建议）。

### 安装

> 多路径规则 → [05 · 多路径安装](05-安装方式参考手册.md#多路径安装怎么选装完怎么验)。**无 npx 路径**；与 [valorisa Skill Creator](#valorisa-claude-skill-creator社区增强版) **二选一**，不要两个 Creator Plugin 都装。

- **推荐（官方 skill-creator）**：日常建 Skill、Eval/Benchmark；Claude Code `/plugin` 唯一路径。
- **备选（valorisa 社区版）**：发布给他人前做安全审查时用 → [valorisa Skill Creator](#valorisa-claude-skill-creator社区增强版)。

```
# 推荐 — 官方 Market
/plugin install skill-creator@claude-plugins-official
/reload-plugins
/skills
/skill-creator
/doctor
```

**排错** — 报找不到包时：

```
/plugin marketplace update claude-plugins-official
# 或
/plugin marketplace add anthropics/claude-plugins-official
```

然后重试 `/plugin install skill-creator@claude-plugins-official`（见 [官方文档](https://code.claude.com/docs/en/skills)）。

### 使用方式

```
/skill-creator

# 或直接说：
Help me create a skill that enforces conventional commits in my project
Evaluate my summarize-changes skill with skill-creator
Benchmark my deploy skill across 10 runs
```

### Create 模式交互流程

Menu Interview 阶段会引导你明确：

| 维度 | 示例 |
|------|------|
| Purpose | code generation / workflow / analysis |
| Trigger style | Slash command / Auto-detect / Both |
| Tools needed | File system / Shell / Web / MCP |
| Output format | Code / Text / Structured / Files |
| Complexity | Simple / Moderate / Complex |
| Dependencies | 其他 Skill 或 MCP |
| Persistence | 是否需要跨会话记忆 |

### 注意事项

skill-creator 生成的 Skill 是**起点**，不是终点：

- 先用 skill-creator 生成初版
- 用 **Eval 模式**验证触发准确率
- 在实际使用中收集失败点，更新 Gotchas 区
- 迭代 3-5 次后成为成熟 Skill

## `npx skills` — Vercel Labs 的通用 Skill 包管理器

### 项目概述

- **仓库**：[vercel-labs/skills](https://github.com/vercel-labs/skills)
- **注册表**：[skills.sh](https://skills.sh)
- **定位**：跨平台 Skill 安装、发现、更新 CLI
- **支持平台**：Claude Code、Cursor、Codex CLI、Gemini CLI、OpenCode、Windsurf、Copilot CLI 等 20+ 平台

### 推荐工作流（装单体 Skill）

> 多路径规则 → [05 · 多路径安装](05-安装方式参考手册.md#多路径安装怎么选装完怎么验)

- **推荐（CC Switch 用户）**：`cc-switch skills install owner/repo:<目录名>` → `skills sync`（**无** `--skill`）。
- **备选（终端 / 无 CC Switch）**：`npx skills add … -g -a claude-code -y`。
- **不要两条都跑**（同一 Skill 混装会占双份描述预算）。

```bash
# 1. 发现
npx skills find "code review"

# 2. 看仓库里有哪些 Skill（不安装）
npx skills add owner/repo --list

# 3. 安装 — CC Switch（推荐）
cc-switch skills install mattpocock/skills:grill-me && cc-switch skills sync

# 3. 安装 — npx（备选）
# npx skills add mattpocock/skills --skill grill-me -g -a claude-code -y

# 4. 收尾（Claude Code 会话内）
# /skills → /doctor
```

**不要**对陌生大仓库直接用 `--all`（会一次装进几十个 Skill，描述预算易爆）。需要多个 Skill 时逐个 `--skill` 安装。

### 命令参考

```bash
npx skills find [query]          # 发现
npx skills add owner/repo --list # 只列不装
npx skills add owner/repo --skill <name> -g -a claude-code -y
npx skills add owner/repo -a claude-code -a cursor   # 多 Agent
```

### 技术原理

1. 解析仓库 URL（支持 `owner/repo` 简写）
2. 从 GitHub 获取 Skill 目录结构（支持 `skills/`、` .claude/skills/` 等多种布局）
3. 写入 `.agents/skills/` 中心目录
4. 按 `-a` 参数 symlink/copy 到各 Agent 目录

**管理**

```bash
npx skills list
npx skills update
npx skills remove <skill-name>
npx skills init my-skill
npx skills use owner/repo@skill   # 临时用，不写入磁盘
```

## valorisa/Claude-Skill-Creator（社区增强版）

### 项目概述

- **仓库**：[valorisa/Claude-Skill-Creator](https://github.com/valorisa/Claude-Skill-Creator)
- **定位**：社区增强版 Skill Creator，增加 llm-council 多模型审查
- **协议**：MIT

与官方 skill-creator 的差异：

| 阶段 | 社区版增强 |
|------|-----------|
| Menu interview | 增加安全边界询问（文件删除、网络请求等） |
| Generate | 自动检测危险指令模式 |
| Validate | llm-council 多模型交叉审查 |
| Package | 额外安全检查清单 |

### 安装

> 多路径规则 → [05 · 多路径安装](05-安装方式参考手册.md#多路径安装怎么选装完怎么验)。与 [skill-creator](#skill-creatoranthropic-官方-plugin) **二选一**。

- **推荐（官方 skill-creator）**：日常开发、Eval/Benchmark、与 Claude Code 同步更新。
- **备选（本 Plugin）**：发布 Skill 给他人前，用 llm-council 做安全审查。
- CC Switch **不能**代装 Plugin。

```
# 备选 — 作者 Market（发布前安全审查时用）
/plugin marketplace add valorisa/Claude-Skill-Creator
/plugin install claude-skill-creator@Claude-Skill-Creator
/reload-plugins
/skills
/doctor
```

### 何时选官方 vs 社区版

- 日常开发选**官方 skill-creator**——集成 Eval/Benchmark，与 Claude Code 同步更新
- 发布 Skill 给他人使用前，用**社区版**做一次安全审查

## agentskills 规范验证工具

[agentskills/agentskills](https://github.com/agentskills/agentskills) 仓库提供开放标准的验证 CLI，可检查 Skill 是否符合 [agentskills.io](https://agentskills.io/specification) 规范（name/description 必填、目录结构等）。

**推荐** — 发布跨平台 Skill 前，在 Skill 目录执行（以仓库当前 README 为准）：

```bash
git clone --depth 1 https://github.com/agentskills/agentskills.git
# 按仓库 README 运行 validate 子命令，指向你的 SKILL.md 目录
```

与 `skill-creator` 的 Eval 模式互补：规范校验管格式，Eval 管触发准确率。

---

> **本章资料来源**：[Anthropic skill-creator Plugin](https://claude.com/plugins/skill-creator)、[Claude Code Skills 文档](https://code.claude.com/docs/en/skills)、[vercel-labs/skills](https://github.com/vercel-labs/skills)、[agentskills.io 规范](https://agentskills.io/specification)
