---
order: 16
date: 2026-06-26
---

# Plugins 与 Marketplaces

## 什么是 Plugin

Plugin 是 Claude Code 扩展的**打包分发单位**——一个 Plugin 目录可以包含 Skills、Hooks、MCP Server 配置、Subagents、Slash Commands 和 LSP 定义，打包成一个可安装的单元。

Plugin 的价值在于**分发**——团队中某个工程师写好的 Hook、Skill、MCP 配置，可以通过 Plugin 一键安装到其他成员的 Claude Code 中，版本化管理，更新时自动获取。

## Plugin 结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        # 清单文件（必须）
├── skills/                 # Skills 目录
│   └── my-skill/
│       └── SKILL.md
├── hooks/
│   └── hooks.json         # Hook 配置
├── agents/                 # Subagents 定义
├── commands/               # Slash Commands
├── .mcp.json              # MCP Server 配置
├── .lsp.json              # LSP Server 配置
├── monitors/
│   └── monitors.json      # 后台监控器
└── settings.json          # 默认设置
```

**关键规则**：只有 `plugin.json` 放在 `.claude-plugin/` 目录内，其余所有内容放在 Plugin 根目录。

### 最小 plugin.json

```json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

`name` 会成为 Plugin 中所有 Skill 的命名空间前缀：`plugin-name:skill-name`。`version` 可选，但设置后用户只在你更新版本号时收到更新。

## 市场（Marketplaces）

### 官方市场

Claude Code 自带 `claude-plugins-official` 官方市场。运行 `/plugin` 在 **Discover** 标签页浏览可用插件。

安装官方市场插件：

```
/plugin install <name>@claude-plugins-official
```

例如：

```
/plugin install github@claude-plugins-official
```

### 社区市场

社区维护了大量市场，2500+ 注册市场涵盖各种场景。通过 GitHub 仓库添加：

```
/plugin marketplace add anthropics/claude-plugins-community
```

然后从中安装：

```
/plugin install <name>@claude-community
```

### 企业团队市场

企业可以创建内部市场供团队成员共享 Plugin：

```json
{
  "extraKnownMarketplaces": {
    "my-org-marketplace": {
      "source": "github",
      "repo": "my-org/claude-plugins"
    }
  }
}
```

管理员可通过 `blockedMarketplaces` 和 `strictKnownMarketplaces` 控制市场来源。

## Plugin 命令

| 命令 | 功能 |
|------|------|
| `/plugin` | 打开 Plugin 交互界面 |
| `/plugin install <name>` | 安装 Plugin |
| `/plugin update <name>` | 更新 Plugin |
| `/plugin marketplace add <repo>` | 添加市场 |
| `/plugin marketplace update <name>` | 刷新市场 |

## Plugin 安全

- Plugin 未经 Anthropic 审核（官方市场除外）
- 安装 Plugin 意味着信任其作者
- Plugin 使用 `plugin-name:skill-name` 命名空间避免冲突
- 组织可通过 Managed Settings 控制 Plugin 安装权限
- 被封禁的市场来源在下载前即被检查，不会触碰文件系统

> **资料来源**：[Claude Code 官方文档 - Plugins](https://code.claude.com/docs/en/plugins)、[Discover Plugins](https://code.claude.com/docs/en/discover-plugins)
