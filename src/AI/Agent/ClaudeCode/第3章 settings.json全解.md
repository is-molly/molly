---
order: 3
date: 2026-06-26
---

# settings.json 全解

## 配置体系

Claude Code 的配置通过**层级化**的 settings.json 文件管理。每一层有不同的生效范围和控制力。

### 四个配置作用域

| 作用域 | 存储位置 | 影响范围 | 是否团队共享 |
|--------|----------|----------|--------------|
| **Managed** | 服务器下发、MDM/OS 策略、或系统级 `managed-settings.json` | 整机所有用户 | 是（IT 统一部署） |
| **User** | `~/.claude/settings.json` | 所有项目 | 否 |
| **Project** | `.claude/settings.json` | 当前项目的所有协作者 | 是（可提交 git） |
| **Local** | `.claude/settings.local.json` | 仅你、仅当前项目 | 否（自动 gitignore） |

### 层级优先级（从高到低）

```
1. Managed 策略         ← 最高，不可被覆盖
2. 命令行参数           ← 临时会话覆盖
3. Local 设置           ← 覆盖项目和用户设置
4. Project 设置          ← 覆盖用户设置
5. User 设置            ← 最低，其他未覆盖时生效
```

对于**权限规则**（`permissions.allow` / `permissions.deny` / `permissions.ask`），不同作用域会**合并**而非覆盖。其他配置键遵循覆盖规则。

### Managed 策略的多种交付方式

企业可以通过以下方式集中管理配置（均使用相同的 JSON 格式，优先级最高不可被用户覆盖）：

| 交付方式 | 平台 | 说明 |
|----------|------|------|
| 服务器下发 | 全平台 | 通过 claude.ai 管理后台下发 |
| MDM / OS 策略 | macOS / Windows | macOS: `com.anthropic.claudecode` plist；Windows: `HKLM\SOFTWARE\Policies\ClaudeCode` 注册表 |
| 文件部署 | 全平台 | macOS: `/Library/Application Support/ClaudeCode/`；Linux: `/etc/claude-code/`；Windows: `C:\Program Files\ClaudeCode\` |

Managed 设置还支持**分片部署**——在 `managed-settings.d/` 目录下放置多个 `.json` 文件，不同团队可以独立部署各自的策略片段：

```
/Library/Application Support/ClaudeCode/
├── managed-settings.json         ← 基础策略（先合并）
└── managed-settings.d/
    ├── 10-telemetry.json         ← 按数字前缀排序合并
    ├── 20-security.json
    └── 30-tools.json
```

### 各地存储对应关系

| 配置类型 | User 级 | Project 级 | Local 级 |
|----------|---------|------------|----------|
| Settings | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| Subagents | `~/.claude/agents/` | `.claude/agents/` | 无 |
| MCP Servers | `~/.claude.json` | `.mcp.json` | `~/.claude.json`（按项目） |
| Plugins | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| CLAUDE.md | `~/.claude/CLAUDE.md` | `CLAUDE.md` 或 `.claude/CLAUDE.md` | `CLAUDE.local.md` |

## settings.json 示例

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Read(~/.zshrc)"
    ],
    "deny": [
      "Bash(curl *)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)"
    ]
  },
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp"
  },
  "companyAnnouncements": [
    "Welcome to Acme Corp! Review our code guidelines at docs.acme.com",
    "Reminder: Code reviews required for all PRs",
    "New security policy in effect"
  ]
}
```

在 settings.json 顶部添加 `$schema` 行可以在 VS Code 等支持 JSON Schema 的编辑器中获得自动补全和验证：

```json
"$schema": "https://json.schemastore.org/claude-code-settings.json"
```

## 核心配置项分类

### 权限与安全

| 配置键 | 说明 |
|--------|------|
| `permissions.allow` | 预先批准的工具调用规则，匹配的工具不再弹确认框 |
| `permissions.deny` | 预先禁止的工具调用规则 |
| `permissions.ask` | 强制弹出确认框的工具规则 |
| `permissions.defaultMode` | 默认权限模式 |
| `allowManagedPermissionRulesOnly` | （仅 Managed）禁止用户和项目定义权限规则 |
| `sandbox` | 沙箱配置块，详见第 5 章 |

**权限规则语法**：

```
ToolName(pattern)
```

- `Bash(npm run *)`：匹配所有 `npm run` 开头的命令
- `Read(./.env)`：精确匹配 `.env` 文件
- `Read(./.env.*)`：匹配 `.env.local`、`.env.production` 等
- `Read(./secrets/**)`：匹配 secrets 目录下所有文件
- `Edit`：匹配所有 Edit 操作
- `*`：匹配所有工具

### 模型与行为

| 配置键 | 说明 |
|--------|------|
| `model` | 默认模型 |
| `effortLevel` | 全局 Effort 等级：`low`/`medium`/`high`/`xhigh`（`max` 仅会话级，不在此设置） |
| `alwaysThinkingEnabled` | 全局启用 Extended Thinking |
| `availableModels` | （Managed）限制用户可选的模型列表 |
| `autoScrollEnabled` | 全屏模式下是否自动跟随输出滚动到底部。默认 `true` |
| `awaySummaryEnabled` | 离开终端几分钟后返回时显示一行会话概览。默认 `true` |

### 更新与通道

| 配置键 | 说明 |
|--------|------|
| `autoUpdatesChannel` | 更新通道：`"latest"`（默认）或 `"stable"` |
| `cleanupPeriodDays` | 会话文件保留天数，过期自动清理。默认 30 天 |

### 内存与记忆

| 配置键 | 说明 |
|--------|------|
| `autoMemoryEnabled` | 启用/禁用 Auto Memory。默认 `true` |
| `autoMemoryDirectory` | 自定义 Auto Memory 存储目录。只接受来自 policy 和 user 的设置 |
| `claudeMdExcludes` | 跳过加载的 CLAUDE.md 文件 glob 模式 |

### Subagent 与 Agent

| 配置键 | 说明 |
|--------|------|
| `agent` | 将主会话作为命名子代理运行，应用该子代理的 system prompt 和工具限制 |

### 插件与市场

| 配置键 | 说明 |
|--------|------|
| `enabledPlugins` | 已启用的插件列表 |
| `extraKnownMarketplaces` | 额外的插件市场注册 |
| `strictKnownMarketplaces` | （Managed）严格限制市场来源 |
| `blockedMarketplaces` | （Managed）封禁的市场列表 |

### 认证与 API

| 配置键 | 说明 |
|--------|------|
| `apiKeyHelper` | 动态生成 API Key 的脚本路径 |
| `awsAuthRefresh` | 刷新 AWS 凭证的脚本（Bedrock 用户） |
| `awsCredentialExport` | 导出 AWS 凭证 JSON 的脚本（Bedrock 用户） |

### 企业专属（Managed Only）

| 配置键 | 说明 |
|--------|------|
| `claudeMd` | 组织级 CLAUDE.md 内容注入 |
| `channelsEnabled` | 启用 Channels |
| `allowedChannelPlugins` | Channels 插件白名单 |
| `allowedMcpServers` | MCP Server 白名单 |
| `deniedMcpServers` | MCP Server 黑名单 |
| `allowedHttpHookUrls` | HTTP Hook 目标 URL 白名单 |
| `allowManagedHooksOnly` | 只加载 Managed Hooks |
| `allowManagedMcpServersOnly` | 只使用 Managed 的 allowedMcpServers 列表 |
| `companyAnnouncements` | 启动时显示的公司公告列表 |

### 环境变量注入

通过 `env` 块向 Claude Code 会话中注入环境变量：

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "1"
  }
}
```

常用环境变量详见附录 B。

### 归属与签名

| 配置键 | 说明 |
|--------|------|
| `attribution.commit` | 自定义 commit 归属信息 |
| `attribution.pr` | 自定义 PR 归属信息 |

## `/config` 命令

Claude Code 提供了一个交互式设置界面，通过以下命令打开：

```
/config
```

自 v2.1.181 起，还支持直接在 prompt 中设置任意配置项：

```
/config key=value
```

例如：

```
/config thinking=false
/config autoUpdatesChannel stable
```

该语法在交互模式、`-p` 模式和 Remote Control 中均可用。

## 配置备份

Claude Code 会自动为配置文件创建带时间戳的备份，保留最近 5 个备份以防止数据丢失。

> **资料来源**：[Claude Code 官方文档 - Settings](https://code.claude.com/docs/en/settings)、[Permissions](https://code.claude.com/docs/en/permissions)
