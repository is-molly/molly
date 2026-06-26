---
order: 31
date: 2026-06-26
---

# 附录C：衍生工具简介

本章介绍对 Claude Code 开发者有帮助的衍生工具。重点展开 **cc-switch CLI**，它是日常开发中最实用的配置管理工具。

---

## cc-switch CLI

**GitHub**：[SaladDay/cc-switch-cli](https://github.com/SaladDay/cc-switch-cli) ｜ **当前版本**：v5.4.0 ｜ **编写语言**：Rust

### 定位

cc-switch CLI 用 Rust 编写，是一个纯终端的配置管理工具。它的核心价值是**管理多个 AI 编码工具的 API 提供方（Provider）配置**——Claude Code、Codex、Gemini CLI、OpenCode、OpenClaw 各有不同的配置格式（JSON / TOML / .env），每换一个 API 提供方就要手动改文件。cc-switch 提供一套统一的命令行界面来管理这一切。

与 Claude Code 的关系：cc-switch 不是 Claude Code 的替代品，也不修改 Claude Code 本身。它只是帮你管理 `~/.claude/settings.json`、`~/.claude.json`、`~/.codex/config.toml` 这些配置文件，让你在多个 API 提供方之间快速切换。

零后台进程——切换配置并启动 Claude 后立即退出，不驻留守护进程。

### 安装

快速安装（推荐）：

```shell
curl -fsSL https://github.com/SaladDay/cc-switch-cli/releases/latest/download/install.sh | bash
```

默认安装到 `~/.local/bin`，可通过 `CC_SWITCH_INSTALL_DIR` 自定义。

其他方式：

```shell
# Homebrew
brew install Linuxdazhao/cc-switch/cc-switch

# Cargo
cargo install cc-switch

# Windows（Scoop）
scoop bucket add cc-switch https://github.com/Linuxdazhao/scoop-cc-switch
scoop install cc-switch
```

### 启动

```shell
cc-switch                 # 进入交互式 TUI
cc-switch provider list   # 直接执行命令
```

交互模式支持原生 Vim 键位：`j`/`k` 导航、`n`/`p` 翻页。全局 `--app` 标志指定目标应用：`claude`（默认）、`codex`、`gemini`、`opencode`、`openclaw`。

---

## 提供方管理

提供方（Provider）是一组 API 配置的集合，包含 API 地址、密钥、模型映射。cc-switch 支持为不同 AI 工具管理独立的提供方列表。

### 基本命令

```
# 列出、查看当前
cc-switch provider list               # Claude 的提供方
cc-switch --app codex provider list   # Codex 的提供方
cc-switch provider current            # 当前激活的

# 切换
cc-switch provider switch <id>
cc-switch use <name>                  # 快捷方式：切换并启动 Claude

# 增删改
cc-switch provider add                # 交互式创建
cc-switch provider edit <id>          # 修改
cc-switch provider duplicate <id>     # 复制
cc-switch provider delete <id>        # 删除

# 诊断
cc-switch provider speedtest <id>     # API 延迟测试
cc-switch provider stream-check <id>  # 流式响应健康检查
cc-switch provider fetch-models <id>  # 从 API 获取远程模型列表

# 导出
cc-switch provider export <id>        # 导出到 ./.claude/settings.local.json
```

### 创建提供方（完整交互流程）

运行 `cc-switch provider add` 后，进入 12 步交互式向导：

**Step 1：Select provider type**

选择 `Custom`（自定义）。cc-switch 内置了部分服务商预设，但建议始终从 Custom 开始理解每个字段的含义。如果上游服务商提供了预设，可以直接选用。

**Step 2：Provider Name**

随意命名，方便在列表中识别。如 `DeepSeek V4`、`智谱 GLM`。

**Step 3：Website URL（可选）**

可留空，或填入服务商的网址（如 `https://api.deepseek.com`）。仅用于备忘，不影响实际请求。

**Step 4：Generated ID**

自动根据名称生成（如输入 `DeepSeek V4` 生成 `deepseek-v4`），用于命令行引用。可以手动修改，但通常保持默认即可。

**Step 5：Auth Field**

选择认证字段类型：

| 选项 | 说明 |
|------|------|
| `ANTHROPIC_AUTH_TOKEN` | OAuth 登录时生成的短期 Token。官方 Claude 订阅用户选这个 |
| `ANTHROPIC_API_KEY` | 传统的 API Key。**使用第三方服务商（DeepSeek、智谱等）时选这个** |

第三方 API 的 Key 格式就是 `ANTHROPIC_API_KEY`，不是 `ANTHROPIC_AUTH_TOKEN`。选错会导致认证失败。

**Step 6：API Key**

粘贴服务商提供的密钥。如 DeepSeek 的 `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`。

**Step 7：Base URL**

API 端点地址。对于提供 Anthropic 兼容端点的服务商（如 DeepSeek、智谱），直接填兼容端点的完整 URL；对于只提供 OpenAI 格式的服务商，填 OpenAI 格式的端点，后续 Step 11 通过 `API Format` 配置格式转换。

**Step 8：Configure model names?**

选 `Yes` 会进入模型映射子流程，为 5 个角色分别指定模型：

| 角色 | 说明 |
|------|------|
| Default Model | 默认使用的模型 |
| Reasoning Model (Thinking) | 需要深度推理时使用的模型（对应 Extended Thinking） |
| Haiku Model | 对应 Haiku 的快速轻量模型 |
| Sonnet Model | 对应 Sonnet 的日常编码模型 |
| Opus Model | 对应 Opus 的复杂任务模型 |

每个槽位填实际的上游模型名（如 `deepseek-v4-pro`、`deepseek-v4-flash`、`glm-5.2`、`glm-5.1`）。不需要填 Claude 的模型名，cc-switch 会自动处理映射。

如果上游服务商只有一个模型，可以在 `Default Model` 填一个，其余留空。

**Step 9：Hide AI Attribution**

是否隐藏 AI 署名。选 `No` 时 Claude Code 的 commit 消息会附带 AI 生成标记。选 `Yes` 则隐藏。一般保持 `No`。

**Step 10：Configure optional fields?**

添加备注或自定义排序索引。通常选 `No` 跳过。

**Step 11：API Format**

选择上游 API 的协议格式：

| 选项 | 说明 | 是否需要代理 |
|------|------|-------------|
| `Anthropic Messages (Native)` | Anthropic 原生格式 | 否，直连 |
| `OpenAI Chat Completions (Requires proxy)` | OpenAI 格式 | **是**，需要 cc-switch 代理做格式转换 |

DeepSeek 和智谱都有 Anthropic 兼容端点 → 选 `Anthropic Messages (Native)`。

选了需要代理的格式后，创建完提供方还需要执行 `cc-switch proxy enable` 启动代理（见[代理管理](#代理管理)）。

**Step 12：Attach Common Config**

是否附加已保存的通用配置片段（`common config`）。这个配置片段是你在 `cc-switch config common set` 中设置的跨提供方公共配置（如禁用的遥测、commit 归属信息等）。

- 选 `Y` 会将通用配置合并到当前 Provider 中
- 选 `n` 则跳过

如果你之前没有设置过 `common config`，选 `n` 即可。

**Step 13：Confirm**

展示配置摘要，确认后创建。确认前仔细检查 `API Format` 和 `Base URL`——这两个字段创建后修改比较麻烦（需要 `cc-switch provider edit <id>`）。

### 切换与验证

```shell
cc-switch provider list                 # 查看所有提供方，找到 ID
cc-switch provider switch <id>          # 切换
cc-switch provider stream-check <id>    # 测试流式连接
claude                                  # 启动 Claude Code
```

如果切换后 Claude Code 仍然走官方 API，检查：
1. 是否还有系统级 `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN` 环境变量覆盖了 cc-switch 的配置（用 `cc-switch env check` 诊断）
2. 是否之前用 `claude login` 登录了 OAuth（执行 `claude /logout` 退出）

---

## 接入 DeepSeek

DeepSeek 提供了 Anthropic 兼容端点，可以直接接收 Anthropic Messages 格式的请求，**无需格式转换**。

### 获取 API Key

1. 访问 [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)，注册并登录
2. 点击「创建 API Key」，命名后复制保存（只显示一次）
3. 确保账户有余额

### 创建提供方

```shell
cc-switch provider add
```

**Step 1** → `Custom`
**Step 2** Provider Name → `DeepSeek V4`
**Step 3** Website URL → 留空（或填 `https://api.deepseek.com`，仅备忘）
**Step 4** Generated ID → 保持默认（自动生成 `deepseek-v4`）
**Step 5** Auth Field → **`ANTHROPIC_API_KEY`**（第三方 Key 必须选这个，不是 `ANTHROPIC_AUTH_TOKEN`）
**Step 6** API Key → 粘贴 DeepSeek 的 `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
**Step 7** Base URL → `https://api.deepseek.com/anthropic`
**Step 8** Configure model names? → **Yes**

模型映射：

| 角色 | 填写内容 | 说明 |
|------|----------|------|
| Default Model | `deepseek-v4-pro` | 默认使用 V4-Pro |
| Reasoning Model (Thinking) | `deepseek-v4-pro` | 深度推理也用 V4-Pro |
| Haiku Model | `deepseek-v4-flash` | 快速轻量任务用 Flash |
| Sonnet Model | `deepseek-v4-pro` | 日常编码用 Pro |
| Opus Model | `deepseek-v4-pro` | 复杂任务用 Pro |

**Step 9** Hide AI Attribution → `No`
**Step 10** Configure optional fields? → `No`
**Step 11** API Format → **`Anthropic Messages (Native)`**（DeepSeek 端点兼容 Anthropic 格式）
**Step 12** Attach Common Config → `n`（没设置过的话跳过）
**Step 13** → 确认创建

### 切换与验证

```shell
cc-switch provider list
cc-switch provider switch <id>         # 切换
cc-switch provider stream-check <id>   # 测试连接
claude                                  # 启动
```

### 手动配置（不用 cc-switch 的备选方案）

编辑 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_API_KEY": "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "ANTHROPIC_MODEL": "claude-sonnet-4-20250514"
  }
}
```

然后 `claude --dangerously-skip-permissions` 启动（使用第三方 API 时 OAuth 认证不可用）。

---

## 接入智谱 GLM

智谱同时提供 OpenAI 兼容端点和 Anthropic 兼容端点。普通按量计费 token 用户可以直接用 Anthropic 兼容端点 `https://open.bigmodel.cn/api/anthropic` 直连，**无需代理**。

> **注意**：智谱还有一个 OpenAI 兼容的 Coding 专属端点 `https://open.bigmodel.cn/api/coding/paas/v4`，以及额度查询端点 `https://open.bigmodel.cn/api/monitor/usage/quota/limit`，这两个端点**仅限 Coding Plan 订阅用户**。普通 token 用户调用会报 `当前用户不存在coding plan`。但模型调用用的 `/api/anthropic` 端点对普通 token 开放，不受影响。

### 获取 API Key

1. 访问 [open.bigmodel.cn](https://open.bigmodel.cn/)，注册并登录
2. 进入控制台 [API Keys 管理页面](https://bigmodel.cn/usercenter/proj-mgmt/apikeys) 创建 API Key
3. 复制保存（格式为 `xxxxxxxx.xxxxxx`）

### 创建提供方

```shell
cc-switch provider add
```

**Step 1** → `Custom`
**Step 2** Provider Name → `智谱 GLM`
**Step 3** Website URL → 留空（或填 `https://open.bigmodel.cn`，仅备忘）
**Step 4** Generated ID → 保持默认（自动生成 `智谱-glm`）
**Step 5** Auth Field → **`ANTHROPIC_API_KEY`**
**Step 6** API Key → 粘贴智谱的 API Key
**Step 7** Base URL → `https://open.bigmodel.cn/api/anthropic`
**Step 8** Configure model names? → **Yes**

模型映射：

| 角色 | 填写内容 | 说明 |
|------|----------|------|
| Default Model | `glm-5.2` | 最新旗舰，744B MoE，1M 上下文 |
| Reasoning Model (Thinking) | `glm-5.2` | 深度推理 |
| Haiku Model | `glm-5.1` | 快速轻量任务 |
| Sonnet Model | `glm-5.2` | 日常编码 |
| Opus Model | `glm-5.2` | 复杂任务 |

**Step 9** Hide AI Attribution → `No`
**Step 10** Configure optional fields? → `No`
**Step 11** API Format → **`Anthropic Messages (Native)`**（智谱有 Anthropic 兼容端点，直连无需代理）
**Step 12** Attach Common Config → `n`
**Step 13** → 确认创建

### 切换与验证

```shell
cc-switch provider list
cc-switch provider switch <id>
cc-switch provider stream-check <id>
claude
```

### 关于 Quota 查询报错

切换到智谱后，cc-switch 的 Provider 详情面板会显示：

```
Quota: query failed  API error: 当前用户不存在coding plan
```

这是**正常现象**，不影响模型调用。原因：cc-switch 的额度查询走智谱的 `/api/monitor/usage/quota/limit` 端点，该端点仅对 Coding Plan 订阅用户开放。普通按量计费 token 用户无法查询套餐额度，但模型调用走的是 `/api/anthropic` 端点，完全正常。

如果需要查看用量，登录 [智谱开放平台控制台](https://open.bigmodel.cn/) 查看账单余额即可。

---

## MCP Server 管理

统一管理 Claude Code、Codex、Gemini CLI、OpenCode 的 MCP Server 配置。支持 stdio / HTTP / SSE 三种传输协议。

```
cc-switch mcp list                           # 列出
cc-switch --app codex mcp list               # Codex 的 MCP
cc-switch mcp add                            # 交互式添加
cc-switch mcp edit <id>                      # 编辑
cc-switch mcp delete <id>                    # 删除
cc-switch mcp enable <id> --app claude       # 为指定应用启用
cc-switch mcp disable <id> --app claude      # 为指定应用禁用
cc-switch mcp sync                           # 同步到 live 配置文件
cc-switch mcp import --app claude            # 从 live 配置导入
cc-switch mcp validate <command>             # 验证命令是否在 PATH 中
```

---

## Skills 管理

跨应用的 Skills 统一管理，与全书第 13 章的 Skills 体系互补——在 cc-switch 中统一安装，Claude Code 自动加载。

```
cc-switch skills discover <query>            # 搜索
cc-switch skills install <name>              # 安装
cc-switch skills list                        # 已安装列表
cc-switch skills enable <name> --app claude  # 为指定应用启用
cc-switch skills disable <name> --app claude # 禁用
cc-switch skills sync                        # 同步到应用目录
cc-switch skills sync-method symlink         # 设同步方式（auto|symlink|copy）
cc-switch skills repos add anthropics/skills # 添加仓库
cc-switch skills repos list                  # 仓库列表
cc-switch skills scan-unmanaged              # 扫描未管理 Skills
cc-switch skills import-from-apps            # 导入
cc-switch skills info <name>                 # 详情
cc-switch skills uninstall <name>            # 卸载
```

---

## Prompts 管理

管理跨应用的 System Prompt 预设，写入 CLAUDE.md / AGENTS.md / GEMINI.md。

```
cc-switch prompts create         # 创建
cc-switch prompts list           # 列出
cc-switch prompts current        # 当前激活的
cc-switch prompts activate <id>  # 激活
cc-switch prompts deactivate     # 停用
cc-switch prompts edit <id>      # 编辑
cc-switch prompts show <id>      # 查看完整内容
cc-switch prompts delete <id>    # 删除
```

---

## 配置管理

### 备份与恢复

```
cc-switch config backup
cc-switch config backup --name before-migration
cc-switch config restore                           # 交互式选择
cc-switch config restore --backup <id>             # 按 ID 恢复
cc-switch config export ~/cc-switch-backup.json
cc-switch config import ~/cc-switch-backup.json
cc-switch config reset                              # 恢复默认
```

### WebDAV 云同步

```
cc-switch config webdav show
cc-switch config webdav set --base-url <url> --username <user> --password <password> --enable
cc-switch config webdav jianguoyun --username <user> --password <password>
cc-switch config webdav check-connection
cc-switch config webdav upload
cc-switch config webdav download
```

### 通用配置片段

跨所有提供方共享的公共配置：

```shell
cc-switch --app claude config common show
cc-switch --app claude config common set --snippet \
  '{"env":{"CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC":1}}'
cc-switch --app claude config common clear
```

---

## 代理管理

cc-switch 内置本地 HTTP 代理，用于上游 API 格式不兼容时做格式转换。

```
cc-switch proxy show        # 查看状态和路由
cc-switch proxy enable      # 启用
cc-switch proxy disable     # 禁用
cc-switch proxy serve       # 前台运行（调试，可看请求日志）
```

默认监听 `http://127.0.0.1:3456`，启用后自动重定向 Claude Code 请求。

---

## 环境诊断

```
cc-switch env check         # 检查环境变量冲突
cc-switch --app claude env check
cc-switch env list          # 列出相关环境变量
cc-switch env tools         # 检查 CLI 工具是否已安装
```

---

## 其他辅助命令

```
cc-switch completions bash  # Shell 补全（zsh/fish/powershell）
cc-switch update            # 更新
cc-switch config path       # 查看配置路径
```

---

## 数据结构

cc-switch 的存储目录（可通过 `CC_SWITCH_CONFIG_DIR` 自定义）：

```
~/.cc-switch/
├── cc-switch.db          # SQLite 主数据库
├── settings.json         # 设备级设置
├── skills/               # Skills 源文件（SSOT）
├── backups/              # 自动备份（保留最近 10 个）
└── config.json           # 旧版兼容文件
```

cc-switch 修改的各工具 live 配置文件：

| 目标工具 | 被修改的文件 |
|----------|-------------|
| Claude Code | `~/.claude/settings.json`、`~/.claude.json`、`~/.claude/CLAUDE.md` |
| Codex | `~/.codex/auth.json`、`~/.codex/config.toml`、`~/.codex/AGENTS.md` |
| Gemini CLI | `~/.gemini/.env`、`~/.gemini/settings.json`、`~/.gemini/GEMINI.md` |
| OpenCode | `~/.config/opencode/opencode.json`、`~/.config/opencode/AGENTS.md` |
| OpenClaw | `~/.openclaw/openclaw.json`、`~/.openclaw/AGENTS.md` |

如果目标 CLI 未初始化（对应配置目录不存在），cc-switch 会跳过写入并提示 warning。先运行一次目标 CLI（如 `claude --help`）创建目录，再切换提供方。

---

## 常见问题

**切换提供方后配置没生效？**

检查是否有环境变量冲突（系统级 `ANTHROPIC_API_KEY` 等会覆盖 cc-switch 的配置）：

```shell
cc-switch --app claude env check
```

如果发现冲突，删除 shell 配置文件（`.zshrc` / `.bashrc`）中对应的 `export` 行。

**用第三方 API 时报认证错误？**

确保在终端执行了 `claude /logout` 退出 Claude Code 的 OAuth 登录，否则会优先使用 OAuth token 而非 API Key。

**智谱 GLM 切换后 Quota 查询报错 `当前用户不存在coding plan`？**

这是正常现象。cc-switch 的额度查询走智谱的 `/api/monitor/usage/quota/limit` 端点，该端点仅对 Coding Plan 订阅用户开放。普通按量计费 token 用户会报这个错，但模型调用走的是 `/api/anthropic` 端点，不受影响。详见[接入智谱 GLM](#接入智谱-glm) 章节。

---

> **资料来源**：[cc-switch CLI GitHub](https://github.com/SaladDay/cc-switch-cli)、[cc-switch CLI README（中文）](https://github.com/SaladDay/cc-switch-cli/blob/main/README_ZH.md)
