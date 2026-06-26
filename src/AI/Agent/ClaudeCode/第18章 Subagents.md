---
order: 18
date: 2026-06-26
---

# Subagents

## 什么是 Subagent

Subagent（子代理）是**预配置的 AI 角色**，Claude Code 可以将任务委派给它执行。每个 Subagent 有独立的上下文窗口、特定的系统提示词和受限的工具集。

当 Claude 遇到匹配 Subagent 专长领域的任务时，可以自动将任务委派，Subagent 独立工作并返回结果。

### 核心优势

| 优势 | 说明 |
|------|------|
| **上下文隔离** | Subagent 在自己的上下文中运行，不污染主会话 |
| **专业专长** | 用详细指令针对特定领域微调，指定任务成功率更高 |
| **可复用** | 创建后可跨项目使用，团队可共享 |
| **灵活的权限** | 不同 Subagent 可以有不同的工具访问级别 |

## 内置 Subagent

Claude Code 自带的内置子代理：

| Subagent | 用途 | 可用工具 |
|----------|------|----------|
| **Explore** | 快速、只读的代码库搜索和分析 | Read、Grep、Glob、LS |
| **Plan** | 在 Plan Mode 中研究代码库，不编辑代码 | Read、Grep、Glob、Bash |
| **General-purpose** | 复杂多步任务 | 全部工具 |

Plan Subagent 仅在 Plan Mode 下使用。在普通执行模式下，Claude 使用 General-purpose 或你自定义的 Subagent。

## 自定义 Subagent

### 存放位置

| 类型 | 位置 | 作用域 |
|------|------|--------|
| **Project** | `.claude/agents/` | 当前项目（优先级最高） |
| **User** | `~/.claude/agents/` | 所有项目 |
| **Plugin** | `<plugin>/agents/` | Plugin 启用的地方 |
| **CLI 动态** | `--agents` 标志 | 当前会话 |

### 文件格式

```markdown
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is simple and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)
```

### Frontmatter 字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 唯一标识符（小写字母和连字符） |
| `description` | 是 | 自然语言描述用途和调用时机 |
| `tools` | 否 | 逗号分隔的工具列表。不填继承主会话全部工具 |
| `model` | 否 | 模型：`sonnet`/`opus`/`haiku`/`inherit`。不填默认 `sonnet` |
| `isolation` | 否 | `worktree` 让 Subagent 在隔离 worktree 中运行 |

### 通过 `/agents` 管理

```
/agents
```

交互式界面支持：
- 查看所有可用 Subagent（内置、用户级、项目级）
- 创建新 Subagent（向导模式）
- 编辑工具权限
- 删除自定义 Subagent

### 通过 CLI 动态定义

```shell
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

优先级：Project > CLI > User。

## 使用方式

### 自动委派

Claude 会根据 Subagent 的 `description` 自动判断何时使用。在 `description` 中加入 "use PROACTIVELY" 或 "MUST BE USED" 增强自动触发的倾向。

### 显式调用

```
Use the code-reviewer subagent to check my recent changes
Have the test-runner subagent fix failing tests
```

## 何时使用 Subagent

### 适合用 Subagent 的场景

| 场景 | 信号 | 好处 |
|------|------|------|
| **研究密集型** | 需要读几十个文件来理解 | 主会话保持整洁 |
| **多个独立任务** | 子任务之间无依赖 | 并行执行 |
| **需要新视角** | 需要不带偏见的审查 | 干净上下文，客观反馈 |
| **验证后提交** | 提交前需要第二意见 | 捕捉熟悉度掩盖的问题 |
| **流水线** | 有明确的分阶段（设计→实现→测试） | 每个阶段独立专注 |

### 不适合用 Subagent 的场景

- 任务微小（读一两个文件即可解决）
- 所有子任务强依赖（需要串行）
- Subagent 的描述不够精确，Claude 无法判断何时调用

## 隔离模式

```yaml
isolation: worktree
```

Subagent 在独立的 git worktree 中运行，适合大规模批量修改。

## Subagent 的深度限制

整个 Subagent 调用链的最大深度为 **5 层**，防止无限递归嵌套。

> **资料来源**：[Claude Code 官方文档 - Subagents](https://code.claude.com/docs/en/sub-agents)
