---
order: 16
date: 2026-06-27
---

# Plugins 与 Marketplaces

Plugin 是 Skills、Hooks、MCP、Subagents、Commands、LSP、Monitors 的**打包分发单位**——适合团队共享、版本化、市场分发。

## Plugin vs 独立 `.claude/` 配置

| 方式 | Skill 命令 | 适用 |
|------|-----------|------|
| 独立 `.claude/skills/` | `/deploy` | 个人实验、单项目、短命令名 |
| **Plugin** | `/my-plugin:deploy` | 团队共享、多项目复用、市场发布、版本更新 |

迭代流程：先在 `.claude/` 试跑 → 成熟后打包为 Plugin。

## Plugin 结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        # 清单（必须在此目录）
├── skills/
│   └── my-skill/SKILL.md
├── agents/                 # Subagents
├── hooks/hooks.json
├── commands/               # 旧式 slash commands
├── .mcp.json
├── .lsp.json
├── monitors/monitors.json  # 后台监控器
└── settings.json           # 插件默认 settings
```

**规则**：仅 `plugin.json` 在 `.claude-plugin/` 内；其余内容在 Plugin **根目录**。

### plugin.json 要点

```json
{
  "name": "my-first-plugin",
  "description": "Team coding standards and review hooks",
  "version": "1.0.0",
  "author": { "name": "Your Team" },
  "repository": "https://github.com/org/my-first-plugin"
}
```

| 字段 | 说明 |
|------|------|
| `name` | 命名空间前缀 → `/name:skill-name` |
| `version` |  bump 后用户才收到更新；git 分发无 version 时用 commit SHA |
| `homepage` / `license` | 可选，展示于市场 |

## 本地开发与测试

```shell
claude --plugin-dir ./my-first-plugin
```

修改 Plugin 文件后，Skills 支持**热加载**（无需重启会话）。

## 市场（Marketplaces）

### 官方市场

```
/plugin
# Discover 标签浏览

/plugin install github@claude-plugins-official
/plugin install security-guidance@claude-plugins-official
```

### 添加社区市场

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install <name>@claude-community
```

社区 **2500+** 注册市场；Managed 可用 `blockedMarketplaces` 在下载前拦截，`strictKnownMarketplaces` 限制白名单。

### 企业内市场

```json
{
  "extraKnownMarketplaces": {
    "my-org": {
      "source": "github",
      "repo": "my-org/claude-plugins"
    }
  }
}
```

Managed `enabledPlugins` 可**强制启用**指定插件（用户无法禁用）。

### 其他安装源

官方支持从 **GitHub 仓库、.zip、URL** 安装市场/插件（v2.1.19+ 等版本逐步增强）。详见 [Discover plugins](https://code.claude.com/docs/en/discover-plugins)。

## Plugin 命令

| 命令 | 功能 |
|------|------|
| `/plugin` | 管理界面：Discover / Installed / Marketplaces |
| `/plugin install <name>@<market>` | 安装 |
| `/plugin update <name>` | 更新 |
| `/plugin enable` / `disable` | 启用/禁用 |
| `/plugin marketplace add <repo\|url>` | 添加市场 |
| `/plugin marketplace update <name>` | 刷新市场索引 |

CLI：`claude plugin` 子命令（见 CLI reference）。

## 与 Skills / Subagents 的命名

- Plugin Skill：`plugin-name:skill-name`（不会与 project/personal skill 冲突）
- Plugin Subagent：typeahead 显示 `my-plugin:code-reviewer`
- 同名冲突时：**Enterprise > Personal > Project > Plugin**（Plugin 有独立 namespace）

## 安全与治理

| 风险 | 缓解 |
|------|------|
| 未审核社区插件 | 仅安装信任来源；官方市场由 Anthropic 维护 |
| 恶意 hooks/MCP | Managed：`allowManagedHooksOnly`、`allowedMcpServers` |
| 市场投毒 | `blockedMarketplaces` 下载前拦截 |
| Shell in Skill | Managed：`disableSkillShellExecution` |

安装 Plugin = 信任作者能在你的 Claude Code 环境执行 hooks、读配置、注册 MCP。

## 发布自己的 Plugin

1. 完善 `plugin.json` 与 README
2. 推送到 GitHub 仓库
3. 用户 ` /plugin marketplace add your-org/repo`
4. 更新时 bump `version`

可选提交到 [claude-plugins-community](https://github.com/anthropics/claude-plugins-community) 或自建市场索引 JSON。

## 与 Routines / 云端的区别

- **本地 CLI Plugin**：存在本机，Routines 默认看不到 CLI `claude mcp add` 的 server
- 要在 Routines 用 MCP：在 claude.ai **Connectors** 添加，或仓库提交 `.mcp.json`

> **资料来源**：[Plugins](https://code.claude.com/docs/en/plugins)、[Discover plugins](https://code.claude.com/docs/en/discover-plugins)、[Plugins reference](https://code.claude.com/docs/en/plugins-reference)
