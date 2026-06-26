---
order: 21
date: 2026-06-26
---

# Background Agents 与定时任务

## Ctrl+B 后台化

在 Claude Code 交互模式中，按 `Ctrl+B` 将当前所有运行中的任务发送到后台，你可以继续其他工作。

## 后台代理

后台代理（Background Agents）可以在你关闭终端后继续运行。在 Desktop App 的 Code 标签页中可以查看和管理。

## `/loop` —— 循环执行

```
/loop 5m check if the deploy finished
```

在会话保持打开期间按指定间隔重复执行 prompt。适用于：
- 轮询部署状态
- 追踪 CI 运行
- 定期重新运行某个 Skill

间隔支持 `s`（秒）、`m`（分）、`h`（小时）。

## 定时任务（Scheduled Tasks）

### 云端定时任务

在 Anthropic 管理的基础设施上运行，计算机关机也能继续执行。

创建方式：
- Web 界面：[claude.ai/code](https://claude.ai/code)
- Desktop App
- CLI：`/schedule`

### 桌面定时任务

在你自己的机器上运行，直接访问本地文件和工具。

### `/schedule` 命令

创建定期重复的任务，如：
- 每日 PR 审查
- 过夜 CI 故障分析
- 每周依赖审计
- PR 合并后同步文档

## 通知系统

### 任务完成通知

Claude 完成任务后可以推送通知：
- 桌面通知
- 移动端通知（通过 Claude iOS/Android 应用）
- 你可以让 Claude 创建 Hook 通知在任务完成时播放声音

### 抑制通知

在机器前工作时，可通过 `CLAUDE_CLIENT_PRESENCE_FILE` 环境变量指向一个标记文件来抑制移动端推送通知：

```shell
export CLAUDE_CLIENT_PRESENCE_FILE=~/.claude/presence
```

### 通知类型的 Hook Matcher

| Notification Matcher | 触发时机 |
|---------------------|----------|
| `permission_prompt` | 需要权限确认 |
| `idle_prompt` | Claude 等待用户输入超过 60 秒 |
| `auth_success` | 认证成功 |
| `elicitation_dialog` | MCP 工具需要用户输入 |

## Agent View

v2.1.158 起提供 Agent View——查看所有 Claude Code 会话的统一列表：

```shell
claude agents
```

显示每个会话的状态：运行中、等待你、已完成。

> **资料来源**：[Claude Code 官方文档 - Scheduled Tasks](https://code.claude.com/docs/en/scheduled-tasks)、[Power User Tips](https://support.claude.com/en/articles/14554000-claude-code-power-user-tips)
