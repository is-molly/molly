---
order: 30
date: 2026-06-26
---

# 附录B：环境变量参考

## API 与认证

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_API_KEY` | API Key 认证 |
| `ANTHROPIC_BASE_URL` | API 端点地址（第三方代理/Gateway） |
| `ANTHROPIC_MODEL` | 默认模型 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `opus` 别名解析为的模型 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `sonnet` 别名解析为的模型 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `haiku` 别名解析为的模型 |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | API Key 刷新间隔（毫秒） |

## Effort 与思考

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_EFFORT_LEVEL` | 全局 Effort 等级 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | 禁用 Adaptive Thinking（Opus 4.6/Sonnet 4.6） |
| `MAX_THINKING_TOKENS` | 手动控制思考 token 上限 |
| `CLAUDE_CODE_DISABLE_THINKING` | 全局禁用 Extended Thinking |

## 上下文与更新

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 禁用 1M 上下文 |
| `DISABLE_AUTOUPDATER` | 禁止自动更新 |

## 内存与行为

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用 Auto Memory |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | 从 `--add-dir` 目录加载 CLAUDE.md |

## 界面与终端

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN` | 禁用全屏渲染 |
| `CLAUDE_CODE_SESSION_ID` | Bash 工具子进程中的会话 ID |
| `CLAUDE_CODE_HIDE_ACCOUNT_INFO` | 隐藏邮箱和组织名（演示/录制用） |
| `CLAUDE_CODE_ENABLE_AWAY_SUMMARY` | 离开后显示会话概要 |
| `CLAUDE_CLIENT_PRESENCE_FILE` | 指向标记文件以抑制移动推送通知 |

## 功能开关

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 启用 Agent Teams |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | Windows 上启用 PowerShell 作为主 Shell |

## 平台特定

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_MCP_SERVER_NAME` | MCP headersHelper 脚本中的服务器名 |
| `CLAUDE_CODE_MCP_SERVER_URL` | MCP headersHelper 脚本中的服务器 URL |
| `CLAUDE_PROJECT_DIR` | Hook 中的项目根目录（Hook 内可用） |
| `CLAUDE_PLUGIN_ROOT` | Plugin Hook 中的 Plugin 根目录 |

## 可观测性

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | 启用遥测 |
| `OTEL_METRICS_EXPORTER` | OpenTelemetry 指标导出器 |
| `OTEL_LOG_TOOL_DETAILS` | 在 OTEL tool_result 中包含工具参数 |

> **资料来源**：[Claude Code 官方文档 - Settings](https://code.claude.com/docs/en/settings)
