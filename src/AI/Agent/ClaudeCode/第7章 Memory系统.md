---
order: 7
date: 2026-06-26
---

# Memory 系统

## 两种记忆机制

Claude Code 有两套互补的记忆机制：

1. **CLAUDE.md（主记忆）**：你主动编写的，Claude 在每次会话启动时加载（见第 6 章）
2. **自动记忆（Auto Memory）**：Claude 从对话中自动提取的偏好和约定，跨会话持久化

## Auto Memory（自动记忆）

Auto Memory 是 Claude Code 的自动学习机制。Claude 在与你对话的过程中，自动提取有价值的偏好和知识，存储到自动记忆目录。

### 什么是 Auto Memory

- Claude 自动从对话中提取你的工作习惯和偏好
- 跨会话持久化，下次对话 Claude 自动回忆
- 与你手写的 CLAUDE.md 互补——Claude 自动发现你没写下来的约定

### 启用/禁用

Auto Memory 默认启用。可以通过以下方式控制：

```json
// settings.json
{ "autoMemoryEnabled": false }
```

或用环境变量：

```shell
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1
```

或在对话中：

```
/memory
```

### 自定义存储位置

```json
// 仅 user 和 managed 设置可配此项（project 和 local 不行，出于安全考虑）
{ "autoMemoryDirectory": "~/my-memory-dir" }
```

## 上下文管理

### `/compact` —— 上下文压缩

随着会话进行，Claude 的上文窗口会被越来越多的文件内容、工具结果填满。`/compact` 命令让 Claude 压缩当前对话上下文：

```
/compact
```

Claude 会将对话历史总结为精简版本，释放上下文空间，同时保留关键信息。

支持两种触发方式：
- **手动**：运行 `/compact`
- **自动**：上下文窗口快满时自动触发

Hooks 的 `PreCompact` 事件支持 `manual` 和 `auto` 两种 matcher，可以在压缩前执行自定义逻辑。

### 1M Token 上下文窗口

Opus 4.7、Opus 4.6、Sonnet 4.6 支持 1M token 上下文（详见第 4 章）。大上下文意味着更少的 Compact 需求和更长的自主任务执行能力。

### `/context` —— 查看上下文使用情况

```
/context
```

显示当前会话的上下文使用情况，帮助你了解剩余空间。

### `/cost` —— 查看费用

```
/cost
```

显示当前会话的 Token 消耗和费用估算。

## Auto-Dream（自动梦境）

Auto-Dream 是 Claude 在空闲时自动反刍记忆、自我改进的机制。Claude 会回顾之前的对话和自动记忆，优化未来会话中的表现。这是一个后台过程，无需用户干预。

> 此功能描述来自 [Power User Tips](https://support.claude.com/en/articles/14554000-claude-code-power-user-tips)，核心 Memory 文档中暂未详细展开。

## 会话持久化

### 会话保存

Claude Code 自动保存所有会话记录。会话文件存储在 `~/.claude/projects/` 下。默认保留 30 天，可通过 `cleanupPeriodDays` 配置：

```json
{ "cleanupPeriodDays": 60 }
```

### 会话恢复

```shell
# 恢复最近一次对话
claude --continue
claude -c

# 按 ID 或名称恢复指定会话
claude --resume <session-id>
claude -r <session-name>

# Fork 会话（创建新会话 ID 复用历史）
claude --resume abc123 --fork-session
```

### 离开后的会话概要（Away Summary）

当你离开终端几分钟后返回，Claude Code 会显示一行会话概要，让你快速了解之前进展。可以通过以下方式关闭：

```json
{ "awaySummaryEnabled": false }
```

## 相关命令速查

| 命令 | 功能 |
|------|------|
| `/init` | 自动生成项目的 CLAUDE.md |
| `/memory` | 编辑记忆文件 |
| `#` 快捷方式 | 快速添加记忆 |
| `/compact` | 压缩上下文 |
| `/context` | 查看上下文使用 |
| `/cost` | 查看 Token 费用 |
| `/recap` | 查看当前会话摘要 |

> **资料来源**：[Claude Code 官方文档 - Memory](https://code.claude.com/docs/en/memory)
