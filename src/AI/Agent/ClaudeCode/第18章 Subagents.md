---
order: 18
date: 2026-06-27
---

# Subagents

Subagent（子代理）是**预配置 AI 角色**——独立 context、专用 system prompt、可选工具限制。Claude 通过 **Agent 工具**或 **@-mention** 委派任务。

与 **background 整个会话**、**Agent Teams 多实例协作**不同：Subagent 在**单会话内**运行，结果汇总回主对话。并行多终端见 worktree + `claude agents`（第 17、21 章）。

## 内置 Subagent

| Subagent | 用途 | 工具 | 备注 |
|----------|------|------|------|
| **Explore** | 快速只读搜索 | Read, Grep, Glob, LS | **跳过** CLAUDE.md、git status |
| **Plan** | Plan Mode 下调研 | Read, Grep, Glob, Bash | 同上；thoroughness: quick/medium/very thorough |
| **General-purpose** | 复杂多步 | 全部 | 默认执行型 |
| **fork** | 继承完整对话的旁路任务 | 全部 | v2.1.161+；见下文 |

Explore/Plan 是 **one-shot**，无 agent ID，**不可 resume**；需持续对话用 custom 或 general-purpose。

## 自定义 Subagent

### 存放位置与优先级

| 类型 | 路径 | 优先级 |
|------|------|--------|
| Project | `.claude/agents/` | 最高 |
| User | `~/.claude/agents/` | 中 |
| Plugin | `<plugin>/agents/` | 命名空间 `plugin:agent` |
| CLI | `--agents '{...}'` | 仅当前会话 |
| Managed | policy 部署 | 随策略 |

**Project > CLI > User**（同名时）。

### 文件示例

```markdown
---
name: code-reviewer
description: Expert code reviewer. Use PROACTIVELY after code changes.
tools: Read, Grep, Glob, Bash
model: inherit
permissionMode: default
isolation: worktree
background: false
maxTurns: 30
effort: high
color: blue
---

You are a senior code reviewer...
```

### Frontmatter 字段

| 字段 | 说明 |
|------|------|
| `name` | 唯一 ID（小写+连字符） |
| `description` | 触发描述；加 PROACTIVELY/MUST BE USED 提高自动委派 |
| `tools` / `disallowedTools` | 工具白名单/黑名单 |
| `model` | `sonnet`/`opus`/`haiku`/`inherit`/`fable` |
| `permissionMode` | 覆盖 subagent 权限模式（含 `plan`） |
| `isolation` | `worktree` → 临时 git worktree |
| `background` | `true` → 总是后台运行 |
| `maxTurns` | 最大轮次 |
| `effort` | 含 `max` |
| `mcpServers` | 限制 MCP（Teammate 模式不完全应用） |
| `skills` | 预加载 skills |
| `hooks` | subagent 生命周期 hooks |
| `memory` | subagent 级 memory 配置 |
| `color` | UI 标识色 |

Subagent **默认 cwd = 主会话 cwd**；`cd` 不跨 Bash 调用持久。`isolation: worktree` 给独立仓库副本。

### `/agents` 与 CLI

```
/agents          # 查看、创建、编辑、删除
```

```shell
claude --agents '{"reviewer":{"description":"...","prompt":"...","tools":["Read","Grep"]}}'
```

主会话设 `"agent": "code-reviewer"` 可让整个会话以该 subagent 身份运行。

## 使用方式

### 自动委派

Claude 匹配 `description` 自动调用 Agent 工具。

### 显式调用

```
Use the code-reviewer subagent on my last commit
@code-reviewer review src/auth/
```

### 禁止某 subagent

```json
{ "permissions": { "deny": ["Agent(Explore)"] } }
```

或 `claude --disallowedTools "Agent(Explore)"`。

## Foreground vs Background

| 模式 | 行为 |
|------|------|
| **Foreground** | 阻塞等待结果 |
| **Background** | 并发执行；v2.1.186+ 权限弹窗** surfaced 到主会话** |

- 说「run in background」或 **`Ctrl+B`** 后台化运行中 subagent/Bash
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` 禁用
- `CLAUDE_CODE_FORK_SUBAGENT=1` 时**所有** spawn 走 background

## Fork Subagent（`/fork` 同类）

**Fork** 继承**完整对话历史**（system prompt、tools、model、messages），适合旁路任务无需重述背景：

- 通过 Agent 工具请求 `fork` 类型
- 工具调用不进主 transcript，仅**最终结果**返回
- Fork **不能再 fork**；其他 subagent 仍计 depth
- 可 `isolation: worktree` 隔离文件写入

会话内 **`/fork <directive>`**（v2.1.161+）是面向用户的 fork 入口。

## 调用深度与 Resume

- 最大深度 **5 层**（固定，不可配置）
- 第 5 层 subagent **无 Agent 工具**，不能再 spawn
- v2.1.187+：background subagent depth **spawn 时固定**，resume 不会变浅
- 完成后的 subagent 返回 **agent ID**，可 resume 继续（Explore/Plan 除外）
- `SendMessage` 到已停止 subagent 会**自动 background resume**

## 加载到 Subagent 的上下文

| 内容 | 是否加载 |
|------|----------|
| CLAUDE.md、rules、managed policy | ✓（Explore/Plan **除外**） |
| Git status 快照 | ✓（非 git 或 `includeGitInstructions: false` 时除外；Explore/Plan 除外） |
| MCP servers | ✓（来自 project/user settings） |
| 主会话对话历史 | ✗（fork 除外） |
| Spawn prompt | ✓ |

Explore/Plan 省略 memory 为提速；若规则必须生效（如「忽略 vendor/」），在委派 prompt 里**重申**。

## Plan Mode 与 Subagent

Plan Mode 下 Claude 委派 **Plan** subagent 调研；主会话保持只读。Teammate 可设 plan approval（第 19 章）。

## 何时用 / 不用

| 适合 | 不适合 |
|------|--------|
| 读大量文件、多模块探索 | 读 1–2 文件即可 |
| 独立并行子任务 | 强依赖串行上下文 |
| 客观 code review | description 模糊无法触发 |
| 流水线：设计→实现→测试 | — |

## Plugin Subagent

启用 plugin 后 typeahead 显示 `my-plugin:security-reviewer`；子目录 agent 可多级命名 `my-plugin:review:security`。

> **资料来源**：[Subagents](https://code.claude.com/docs/en/sub-agents)、[Commands - /fork](https://code.claude.com/docs/en/commands)
