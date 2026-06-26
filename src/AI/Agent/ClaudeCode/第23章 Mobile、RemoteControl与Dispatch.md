---
order: 23
date: 2026-06-26
---

# Mobile、Remote Control 与 Dispatch

## Remote Control

Remote Control 让你从手机或其他设备继续本地会话。在任意设备上打开浏览器，连接到正在运行的 Claude Code 会话，继续工作。

## Teleport（会话传送）

在不同环境间**迁移**会话：

```shell
# 从 Web 会话拉回本地终端
claude --teleport

# 启动 Web 会话
claude --remote "Fix the login bug"
```

Teleport 让会话可以在终端、Web、Desktop App、手机之间自由迁移。

## Dispatch

Dispatch 让 Claude 在你离开时自主工作。从手机发送任务给 Dispatch，Claude 会创建 Desktop 会话并自主完成任务。支持 Computer Use 预览——Claude 可以操作你的屏幕。

## Computer Use（研究预览）

Computer Use 让 Claude 可以**直接操作你的计算机屏幕**——打开文件、运行开发工具、点击和导航，无须额外设置。

Pro 和 Max 用户可用。

## 移动端

Claude iOS/Android 应用可以：
- 连接交互式应用（实时图表、草图、可分享资产）
- 通过 Remote Control 继续终端中的会话
- 接收任务完成通知

## 会话不绑定于单一平台

Claude Code 的会话可以在以下平台间自由迁移：

| 路径 | 方式 |
|------|------|
| 本地终端 → 手机/其他设备 | Remote Control |
| 本地终端 → Desktop App | `/desktop` |
| Web/手机 → 本地终端 | `claude --teleport` |
| Slack → Pull Request | `@Claude` in Slack |

> **资料来源**：[Claude Code 官方文档 - Remote Control](https://code.claude.com/docs/en/remote-control)、[Computer Use](https://code.claude.com/docs/en/computer-use)
