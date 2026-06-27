---
order: 23
date: 2026-06-27
---

# Mobile、Remote Control 与 Dispatch

## Remote Control（`--remote-control` / `--rc`）

让**正在本地终端运行的交互会话**可从手机/浏览器继续。启动：

```shell
claude --remote-control
# 或
claude --rc
```

会话内会显示连接 URL/二维码。与 `--remote`（创建**新**云端会话）不同——Remote Control 绑定**已有本地会话**。

## Teleport（会话迁移）

| 方向 | 命令 |
|------|------|
| Web → 本地 | `claude --teleport` / `claude --tp` |
| 本地 → 新 Web 会话 | `claude --remote "Fix the login bug"` |

Teleport 在终端、Web（claude.ai/code）、Desktop、手机间迁移**同一会话**。

## Dispatch（Desktop）

Dispatch 从手机向 Desktop App 发送任务，Claude 创建 Desktop 会话自主完成。可选 **Computer Use**（见下）。与 Remote Control（继续已有终端会话）是不同路径。

## Computer Use（研究预览，CLI）

让 Claude 在 **macOS CLI** 上操作 GUI——打开 App、点击、输入、截屏。

| 条件 | 说明 |
|------|------|
| 计划 | **Pro / Max**（Team/Enterprise 不可用） |
| 平台 | **macOS only**（Windows 用 Desktop 版 Computer Use） |
| 版本 | Claude Code **v2.1.85+** |
| 模式 | **交互会话 only**（`-p` 不可用） |
| 认证 | 需 **claude.ai** 认证（第三方 API 提供商不支持） |

启用：交互会话中 `/mcp` → 找到内置 MCP server **`computer-use`** → Enable。首次使用需授予 macOS **Accessibility** 与 **Screen Recording** 权限。每个 App 需**会话级批准**；`Esc` 随时中止。

Claude 优先 MCP → Bash → Chrome → Computer Use（最慢、最宽）。

## 移动端

Claude iOS/Android 应用可：
- 通过 Remote Control 继续终端会话
- 接收 background 任务完成通知
- Dispatch 向 Desktop 发任务

## 平台迁移速查

| 路径 | 方式 |
|------|------|
| 本地终端 → 手机/浏览器 | `--remote-control` / `--rc` |
| 本地终端 → Desktop | `/desktop` |
| Web/手机 → 本地终端 | `claude --teleport` |
| Slack → 云端 coding | `@Claude` → claude.ai/code 会话 |

> **资料来源**：[Remote Control](https://code.claude.com/docs/en/remote-control)、[Computer Use](https://code.claude.com/docs/en/computer-use)、[Slack](https://code.claude.com/docs/en/slack)
