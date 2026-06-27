---
order: 3
date: 2026-06-27
---

# settings.json 全解

Claude Code 通过**分层 settings.json** 管理行为。本章按作用域、优先级和主要配置键展开；完整字段见官方 [Settings reference](https://code.claude.com/docs/en/settings)。

## 配置体系

### 四个作用域

| 作用域 | 存储位置 | 影响范围 | 团队共享 |
|--------|----------|----------|----------|
| **Managed** | 服务器下发 / MDM / 系统目录 `managed-settings.json` | 整机 | IT 统一部署 |
| **User** | `~/.claude/settings.json` | 所有项目 | 否 |
| **Project** | `.claude/settings.json` | 当前仓库协作者 | 是（git） |
| **Local** | `.claude/settings.local.json` | 仅你、仅当前项目 | 否（应 gitignore） |

### 优先级（非权限规则）

```
Managed > 命令行参数 > Local > Project > User
```

**权限规则**（`permissions.allow/deny/ask`）在**各层合并**，不是简单覆盖。Managed 还可设 `allowManagedPermissionRulesOnly` 禁止用户/项目自定义规则。

### Managed 交付方式

| 方式 | 平台 | 路径/键 |
|------|------|---------|
| 服务器下发 | 全平台 | claude.ai Admin |
| MDM / GPO | macOS / Windows | `com.anthropic.claudecode` plist；`HKLM\SOFTWARE\Policies\ClaudeCode` |
| 文件部署 | 全平台 | macOS: `/Library/Application Support/ClaudeCode/`<br>Linux: `/etc/claude-code/`<br>Windows: `C:\Program Files\ClaudeCode\` |

**分片部署** `managed-settings.d/*.json` 按数字前缀排序合并。

### 配置与文件对应

| 类型 | User | Project | Local |
|------|------|---------|-------|
| Settings | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| Subagents | `~/.claude/agents/` | `.claude/agents/` | — |
| MCP | `~/.claude.json` | `.mcp.json` | — |
| Plugins | settings 内 `enabledPlugins` | 同左 | 同左 |
| CLAUDE.md | `~/.claude/CLAUDE.md` | `CLAUDE.md` / `.claude/CLAUDE.md` | `CLAUDE.local.md` |
| Rules | `~/.claude/rules/` | `.claude/rules/` | — |

## 快速开始

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": ["Bash(npm run *)", "Bash(git status)"],
    "deny": ["Read(./.env)", "Read(./secrets/**)"],
    "defaultMode": "acceptEdits"
  },
  "model": "sonnet",
  "effortLevel": "high",
  "autoUpdatesChannel": "latest"
}
```

## 核心配置项（按类别）

### 权限与安全

| 键 | 说明 |
|----|------|
| `permissions.allow` / `deny` / `ask` | 工具调用规则，语法 `Tool(pattern)` |
| `permissions.defaultMode` | `default` / `acceptEdits` / `plan` / `auto` / `dontAsk` / `bypassPermissions` |
| `allowManagedPermissionRulesOnly` | （Managed）仅策略层规则生效 |
| `sandbox.*` | 沙箱边界，见第 5 章 |
| `autoMode` | Auto 分类器的 `environment` / `allow` / `soft_deny` / `hard_deny` 规则 |
| `disableAutoMode` | `"disable"` 时从 Shift+Tab 移除 auto |
| `useAutoModeDuringPlan` | Plan 探索阶段是否用 auto 语义，默认 `true` |

**规则示例：**

```
Bash(npm run *)          # 前缀匹配
Read(./.env.*)           # glob
Edit                     # 所有 Edit
Agent(Explore)           # 禁止委派 Explore subagent
WebFetch(domain:example.com)
```

### 模型与 Effort

| 键 | 说明 |
|----|------|
| `model` | 默认模型别名或完整 ID |
| `effortLevel` | `low` / `medium` / `high` / `xhigh`（**不含 `max`**，max 仅会话级） |
| `availableModels` | 限制可选模型列表 |
| `enforceAvailableModels` | （Managed）强制仅列表内模型 |
| `advisorModel` | `/advisor` 使用的第二模型（v2.1.98+） |
| `alwaysThinkingEnabled` | 默认开启 Extended Thinking |

### Worktree

| 键 | 说明 |
|----|------|
| `worktree.baseRef` | `"fresh"`（默认，从 `origin/默认分支`）或 `"head"`（本地 HEAD，含未推送提交） |
| `worktree.bgIsolation` | Background 会话 worktree 隔离；`"none"` 关闭 |

### 记忆与 CLAUDE.md

| 键 | 说明 |
|----|------|
| `autoMemoryEnabled` | Auto Memory，默认 `true` |
| `autoMemoryDirectory` | 自定义目录（user/managed；project 需 workspace trust） |
| `claudeMdExcludes` | 跳过加载的 CLAUDE.md glob |
| `claudeMd` | （Managed）组织级指令注入 |

### Skills

| 键 | 说明 |
|----|------|
| `skillListingBudgetFraction` | Skills 描述 listing 占 context 比例，默认 ~2% |
| `maxSkillDescriptionChars` | 单 Skill description+when_to_use 上限，默认 1536 |
| `skillOverrides` | 如 `"my-skill": "name-only"` 省预算 |
| `disableBundledSkills` | 禁用内置 bundled skills/workflows |

### Subagent 与会话

| 键 | 说明 |
|----|------|
| `agent` | 主会话作为指定 subagent 运行（应用其 prompt/tools/model） |
| `cleanupPeriodDays` | 会话文件保留天数，默认 30；也影响孤儿 subagent worktree 清理 |
| `awaySummaryEnabled` | 离开几分钟后返回显示一行 recap |
| `autoCompactEnabled` | 接近 context 上限自动 compact，默认 `true` |
| `autoScrollEnabled` | 全屏模式跟随输出滚动 |

### 插件与市场

| 键 | 说明 |
|----|------|
| `enabledPlugins` | 已启用插件 |
| `extraKnownMarketplaces` | 额外市场源 |
| `strictKnownMarketplaces` | （Managed）严格市场白名单 |
| `blockedMarketplaces` | （Managed）市场黑名单 |

### MCP（Managed）

| 键 | 说明 |
|----|------|
| `allowedMcpServers` / `deniedMcpServers` | 白名单 / 黑名单 |
| `allowManagedMcpServersOnly` | 仅 Managed 列表生效 |
| `disableClaudeAiConnectors` | 禁用 claude.ai 连接器自动拉取 |
| `enableAllProjectMcpServers` | 自动批准项目 `.mcp.json` 中的 server |

### Hooks 与企业

| 键 | 说明 |
|----|------|
| `allowManagedHooksOnly` | 仅 Managed + 强制插件 hooks |
| `allowedHttpHookUrls` | HTTP hook URL 白名单 |
| `disableAllHooks` | 禁用全部 hooks |
| `companyAnnouncements` | 启动随机展示公告 |

### 功能开关（常见）

| 键 | 说明 |
|----|------|
| `disableAgentView` | 禁用 `claude agents`、`--bg`、`/background` |
| `disableRemoteControl` | 禁用 Remote Control |
| `disableWorkflows` | 禁用 Dynamic Workflows |
| `disableArtifact` | 禁用 Artifact 工具 |
| `channelsEnabled` | （Managed）Channels |

### 归属与 Git

| 键 | 说明 |
|----|------|
| `attribution.commit` / `pr` / `sessionUrl` | 自定义 commit/PR 归属 |

### UI

| 键 | 说明 |
|----|------|
| `editorMode` | 输入框 `normal` / `vim` |
| `defaultShell` | `!` 命令默认 shell：`bash` / `powershell` |
| `axScreenReader` | 无障碍扁平输出（v2.1.181+） |

### 认证

| 键 | 说明 |
|----|------|
| `apiKeyHelper` | 动态 API Key 脚本 |
| `awsAuthRefresh` / `awsCredentialExport` | Bedrock 凭证刷新 |

### 环境变量注入

`env` 块向 Claude Code 进程注入变量（也影响 subagent 默认环境）：

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "DISABLE_AUTOUPDATER": "1",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "1"
  }
}
```

常用变量见附录 B。

## `/config` 命令

```
/config                    # 交互式设置
/config thinking=false
/config autoUpdatesChannel stable
/config effort=xhigh
```

v2.1.181+ 的 `key=value` 语法在交互、`-p`、Remote Control 中均可用；部分项会写回 user settings。

## 配置调试

| 手段 | 用途 |
|------|------|
| `/doctor` | 设置合法性、MCP、安装状态 |
| `claude --safe-mode` | 禁用 CLAUDE.md/skills/plugins/hooks/MCP 等排查 |
| `InstructionsLoaded` hook | 记录哪些 instruction 文件被加载 |
| 自动备份 | 最近 5 份带时间戳 settings 备份 |

## Managed 无效值处理

Managed 中部分键若格式错误会被**强制安全默认**（如 `allowedMcpServers` 无效 → 空 allowlist 拒绝全部 MCP）。详见官方 Settings 文档 Invalid managed values 表。

> **资料来源**：[Settings](https://code.claude.com/docs/en/settings)、[Permissions](https://code.claude.com/docs/en/permissions)、[Debug configuration](https://code.claude.com/docs/en/debug-config)
