---
order: 31
date: 2026-06-26
---

# 附录C：衍生工具简介

本章介绍 Claude Code 日常开发里常用的**本地衍生工具**。和官方 CLI 的关系：它们不替代 `claude`，而是读写 `~/.claude/` 下的配置与会话数据。

| 工具 | 干什么 | 典型场景 |
|------|--------|----------|
| **cc-switch** | 管 Provider、MCP、Skills、Prompts | 换 API、同步 MCP、装 Skill |
| **cc-session-viewer** | 浏览、对比、复制会话 JSONL | 复盘某次对话、对比两次 prompt、导出 transcript |
| **cc-read** | 统计会话里 **Read** 工具读了哪些文件 | 看 Agent 反复读哪些文件、哪个项目读文件最多 |
| **claude-webcache** | **WebFetch / WebSearch** 结果持久化到本地 SQLite | 跨会话复用文档页、少重复抓官网/API 文档 |

下面按工具分节写安装与用法。cc-switch 篇幅最长，因为 Provider / MCP 字段多；后三者命令少或走 Plugin 安装，但数据路径与 Hook 行为需要说清。

---

## cc-switch CLI

**项目**：[farion1231/cc-switch](https://github.com/farion1231/cc-switch)（桌面 GUI + CLI/TUI 一体，如 v5.8.x）｜ CLI fork：[SaladDay/cc-switch-cli](https://github.com/SaladDay/cc-switch-cli) ｜ **编写语言**：Rust

### 三种界面（写文档时最容易混）

| 入口 | 实际体验 | 典型命令 |
|------|----------|----------|
| **桌面 GUI** | 点按钮、表单、Preset 下拉 | 打开 CC Switch 应用 → MCP / Provider / Skills 面板 |
| **TUI** | 全屏终端，`j`/`k` 导航，`a`/`e` 增删改 | `cc-switch`（无子命令） |
| **CLI 子命令** | 单条命令；**MCP / Provider 添加常打开 `$EDITOR`（vim）** | `cc-switch mcp add`、`cc-switch provider add` |

**易错点：** 把 **GUI 表单步骤** 写成 **`cc-switch mcp add` 的行为**——CLI 的 `mcp add` 是 **外部编辑器 + JSON**，不是「选 HTTP → 填 URL」问答。Provider 在较新版本里也可能是 **TUI 问答 + 外部编辑器收尾**，与 MCP 不完全相同；以本机 `cc-switch --version` 为准。

### 定位

cc-switch CLI 用 Rust 编写，是 **TUI + CLI 双模式** 的配置管理工具（桌面版另有 GUI）。核心价值是**管理多个 AI 编码工具的 API 提供方（Provider）配置**——Claude Code、Codex、Gemini CLI、OpenCode、OpenClaw 各有不同的配置格式（JSON / TOML / .env），每换一个 API 提供方就要手动改文件。cc-switch 提供统一界面管理这一切。

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

### 创建提供方：TUI 问答 vs CLI 外部编辑器

**`cc-switch provider add`（CLI）** 在 5.8+ 常见流程：终端问答若干项 → 展示 **Provider Configuration Summary** → 可选 **Opening external editor...** 改 JSON → 确认写入。不是 MCP 那种「一上来就只有 vim」，但**同样不是**桌面 GUI 的鼠标表单。

**`cc-switch`（TUI，无子命令）** 里添加 Provider 则是全屏逐步问答，字段含义与下表一致。

下列 **Step 1–13** 是 **字段说明 + TUI 参考**（DeepSeek / 智谱示例也按此填）；若你本机 CLI 直接进了 vim，在编辑器里填 **等价的 id、name、baseUrl、apiKey、models** 即可。

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

每个槽位填实际上游模型名（如 `deepseek-v4-pro[1m]`、`glm-5.2[1m]`）。**支持 1M 的 Pro/Sonnet/Opus 槽位要加 `[1m]` 后缀**，否则 Claude Code 仍按 200K 预算 compact（详见 [1M 与 cc-switch](#1m-上下文与-1m-后缀cc-switch-必看)）。Haiku / Flash 槽位通常不加。不需要填 Claude 官方模型名。

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

<a id="1m-上下文与-1m-后缀cc-switch-必看"></a>

## 1M 上下文与 `[1m]` 后缀（cc-switch 必看）

DeepSeek V4 Pro、智谱 GLM-5.x 等上游**实际支持 1M token**，但 Claude Code 连第三方 `ANTHROPIC_BASE_URL` 时默认只给 **200K** 的 `/context` 和 compact 预算。要在客户端打开 1M，必须在 cc-switch 写的模型名里加 **`[1m]`**（CLI 本地解析，**发 API 前会去掉**，上游仍收到 `deepseek-v4-pro` 这类裸名）。

| 现象 | 原因 |
|------|------|
| `/context` 显示 200K | Provider 模型映射未加 `[1m]` |
| ~187K 就 AutoCompact | 同上，按 200K 窗口在压 |
| 加了 `[1m]` 报模型不存在 | Claude Code 版本过旧，升级后再试 |

### cc-switch 里怎么填

**DeepSeek（Pro / Sonnet / Opus / Default 槽位）**

| 角色 | 填写内容 |
|------|----------|
| Default / Reasoning / Sonnet / Opus | `deepseek-v4-pro[1m]` |
| Haiku | `deepseek-v4-flash`（无 `[1m]`） |

**智谱 GLM（你用的 5.1 或文档里的 5.2，按订阅型号选）**

| 角色 | 填写内容 |
|------|----------|
| Default / Reasoning / Sonnet / Opus | `glm-5.1[1m]` 或 `glm-5.2[1m]` |
| Haiku | `glm-5.1`（无 `[1m]`） |

智谱还建议在 env 里加（可用 **common config** 或 Provider 内 env）：

```shell
cc-switch --app claude config common set --snippet \
  '{"env":{"CLAUDE_CODE_AUTO_COMPACT_WINDOW":"1000000"}}'
```

或在 `cc-switch provider add` **Step 12 Attach Common Config** 选 `Y`（若 common 里已有上项）。

已有 Provider 漏配时：

```shell
cc-switch provider edit <id>    # 改模型映射，Pro/Sonnet/Opus 加 [1m]
cc-switch provider switch <id>
claude
/context                        # 应约 1M，不是 200K
```

概念说明见 [第 4 章 · 第三方 Provider 与 [1m]](第4章%20模型、Effort与AdaptiveThinking.md#第三方-provider-与-1m-后缀)。

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

模型映射（**Pro 系加 `[1m]`，与 [DeepSeek 官方 Claude Code 文档](https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code) 一致）：

| 角色 | 填写内容 | 说明 |
|------|----------|------|
| Default Model | `deepseek-v4-pro[1m]` | 1M 上下文预算 |
| Reasoning Model (Thinking) | `deepseek-v4-pro[1m]` | 同上 |
| Haiku Model | `deepseek-v4-flash` | 轻量，不加 `[1m]` |
| Sonnet Model | `deepseek-v4-pro[1m]` | 日常编码 |
| Opus Model | `deepseek-v4-pro[1m]` | 复杂任务 |

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
/context                                # 确认约 1M，不是 200K
```

### 手动配置（不用 cc-switch 的备选方案）

编辑 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_API_KEY": "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash"
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

模型映射（**Sonnet/Opus 加 `[1m]`**；下表以 **glm-5.1** 为例，若用 5.2 把型号改成 `glm-5.2[1m]`）：

| 角色 | 填写内容 | 说明 |
|------|----------|------|
| Default Model | `glm-5.1[1m]` | 1M 上下文（智谱 [文档](https://docs.bigmodel.cn/cn/guide/develop/claude)） |
| Reasoning Model (Thinking) | `glm-5.1[1m]` | 深度推理 |
| Haiku Model | `glm-5.1` | 轻量，不加 `[1m]` |
| Sonnet Model | `glm-5.1[1m]` | 日常编码 |
| Opus Model | `glm-5.1[1m]` | 复杂任务 |

**Step 9** Hide AI Attribution → `No`
**Step 10** Configure optional fields? → `No`
**Step 11** API Format → **`Anthropic Messages (Native)`**（智谱有 Anthropic 兼容端点，直连无需代理）
**Step 12** Attach Common Config → 若 common 里**已有** `CLAUDE_CODE_AUTO_COMPACT_WINDOW` 选 **`Y`**，否则选 **`N`**（下一步补写）
**Step 13** → 确认创建

### 智谱 1M 必做（common config）

模型映射里的 `[1m]` 只解决 Claude Code 本地 `/context` 预算；智谱官方还要求客户端把 compact 窗口设到 1M。在 common config 或该 Provider 的 `env` 中确保有：

```json
"CLAUDE_CODE_AUTO_COMPACT_WINDOW": "1000000"
```

一次性写入 common config：

```shell
cc-switch --app claude config common set --snippet \
  '{"env":{"CLAUDE_CODE_AUTO_COMPACT_WINDOW":"1000000","API_TIMEOUT_MS":"3000000"}}'
```

### 切换与验证

```shell
cc-switch provider list
cc-switch provider switch <id>
cc-switch provider stream-check <id>
claude
/context    # 应约 1M
```

### 关于 Quota 查询报错

切换到智谱后，**桌面 GUI** 的 Provider 详情可能显示：

```
Quota: query failed  API error: 当前用户不存在coding plan
```

这是**正常现象**，不影响模型调用。原因：cc-switch 的额度查询走智谱的 `/api/monitor/usage/quota/limit` 端点，该端点仅对 Coding Plan 订阅用户开放。普通按量计费 token 用户无法查询套餐额度，但模型调用走的是 `/api/anthropic` 端点，完全正常。

如果需要查看用量，登录 [智谱开放平台控制台](https://open.bigmodel.cn/) 查看账单余额即可。

---

## MCP Server 管理

统一管理 Claude Code、Codex、Gemini CLI、OpenCode 的 MCP Server 配置。支持 stdio / HTTP / SSE 三种传输协议。

**三种入口（不要混用描述）：**

| 方式 | 行为 |
|------|------|
| **桌面 GUI** | 点 **MCP** → **+** → Preset 或 Custom 表单（Server ID、Transport、URL/Command 等）→ 保存 |
| **`cc-switch mcp add`** | 打开 **`$EDITOR` 外部编辑器**（未设置时常为 vim）编辑 JSON 模板；保存退出后写入 CC Switch 库 |
| **`cc-switch` TUI** | MCP 区 `a` 添加 / `e` 编辑（`j`/`k` 导航） |
| **桌面 GUI** | MCP 面板 **+** → Preset / Custom 表单 |

```
cc-switch mcp list                           # 列出
cc-switch --app codex mcp list               # Codex 的 MCP
cc-switch mcp add                            # 外部编辑器添加（见上表）
cc-switch mcp edit <id>                      # 外部编辑器编辑
cc-switch mcp delete <id>                    # 删除
cc-switch mcp enable <id> --app claude       # 为指定应用启用
cc-switch mcp disable <id> --app claude      # 为指定应用禁用
cc-switch mcp sync                           # 同步到 live 配置文件
cc-switch mcp import --app claude            # 从 live 配置导入
cc-switch mcp validate <command>             # 验证命令是否在 PATH 中
```

不想用 vim：`EDITOR=nano cc-switch mcp add`，或 `EDITOR="code --wait" cc-switch mcp add`。

---

## Skills 管理

跨应用的 Skills 统一管理，与全书第 13 章的 Skills 体系互补——在 cc-switch 中统一安装，Claude Code 自动加载。

**SPEC 格式**：`install` 接受 **`owner/repo:目录名`**（或仓库内唯一的目录短名），**没有** npx 的 `--skill`。首次使用某仓库须 `repos add`。

```
cc-switch skills repos add anthropics/skills   # 首次添加源仓库
cc-switch skills discover frontend-design      # 查目录名
cc-switch skills install anthropics/skills:frontend-design -a claude
cc-switch skills discover <query>              # 搜索（别名 search）
cc-switch skills install <spec>                # 安装（spec = 目录名 或 owner/repo:目录名）
cc-switch skills list                          # 已安装列表
cc-switch skills enable <name> --app claude    # 为指定应用启用
cc-switch skills disable <name> --app claude   # 禁用
cc-switch skills sync                          # 同步到应用目录
cc-switch skills sync-method symlink           # 设同步方式（auto|symlink|copy）
cc-switch skills repos list                    # 仓库列表
cc-switch skills scan-unmanaged                # 扫描未管理 Skills
cc-switch skills import-from-apps              # 导入
cc-switch skills info <name>                   # 详情
cc-switch skills uninstall <name>              # 卸载
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

**DeepSeek / 智谱已接好，但 `/context` 仍是 200K？**

Provider 模型映射里 Pro/Sonnet/Opus **缺 `[1m]` 后缀**。按 [1M 与 cc-switch](#1m-上下文与-1m-后缀cc-switch-必看) 改 `cc-switch provider edit <id>`，智谱另加 `CLAUDE_CODE_AUTO_COMPACT_WINDOW=1000000`，再 `provider switch` 后新开会话验证。

---

## cc-session-viewer

**项目**：[ravikanchikare/cc-session-viewer](https://github.com/ravikanchikare/cc-session-viewer) ｜ **npm**：`cc-session-viewer` ｜ **编写语言**：TypeScript（Express 后端 + React/Vite 前端）

### 定位

Claude Code 把每次会话记成 `~/.claude/projects/<项目目录名>/<sessionId>.jsonl`，一行一条 JSON。终端里只能看到当前输出；想查「上次修 bug 时 Claude 读了哪些文件、调了几次 Bash」，得打开 jsonl 或用工具。

**cc-session-viewer** 在本机起一个 Web 界面，把 jsonl 解析成可点的列表：按**用户每条 prompt** 分成 checkpoint，展开后能看到 tool 调用分组、每条消息的 token、模型、分支等。还支持**两个会话并排对比**、复制 transcript、导出 Markdown。

和 cc-switch 的分工：cc-switch 管**下次**怎么连 API；cc-session-viewer 管**过去**会话里发生了什么。

### 安装与启动

```shell
# 不安装，直接跑（推荐先试）
npx cc-session-viewer

# 全局安装
npm install -g cc-session-viewer
cc-session-viewer
```

启动后默认监听 **`http://localhost:4000`**（上游 `bin/cli.ts` 默认端口；旧 README 写 3333 的以本机行为为准）。会自动打开浏览器；若该端口已有实例在跑，会提示 *already running* 并只打开页面，不会重复起服务。

常用参数：

```shell
cc-session-viewer --no-open              # 不自动开浏览器
cc-session-viewer --port 4000            # 指定端口
cc-session-viewer --roots ~/.claude      # 指定会话根（可多个，逗号分隔）
cc-session-viewer --roots "/path/a/.claude,/path/b/.claude"
cc-session-viewer --restart              # 杀掉占用端口的进程后重启
```

开发模式（改 UI 时用）：`npx tsx bin/cli.ts --dev --no-open` 起 API，另开终端 `npx vite`，前端在 `http://localhost:5173`，Vite 把 `/api` 代理到后端。

### 数据从哪来

| 路径 | 内容 |
|------|------|
| `~/.claude/projects/<projectId>/*.jsonl` | 主会话 |
| `~/.claude/projects/<projectId>/<子目录>/subagents/*.jsonl` | Subagent 会话（界面可读；分支时会过滤 sidechain） |

`<projectId>` 一般是工作目录路径编码后的目录名（例如 `-Users-you-code-myapp`）。侧边栏按 **Root → 项目 → 会话** 三级列出来。

**多 Root：** 除默认 `~/.claude` 外，可在界面里添加别的 `.claude` 目录（例如第二台机器同步过来的副本）。配置写在 **`~/.claude-session-browser.json`**，字段是 `roots[]`（`path` + `label`）。CLI 的 `--roots` 会在启动时把这些路径注册进去。

### 界面里能做什么

**Checkpoint 列表**

每条用户消息对应一行 checkpoint，显示：prompt 摘要、该轮 tool 次数、输入/输出 token、时间。点开一行展开完整消息流；连续的 tool_use 会收成一组（终端里一句「改了 3 个文件」背后可能是十几次 Read/Grep 再 Edit，这里能看清顺序）。

**会话元数据**

顶栏或侧栏会话节点上常见：模型名、git 分支、Claude Code 版本、消息总数、累计 token、最后活跃时间。

**对比模式（Compare）**

选两个会话，左右两栏独立滚动。用来对比：同一任务用 opus / haiku、两种 prompt、两次不同改法。每栏有自己的 metadata + checkpoint 列表。

**复制 Transcript**

可把整段会话、单轮、或对比选中部分复制成纯文本。复制规则（上游 AGENTS.md）：只保留 **User / Claude 正文**，不带 Bash/Read 等 tool 调用细节；子 agent 会话在「整段复制」时也会带上。适合贴进别的 LLM 或文档。

**项目固定（Pin）**

常开的项目可 pin 到侧边栏顶部。

### 会改磁盘的操作（使用前确认）

浏览、对比、复制**只读** jsonl。下面操作**会写** `~/.claude/projects/` 下的文件，和 Claude Code 内置 `/rewind`、`/fork` 同类，无法从 viewer 里撤销：

| 操作 | 行为 |
|------|------|
| **Rename** | 在 jsonl **末尾追加**一条 `custom-title` 记录（与 Claude Code 命名规则一致） |
| **Rewind** | 截断 jsonl：保留到某条 user 消息（含该条），**后面全部删除** |
| **Branch** | 从某 checkpoint 复制出新 `.jsonl`，新 `sessionId`，带 `forkedFrom` 元数据 |
| **Delete** | 删除整个 `<sessionId>.jsonl` |
| **Move** | 把会话文件挪到同一 root 下另一个 project 目录 |

Rewind / Branch 前应在 UI 里看清 checkpoint；重要会话先备份 `~/.claude/projects/` 对应目录。

### 导出 Markdown（CLI 脚本）

仓库自带导出脚本，不经过 Web UI，适合归档或给别的工具索引：

```shell
# 在 clone 后的仓库里，或 npm 包内通过 npm run
npm run list-projects
npm run list-projects -- -r ~/.claude

npm run export-md -- --out ./export --project <projectId>
npm run export-md -- --out ./all-md --all -r ~/.claude
npm run export-md -- --dry-run -o ./tmp -p <projectId>
```

输出布局：`projects/{projectId}/{sessionId}/transcript.md`，子 agent 在 `subagents/agent-*.md`。

统计导出文件 token 数（依赖 tiktoken，首次会建 `scripts/.venv`）：

```shell
npm run count-tokens
npm run count-tokens -- --format md
```

### HTTP API（自建脚本时用）

后端路由前缀 `/api/roots`：

| 方法 | 路径 | 作用 |
|------|------|------|
| GET | `/api/roots` | 已注册的 root 列表 |
| POST | `/api/roots` | `{ "path", "label?" }` 添加 root |
| DELETE | `/api/roots/:rootId` | 移除 root |
| GET | `.../projects/:projectId/sessions` | 会话列表 |
| GET | `.../sessions/:sessionId/messages` | 解析后的消息（`?filter=user,assistant` 可过滤） |
| POST | `.../sessions/:sessionId/rename` | `{ "title" }` |
| POST | `.../sessions/:sessionId/rewind` | `{ "targetMessageUuid" }` |
| POST | `.../sessions/:sessionId/branch` | `{ "targetMessageUuid" }` |
| POST | `.../sessions/:sessionId/move` | `{ "targetProjectId" }` |
| DELETE | `.../sessions/:sessionId` | 删会话文件 |

### 常见问题

**端口被占用 / 页面打不开**

用 `cc-session-viewer --restart`，或换 `--port`。

**侧边栏没有项目**

确认本机跑过 Claude Code，且存在 `~/.claude/projects/`。若会话在别的目录，用 `--roots` 或界面添加 root。

**和 Claude Code 内置 `/resume` 的关系**

Viewer 不负责恢复 TUI 会话；它用来**看**和**整理** jsonl。要在 Claude Code 里接着聊，仍用 `claude --resume` 或会话列表；Rewind 之后只有被截断后的 jsonl 会被 resume 读到。

---

## cc-read

**项目**：[yurukusa/cc-read](https://github.com/yurukusa/cc-read) ｜ **npm**：`cc-read` ｜ **编写语言**：单文件 Node（`cli.mjs`，**零 npm 依赖**）

### 定位

Claude Code 会大量调用内置 **Read** 工具读仓库文件。读了多少次、集中在哪些后缀、哪个项目最「费眼」，jsonl 里都有，但没人会手 grep。

**cc-read** 扫描 `~/.claude/projects/**/*.jsonl`，抽出所有 `tool_use` 且 `name === "Read"` 的记录，按**文件名**、**扩展名**、**项目目录名**汇总，在终端打柱状图。属于 [cc-toolkit](https://yurukusa.github.io/cc-toolkit/) 里 60+ 会话分析小工具之一；和 cc-session-viewer 互补——viewer 看**单次对话结构**，cc-read 看**跨会话阅读习惯**。

### 安装与运行

```shell
npx cc-read
npx cc-read --json
```

没有子命令，没有配置文件。Node **≥ 18**。只认 **`~/.claude/projects/`**（源码写死 `join(homedir(), '.claude', 'projects')`），暂不支持 `--roots`；要分析别的机器拷过来的数据，需放到该路径下或改 clone 后的 `cli.mjs`。

扫描范围：

- 各项目目录下 `*.jsonl`
- 各项目下 `*/subagents/*.jsonl`（子 agent 的 Read 也计入）

### 终端输出读什么

```
  cc-read — Most-Read Files
  ════════════════════════════════════════

  ▸ Overview
    Total Read calls:   …        # Read 工具调用总次数
    Unique files read:  …        # 按 basename 去重后的文件数
    Most-read file:     foo.py   # 读次数最多的文件名
    Top extension:      .py      # 占比最高的后缀
    Hall of Fame (1k+): N files  # 单文件 Read ≥ 1000 次

  ▸ Most-read files (top 15)     # 柱状条 + ★ 表示 Hall of Fame
  ▸ By extension                 # 按后缀占比
  ▸ By project (top 10)          # projectId 目录名，不是 git 仓库友好名
  ▸ Hall of Fame                 # 列出所有 ≥1000 次的文件
```

**统计口径要注意：**

1. **只统计 Read**，不含 Grep、Glob、Bash `cat` 等。
2. 文件键是 **`basename`**（`cli.mjs` 里 `basename(fp)`），不同目录下同名 `index.ts` 会**合并**成一条。
3. **项目**键来自路径里 `projects/` 下一级目录名，和 cc-session-viewer 侧边栏一致，不是 `package.json` 里的项目名。

### JSON 输出（脚本 / CI）

```shell
npx cc-read --json
```

结构（v1.0.0）：

```json
{
  "version": "1.0.0",
  "totalReads": 36973,
  "uniqueFiles": 4512,
  "topFile": { "name": "dungeon_game.py", "count": 6723 },
  "topExtension": { "ext": ".py", "count": 12683 },
  "byFile": [{ "name": "…", "count": 6723 }],
  "byExtension": [{ "ext": ".py", "count": 12683 }],
  "byProject": [{ "project", "count": 1234 }]
}
```

`byFile` 最多 20 条；扩展名、项目列表为全量排序结果。适合 `jq` 接报表或和团队共享「哪些文件被 Agent 读烂了该拆模块」。

### 浏览器版（免安装）

[https://yurukusa.github.io/cc-read/](https://yurukusa.github.io/cc-read/) 把 `~/.claude/projects/`（或任意子文件夹）拖进页面即可分析，逻辑与 CLI 相同，数据不上传（纯前端读本地文件夹）。

### 和 cc-session-viewer 怎么配合

| 问题 | 用谁 |
|------|------|
| 某次对话里 Claude 具体读了啥、调了啥 tool | cc-session-viewer 展开 checkpoint |
| 过去一个月哪个 `.py` 被 Read 最多 | cc-read |
| 对比两次会话的 prompt 效果 | cc-session-viewer Compare |
| 导出会话给人看 | cc-session-viewer 复制 / export-md |
| 写脚本拉 Read 统计 | cc-read `--json` |

### 常见问题

**`Could not read ~/.claude/projects/`**

目录不存在或权限不足。先在本机跑过一次 Claude Code。

**数字特别大**

长会话 + 反复改同一文件时，单次任务可能 Read 同一文件几十次，属正常；Hall of Fame（1000+）说明该文件长期是 Agent 的「主战场」，可以考虑拆文件或加 CLAUDE.md 指引减少盲读。

**和 cc-switch 无关**

cc-read 不读 Provider、不读 MCP 配置，只读会话 jsonl。

---

## claude-webcache

**项目**：[theYahia/claude-webcache](https://github.com/theYahia/claude-webcache) ｜ **npm**：`@theyahia/claude-webcache` ｜ **安装方式**：Claude Code **Plugin**（推荐）或 npm 全局 + 手动写 `settings.json`

### 定位

Claude Code 内置 **WebFetch** / **WebSearch** 有约 **15 分钟、单会话** 的内存缓存。新开一个 `claude` 会话，同一文档 URL 往往会再抓一遍。

**claude-webcache** 用 Hook + 本地 **SQLite**（`~/.webcache/cache.db`）把抓取结果**跨会话持久化**。v0.5+ 在 **PreToolUse** 里先查缓存，命中则**跳过真实 WebFetch**；**PostToolUse** 把新结果写入库。默认 TTL **不限**（可用环境变量设过期）。

和 cc-switch 的分工：cc-switch 管**模型走哪家 API**；webcache 管**网页抓取结果存哪**。接 DeepSeek、智谱 GLM 等第三方 Provider **完全兼容**——它不经过 `ANTHROPIC_BASE_URL`，也不改 cc-switch 写的 Provider 配置。

### 能省什么、不能省什么

| 能省 | 不能省 |
|------|--------|
| 同 URL + 同 prompt **跨会话**不再重复 HTTP 抓取 | **Read / Grep / Bash / MCP** 占用的模型输入 token |
| 命中缓存时跳过 WebFetch 调用（省网络、省一轮 tool 往返） | 缓存正文**仍作为 tool 结果进上下文**（大页面照样占 token） |
| WebSearch 结果单独命名空间，默认 **6 小时** TTL | 几乎不写代码、很少 WebFetch 的任务 **收益接近零** |

缓存键 = `namespace + 规范化 URL + prompt`。同一文档 URL，**换一句 prompt 可能命不中**。

### 安装（Plugin，推荐）

在 Claude Code 里执行（CLI / Desktop / VS Code 扩展命令相同）：

```shell
claude plugin marketplace add theYahia/claude-webcache
claude plugin install claude-webcache@theyahia
```

装好后**无需再配 Hook**——Plugin 自带 MCP Server 与 Pre/PostToolUse。若 `/plugin install` TUI 报第三方源不支持（[Claude Code #41653](https://github.com/anthropics/claude-code/issues/41653)），用上面 CLI 命令即可。

**不要用 cc-switch 管这个**——cc-switch 同步的是 `mcpServers` / Provider；Plugin 走 `/plugin` 生态，二者独立。

备选：npm 全局安装后手改 `~/.claude/settings.json` 注册 MCP 与 SessionStart Hook，见上游 [README](https://github.com/theYahia/claude-webcache#option-2--npm-global)（需 Node **22.5+**，用内置 `node:sqlite`）。

### 工作机制（简图）

```
WebFetch 请求
    → PreToolUse：查 ~/.webcache/cache.db
        命中 → 直接返回缓存，跳过网络
        未命中 → 正常 WebFetch
    → PostToolUse：把结果写入 SQLite
```

SessionStart 时 Hook 可能注入一行统计，例如 `webcache 142 pages cached, 87% hit rate`（缓存为空则不显示）。

### 常用环境变量

| 变量 | 默认 | 作用 |
|------|------|------|
| `WEBCACHE_NAMESPACE` | `""`（共享） | 按项目隔离缓存，如 `WEBCACHE_NAMESPACE=myapp claude` |
| `WEBCACHE_TTL_DAYS` | 不限 | 全局过期天数；`0` 或未设 = 不限 |
| `WEBCACHE_DOMAIN_TTL` | 无 | JSON 按域名 TTL，如 `{"news.com":1,"arxiv.org":0}` |
| `WEBCACHE_MAX_OUTPUT_MB` | 10 | 超过则拒绝写入 |
| `WEBCACHE_AUTOREAD` | 开 | 设 `0` 关闭 PreToolUse 自动读（仍会自动写） |
| `WEBCACHE_SEARCH_TTL_HOURS` | 6 | WebSearch 缓存 TTL |

### CLI 与 MCP

npm 包带 `claude-webcache` 命令：`stats`、`list`、`invalidate`、`clear`、`dashboard`（默认 `http://localhost:37778`）等。

MCP 工具包括 `cached_fetch`、`cached_search`、`cache_stats`、`cache_list`、`cache_invalidate`、`cache_clear` 等；一般靠 Hook 自动即可，不必手调。

### 安全

库文件在 `~/.webcache/cache.db`，**不要提交到 git**。写入前会对 URL 里明显的 `user:pass@`、常见 query token 做**展示级脱敏**；带鉴权的 URL 仍建议用 **Header** 传 token，少用 URL 内嵌密钥。`WEBCACHE_STRICT_REDACT=1` 会让缓存键也基于脱敏 URL——适合静态文档，**不适合**「同一 path、不同 token 返回不同用户数据」的接口。

### 和 cc-read / cc-session-viewer 怎么配合

| 问题 | 用谁 |
|------|------|
| 某次对话读了哪些仓库文件 | cc-session-viewer / cc-read |
| 反复抓同一 API 文档、Release Notes | claude-webcache |
| 换 DeepSeek / 智谱 Provider | cc-switch |
| 看 WebFetch 命中率、清某 URL 缓存 | `claude-webcache stats` / `invalidate` |

### 常见问题

**装了插件，/context 还是很大**

webcache 省的是**重复抓取**，不是把网页从上下文里删掉。第一次 Fetch 的大页面仍会进模型输入。

**和 cc-switch 的 MCP 列表冲突吗**

Plugin 注册的 MCP 与 cc-switch 同步的 MCP **并存**。会话里 tool schema 略增，通常远小于重复 WebFetch 的成本。

**多项目共用一台机器**

给不同仓库设不同 `WEBCACHE_NAMESPACE`，避免 A 项目的文档缓存被 B 项目误命中。

---

> **资料来源**：[cc-switch CLI GitHub](https://github.com/SaladDay/cc-switch-cli)、[cc-switch CLI README（中文）](https://github.com/SaladDay/cc-switch-cli/blob/main/README_ZH.md)、[cc-session-viewer](https://github.com/ravikanchikare/cc-session-viewer)（2026-06 clone 核对）、[cc-read](https://github.com/yurukusa/cc-read)（2026-06 clone 核对）、[claude-webcache README](https://github.com/theYahia/claude-webcache)（2026-06 核对）
