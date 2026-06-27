---
order: 13
date: 2026-06-27
---

# Skills

## 什么是 Skill

Skill 是 Claude Code 的**可复用指令集**。创建一个包含 `SKILL.md` 文件的目录，Claude 会将其加入自己的工具箱。当对话内容或你键入 `/skill-name` 与 Skill 匹配时，Claude 加载该 Skill 的指令并按照其指示行动。

Skill 遵循 [Agent Skills](https://agentskills.io/) 开放标准，同一个 `SKILL.md` 可以在 Cursor、Windsurf、Gemini CLI 等多个 AI 工具中使用。

### Skill vs 内建命令 vs Plugin

| 概念 | 定位 |
|------|------|
| **Skill** | 单个可复用指令集（一个 `SKILL.md` 文件） |
| **Slash Command** | 从 `.claude/commands/*.md` 或 Skill 的 `name` 生成的 `/` 命令 |
| **Plugin** | 多个 Skills + Hooks + MCP + Agents + Commands 的打包分发单位 |

自定义命令已合并进 Skills 体系——`.claude/commands/deploy.md` 和 `.claude/skills/deploy/SKILL.md` 都能生成 `/deploy` 命令。Skills 额外支持目录内附带文件、frontmatter 配置、自动加载等能力。

## Skill 的存放位置

| 位置 | 路径 | 生效范围 |
|------|------|----------|
| Enterprise | 由 Managed Settings 部署 | 全组织 |
| Personal | `~/.claude/skills/<name>/SKILL.md` | 所有项目 |
| Project | `.claude/skills/<name>/SKILL.md` | 当前项目 |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Plugin 启用的地方 |

同名 Skill 的优先级：Enterprise > Personal > Project。Plugin 的 Skill 使用 `plugin-name:skill-name` 命名空间，不会冲突。

### 嵌套目录自动发现

在 monorepo 中，Claude Code 会自动发现嵌套子目录中的 `.claude/skills/`。例如编辑 `packages/frontend/` 下的文件时，Claude 也会加载 `packages/frontend/.claude/skills/` 下的 Skill。

### 额外目录的 Skills

`--add-dir` 标志添加的目录中的 `.claude/skills/` 也会被自动加载，且支持**实时变更检测**——编辑 Skill 后无需重启会话即可生效。

## SKILL.md 结构

```markdown
---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works.
disable-model-invocation: true
allowed-tools: Read, Grep
model: sonnet
effort: high
---

When explaining code, always include:

1. **Start with an analogy**: Compare the code to something from everyday life
2. **Draw a diagram**: Use ASCII art to show the flow
3. **Walk through the code**: Explain step-by-step what happens
4. **Highlight a gotcha**: What's a common mistake or misconception?
```

### Frontmatter 全部字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 否 | 显示名称（也生成 `/name` 命令）。不填时使用目录名 |
| `description` | **推荐** | Skill 功能与触发时机；Claude 据此自动加载 |
| `when_to_use` | 否 | 补充触发场景；与 `description` **合计上限 1536 字符**（可用 `maxSkillDescriptionChars` 调整） |
| `argument-hint` | 否 | 补全时显示的参数提示，如 `[issue-number]` |
| `disable-model-invocation` | 否 | 设为 `true` 禁止 Claude 自动加载。仅你手动 `/name` 触发 |
| `user-invocable` | 否 | 设为 `false` 从 `/` 菜单隐藏。用于背景知识，不应直接调用 |
| `allowed-tools` | 否 | 该 Skill 激活时 Claude 可免确认使用的工具列表 |
| `model` | 否 | 该 Skill 激活时使用的模型 |
| `effort` | 否 | 该 Skill 激活时的 Effort 等级，覆盖会话等级（含 `max`） |
| `context` | 否 | 设为 `fork` 时在 Fork 子代理上下文中运行 |
| `agent` | 否 | `context: fork` 时指定子代理类型 |
| `hooks` | 否 | 该 Skill 生命周期的 Hooks |
| `paths` | 否 | Glob 模式，限制该 Skill 仅在操作匹配文件时激活 |
| `shell` | 否 | 行内命令使用的 Shell：`bash`（默认）或 `powershell`（需 `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`） |

### 变量替换

Skill 内容中支持以下变量：

| 变量 | 说明 |
|------|------|
| `$ARGUMENTS` | 调用时传入的全部参数 |
| `$ARGUMENTS[N]` / `$N` | 按索引访问参数 |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID |
| `${CLAUDE_SKILL_DIR}` | Skill 所在目录的绝对路径 |

## Skill 目录结构

```
my-skill/
├── SKILL.md          # 主指令文件（必须）
├── reference.md      # 详细参考文档（Claude 按需加载）
├── examples.md       # 示例（Claude 按需加载）
├── template.md       # Claude 填充的模板
└── scripts/
    └── validate.sh   # Claude 可执行的脚本
```

### 附带文件

`SKILL.md` 应保持在 500 行以内。大型参考文档放在独立文件中，由 `SKILL.md` 引用：

```markdown
## Additional resources
- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

## 描述预算与 `/skills`

Claude 启动时按 **2% context 预算**（默认）列出可用 Skills 供模型选择。预算不足时低优先级 Skill 可能只显示名称。调整方式：

```json
{
  "skillListingBudgetFraction": 0.02,
  "maxSkillDescriptionChars": 1536,
  "skillOverrides": {
    "low-priority-skill": "name-only"
  }
}
```

环境变量 `SLASH_COMMAND_TOOL_CHAR_BUDGET` 可设固定字符预算。用 `/skills` 查看已加载 Skills 及描述占用。

> Skills 专书：[01 管理架构](/AI/Skills/01-管理架构与快速上手.html) → [02 选型](/AI/Skills/02-选型与组合方案.html) → [03 基础](/AI/Skills/03-基础与生态概览.html)。

## 何时被调用

默认情况下，**你和 Claude 都可以调用**任何 Skill：

- **你调用**：输入 `/skill-name` 
- **Claude 自动加载**：当对话内容匹配 `description` 时

### 控制调用方

```yaml
# 仅你手动调用（适合有副作用的工作流）
disable-model-invocation: true

# 仅 Claude 可用（适合背景知识）
user-invocable: false
```

## 在子代理中运行

对于复杂的 Skill，可以在隔离的子代理上下文中运行：

```yaml
context: fork
agent: general-purpose
```

适用于需要大量上下文探索但不污染主会话的任务。

## Skill 内容类型

### 参考内容（Reference Content）

添加 Claude 在当前工作中应用的**知识**——约定、模式、风格指南、领域知识。内嵌运行，Claude 可以在对话上下文中在线使用。

```yaml
---
name: api-conventions
description: API design patterns for this codebase
---
When writing API endpoints:
- Use RESTful naming conventions
- Return consistent error formats
- Include request validation
```

### 任务内容（Task Content）

给 Claude **分步指令**执行特定操作——部署、提交、代码生成。通常是你手动触发的操作。

```yaml
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
---
Deploy the application:
1. Run the test suite
2. Build the application
3. Push to the deployment target
```

## 限制 Skill 的路径范围

使用 `paths` 字段限制 Skill 仅在匹配的路径下激活：

```yaml
paths: src/api/**, src/types/**
# 或 YAML 列表格式
paths:
  - src/api/**
  - src/types/**
```

## 社区 Skills 生态

截至 2026 年 6 月，社区已有 **4200+ Skills** 可供安装。热门 Skill 包括：

| Skill | 安装量 | 用途 |
|-------|--------|------|
| frontend-design（Anthropic 官方） | ~300K | 前端设计指南 |
| vercel-react-best-practices | ~320K | React 最佳实践 |
| web-design-guidelines（Vercel） | ~256K | Web 设计规范 |
| find-skills（Vercel） | ~1.1M | 搜索和发现 Skills |
| caveman | ~48K | 简化代码复杂度 |

通过 `/plugin` 命令安装社区 Skills（或通过 Plugin 市场安装——见第 16 章）。

> **资料来源**：[Claude Code 官方文档 - Skills](https://code.claude.com/docs/en/skills)、[Agent Skills 标准](https://agentskills.io/)
