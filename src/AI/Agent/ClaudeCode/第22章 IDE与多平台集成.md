---
order: 22
date: 2026-06-26
---

# IDE 与多平台集成

Claude Code 的引擎是统一的——CLAUDE.md、settings.json、MCP 配置在所有平台上通用。

## VS Code 扩展

安装：VS Code 扩展市场搜索 "Claude Code"，或为 Cursor 安装专用版本。

特性：
- **Inline Diffs**：IDE 内直接查看代码差异
- **@-mentions**：在对话中 @ 文件/文件夹，将上下文传递给 Claude
- **Plan Review**：审查和执行 Claude 的计划
- **对话历史**：查看和恢复之前的会话

启动：命令面板（`Cmd+Shift+P` / `Ctrl+Shift+P`）搜索 "Claude Code"，选择 **Open in New Tab**。

## JetBrains IDE

安装：[JetBrains Marketplace](https://plugins.jetbrains.com/plugin/27310-claude-code-beta-) 下载 Claude Code 插件。

支持 IntelliJ IDEA、PyCharm、WebStorm 等 JetBrains IDE，提供交互式 diff 查看和选择上下文共享。

权限模式切换和 CLI 相同：按 `Shift+Tab` 循环。

## Desktop App

独立桌面应用，运行 Claude Code 无需 IDE 或终端。

**下载**：
- [macOS](https://claude.ai/api/desktop/darwin/universal/dmg/latest/redirect)（Intel + Apple Silicon）
- [Windows x64](https://claude.ai/api/desktop/win32/x64/setup/latest/redirect)
- [Windows ARM64](https://claude.ai/api/desktop/win32/arm64/setup/latest/redirect)

特性：
- **可视化 Diff**：图形界面审查代码变更
- **并排会话**：同时运行多个会话
- **定时任务**：计划和运行重复性任务
- **云端会话**：启动长时间运行的任务

从终端传送到 Desktop App：

```
/desktop
```

## Web 界面

在 [claude.ai/code](https://claude.ai/code) 浏览器中直接使用，无需本地安装。适用于：
- 处理本地没有的仓库
- 并行运行多个任务
- 启动长期任务后关闭浏览器、稍后回来查看

## Slack 集成

在 Slack 中 `@Claude`，附上 Bug 报告或功能描述，Claude Code 会生成 Pull Request。适用于：
- 将 Bug 报告从聊天工具直接转化为代码修复
- 团队协作中快速触发 Claude 任务

## Chrome 扩展（Beta）

Claude Code Chrome 扩展用于 Web 自动化和测试。通过 `--chrome` 标志启用：

```shell
claude --chrome
```

> **资料来源**：[Claude Code 官方文档 - Overview](https://code.claude.com/docs/en/overview)、[VS Code](https://code.claude.com/docs/en/vs-code)、[Desktop](https://code.claude.com/docs/en/desktop)、[Slack](https://code.claude.com/docs/en/slack)
