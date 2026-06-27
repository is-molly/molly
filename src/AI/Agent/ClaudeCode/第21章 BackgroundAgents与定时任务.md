---
order: 21
date: 2026-06-27
---

# Background Agents 与定时任务

官方将「后台运行」和「定时自动化」分为多层，不要混为一谈。

## Ctrl+B — 后台化当前任务（会话内）

**不是**把整个 Claude Code 会话 detach 到后台。

`Ctrl+B` 将**当前正在运行的 Bash 命令或 subagent** 移到后台，主对话可继续。Tmux 用户需按两次（prefix 冲突）。用 `/tasks`（别名 `/bashes`）查看会话内后台项。

> 官方：[Keyboard shortcuts](https://code.claude.com/docs/en/keybindings) — “Backgrounds bash commands and agents”

## `/background` — 整个会话后台化

将整个 Claude Code 会话 detach 为 **background agent**，释放当前终端：

```
/background
/background 继续跑测试并告诉我结果
/bg
```

或启动时：

```shell
claude --bg "investigate the flaky test"
```

用 `claude agents` 监控；`claude attach <id>` 重新 attach。

## Agent View（研究预览）

v2.1.158+ 的 `claude agents` 列出所有 background 会话（运行中、等待输入、已完成）。v2.1.144+ 在 `/resume` 选择器中 background 会话标记为 `bg`。

## `/loop` — 会话内循环

```
/loop 5m check if the deploy finished
```

在**当前会话保持打开**期间按间隔重复执行 prompt。7 天过期；`--resume` 可恢复。禁用：`CLAUDE_CODE_DISABLE_CRON=1`。

适用：轮询部署、追踪 CI。与 Routines（云端）不同——**计算机关闭则停止**。

## 定时自动化三层

| 层级 | 入口 | 运行位置 | 计算机关机 |
|------|------|----------|------------|
| **Routines** | `/schedule`、`/routines`、claude.ai/code/routines | Anthropic 云端 | 可继续 |
| **Desktop 定时任务** | Desktop App UI | 本机 | 需开机 |
| **`/loop`** | 当前 CLI 会话 | 本机 | 停止 |

### Routines（研究预览）

`/schedule` 创建 **Routines**——在 Anthropic 托管基础设施上自主运行（无权限弹窗）。触发方式：Schedule、API POST、GitHub 事件。Team/Enterprise 可由 admin 禁用。

### Desktop 定时任务

在 Desktop App 的 Code 标签创建，直接访问本地文件，机器需保持运行。

## 通知

Claude 完成任务后可推送桌面/移动端通知。抑制移动端推送：

```shell
export CLAUDE_CLIENT_PRESENCE_FILE=~/.claude/presence
```

Hook `Notification` matcher 包括：`permission_prompt`、`idle_prompt`、`auth_success`、`elicitation_dialog` 等。

## 对比速查

| 操作 | 命令/快捷键 | 作用对象 |
|------|-------------|----------|
| 后台化 subagent/Bash | `Ctrl+B` | 会话内单项任务 |
| 后台化整个会话 | `/background`、`claude --bg` | 整个 Claude Code 进程 |
| 查看会话内后台 | `/tasks` | subagent、Bash、workflow |
| 查看所有 background 会话 | `claude agents` | 跨终端会话 |
| 云端定时 | `/schedule` | Routines |
| 会话内轮询 | `/loop` | 当前终端会话 |

> **资料来源**：[Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)、[Routines](https://code.claude.com/docs/en/routines)、[Agent view](https://code.claude.com/docs/en/agent-view)、[Commands](https://code.claude.com/docs/en/commands)
