---
order: 15
date: 2026-06-26
---

# MCP（Model Context Protocol）

## 什么是 MCP

Model Context Protocol（MCP）是由 Anthropic 制定的**开放标准**，用于将 AI 工具与外部数据源和工具连接。通过 MCP，Claude Code 可以读取 Google Drive 中的设计文档、更新 Jira 工单、拉取 Slack 数据，或使用你自己构建的工具。

## 三种传输方式

| 传输方式 | 说明 | 适用场景 |
|----------|------|----------|
| **stdio** | 标准输入输出，Claude Code 启动本地进程 | 本地命令行工具 |
| **HTTP+SSE** | HTTP 请求 + Server-Sent Events | 远程服务 |
| **Streamable HTTP** | HTTP 流式传输 | 云端服务、自托管 |

## 配置 MCP Server

### 用户级和项目级

MCP Server 配置存储在：
- 用户级：`~/.claude.json`
- 项目级：`.mcp.json`

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp"
    },
    "local-tool": {
      "type": "stdio",
      "command": "node",
      "args": ["./my-mcp-server.js"]
    }
  }
}
```

### 企业级 Managed MCP

管理员可通过 Managed Settings 批量部署 MCP 配置，以及通过 `allowedMcpServers` 和 `deniedMcpServers` 控制用户可用的 MCP Server。

### 命令行加载

```shell
claude --mcp-config ./mcp.json
claude --strict-mcp-config --mcp-config ./mcp.json  # 仅使用指定配置
```

## `/mcp` 命令

```
/mcp
```

显示所有已连接的 MCP Server 及其状态，包括每个 Server 提供的工具数量。零工具的 Server 会被标记。

## OAuth 认证

Claude Code 的 MCP OAuth 遵循 **RFC 9728**（Protected Resource Metadata）标准，自动发现授权服务器。OAuth 浏览器页面已升级为与 Claude Code 视觉风格统一，成功认证后自动关闭。

## 安全特性

- MCP Server 重连时不再向对话窗口刷屏工具列表，而是按 Server 前缀汇总
- `headersHelper` 支持多 Server 共用一个认证脚本（通过 `CLAUDE_CODE_MCP_SERVER_NAME` 和 `CLAUDE_CODE_MCP_SERVER_URL` 环境变量区分）
- 子代理可继承主会话的 MCP 工具
- `workspace` 是保留的 MCP Server 名称

## 社区 MCP 生态

截至 2026 年 6 月，社区已有 **770+ MCP Server** 可用，涵盖：

- 数据库（PostgreSQL、MySQL、SQLite、BigQuery）
- 云服务（AWS、GCP、Azure）
- 开发工具（GitHub、GitLab、Jira、Linear）
- 生产力（Google Drive、Slack、Notion、Airtable）
- 浏览器自动化（Puppeteer、Playwright）
- 文件系统、搜索引擎、API 封装等

## 自建 MCP Server 要点

1. 选择传输方式（stdio 最简单，HTTP 最灵活）
2. 实现标准的 MCP 协议接口
3. 在 `.mcp.json` 或 `~/.claude.json` 中注册
4. 通过 `/mcp` 验证连接和工具列表

> **资料来源**：[Claude Code 官方文档 - MCP](https://code.claude.com/docs/en/mcp)
