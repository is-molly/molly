---
order: 3
date: 2026-06-27
---

> 上游：`upstash/context7-mcp` 的 `packages/cli`（npm **`ctx7`**）。下文路径、Skill 名、命令参数均来自 2026-06 本地 clone 的源码与 README。

# Context7 CLI

---

## 是什么

<a id="是什么"></a>

**Context7**（[context7.com](https://context7.com)）维护大量开源库的**版本化官方文档**索引。模型写 React、Next.js、Prisma 等代码时，若凭训练数据容易**幻觉 API**；Context7 提供「先查文档、再写代码」的数据源。

本书用的 **`ctx7`** 是上游 npm CLI（仓库 `upstash/context7-mcp` · `packages/cli`），不是 Context7 网站本身。装好后 Agent 侧形态是：

| 组件 | 作用 |
|------|------|
| **Skill `find-docs`** | 告诉模型：遇到库 API 问题该触发 Context7 |
| **规则 `context7.md`** | 规定两步：先 `ctx7 library` 解析库 ID，再 `ctx7 docs` 拉片段 |
| **Bash 命令** | 模型在终端跑 `npx ctx7@latest library …` / `docs …`，输出进对话 |

**会话里长什么样：** 你问「Next.js 15 middleware 怎么鉴权」→ Agent（在 Skill 触发下）跑 `ctx7 library nextjs "middleware auth"` → 得到 `/vercel/next.js` → 再 `ctx7 docs /vercel/next.js "middleware authentication"` → 用返回的文档片段写代码。**不**占 MCP tool 槽。

**和 MCP 版 Context7 的区别：** 同一文档源；CLI 省 context（无整屏 tool schema），上游 **coding agent 推荐 CLI + Skill**。MCP 备选见 [章末](#mcp-fallback)。

Claude Code 推荐安装：`npx ctx7 setup --cli --claude`（下文 [安装](#安装与-setup-cli-实际做了什么)）。

---

## CLI 与 MCP 是两套安装路径

`setup.ts` 里 `resolveMode()` 决定走哪条路：

| 命令 | 写入内容 |
|------|----------|
| `npx ctx7 setup --cli --claude` | Skill **`find-docs`** + 规则 `context7.md`；**无** MCP |
| `npx ctx7 setup --mcp --claude` | `~/.claude.json` 里 `mcpServers.context7` + Skill **`context7-mcp`** + MCP 规则 |

`remove.ts` 里按模式删不同 Skill：

| `ctx7 remove` | 删除的 Skill |
|---------------|--------------|
| `--cli` | `find-docs` |
| `--mcp` | `context7-mcp` |

**不要两种都装。** 交互式 `npx ctx7 setup` 会问 *How should your agent access Context7?*，选 **CLI + Skills**。

---

<a id="安装与-setup-cli-实际做了什么"></a>

## 安装与 `setup --cli` 实际做了什么

### 一条命令

```bash
npx ctx7 setup --cli --claude
```

首次 `npx` 会提示安装 npm 包 **`ctx7`**（如 `ctx7@0.5.3`），确认后继续。

### OAuth 与参数

<a id="ctx7-oauth"></a>

**首次安装：OAuth 设备码（`resolveCliAuth`）**

带 `--cli` **不会**问「MCP 还是 CLI」；但若本机**没有**有效登录且**未**传 `--api-key`，`setupCli()` 第一步会调用 **`resolveCliAuth`**，弹出 **Sign in to Context7** 设备码框：

| 终端提示 | 含义 |
|----------|------|
| `Your one-time code: XXXX-XXXX` | 在浏览器输入的一次性码 |
| `https://context7.com/oauth/device?user_code=…` | 授权链接（可 Enter 自动打开浏览器） |
| `Waiting for authorization...` | 在网页点 Approve 前会一直转；**正常，不是卡死** |

授权成功后才会：`downloadSkill` → 写 `~/.claude/skills/find-docs/` → 写 `~/.claude/rules/context7.md`。

**不想走 OAuth：**

```bash
npx ctx7 setup --cli --claude --api-key <key>   # dashboard 创建 API Key
# 或事先：export CONTEXT7_API_KEY=…
# 或：ctx7 login                                  # 单独刷新 OAuth
```

常用参数（`packages/cli/README.md`、`setup.ts`）：

| 参数 | 作用 |
|------|------|
| `--cli` | 走 `setupCli()`，不写 MCP |
| `--mcp` | 写 MCP server + **`context7-mcp`** Skill + MCP 规则 |
| `--claude` | 只配 Claude Code |
| `--cursor` | Cursor（`.cursor/skills/`） |
| `--codex` | Codex |
| `--opencode` | OpenCode |
| `--gemini` | Gemini CLI |
| `--antigravity` | Antigravity（`.agent/skills/`） |
| `--project` / `-p` | 项目级：写到当前目录下的 `.claude/` 等，而非 `~/.claude` |
| `--yes` / `-y` | 跳过**部分**交互（如 agent 选择）；**不能**代替 OAuth，未登录仍会出设备码 |
| `--api-key <key>` | 用 API Key，**跳过 OAuth 设备码** |
| `--oauth` | **MCP 模式**用 OAuth endpoint（`…/mcp/oauth`）；CLI 模式默认也是 context7.com OAuth，不必单独加此 flag |
| `--stdio` | MCP 模式用 **本地 stdio**（默认 Remote HTTP） |

交互式 `ctx7 setup`（**不加** `--cli` / `--mcp`）才会问 *How should your agent access Context7?*，选 **CLI + Skills**。

### 装完落盘

#### 执行步骤（源码 `setupCli`，授权成功之后）

1. **`resolveCliAuth`** — 见上节 OAuth；无 `--api-key` 且未登录时**先**设备码，**后**装 Skill  
2. **`downloadSkill("/upstash/context7", "find-docs")`** — 从 Context7 拉 Skill 包  
3. **`installSkillFiles("find-docs", files, skillDir)`** — 写到 Skill 目录  
4. **`installRule(agentName, "cli", scope)`** — 写规则文件（`FALLBACK_CLI` / `context7-cli.md`）

#### 支持的 Agent 目录（上游 README）

| 客户端 | Skill 目录 |
|--------|------------|
| Claude Code | `.claude/skills/` |
| Cursor | `.cursor/skills/` |
| Antigravity | `.agent/skills/` |
| Universal（Copilot、OpenCode 等） | `.agents/skills/` |

setup 会检测已安装的 agent；多客户端可一次写：`npx ctx7 setup --cli --claude --cursor`。

#### Claude Code 写入路径（`agents.ts` · scope=global）

| 类型 | 路径 |
|------|------|
| Skill 目录 | `~/.claude/skills/find-docs/`（含 `SKILL.md`） |
| 规则 | `~/.claude/rules/context7.md` |
| MCP 配置 | **不写入**（仅 `--mcp` 时写 `~/.claude.json`） |

`--project` 时 Skill 为 `./.claude/skills/find-docs/`，规则为 `./.claude/rules/context7.md`。

#### 成功输出（`ctx7 0.5.x`）

```text
✔ Authenticated

✔ Downloaded find-docs skill
✔ Context7 CLI setup complete

  Claude Code
    + Skill installed
      ~/.claude/skills/find-docs
    + Rule installed
      ~/.claude/rules/context7.md
```

随后 Claude Code 里 `/skills` 应能看到 **find-docs**；`/mcp` **不应**多出 `context7` server。

失败常见 **EACCES**，源码会提示 `chown` `~/.claude`。

### 与 CC Switch

ctx7 **直接写** `~/.claude/skills/` 与 `~/.claude/rules/`，**不经过** CC Switch 的 SSOT 数据库。

| 你的习惯 | 做法 |
|----------|------|
| 不用 CC Switch 管 Skill | 装完即用；`/skills` 验证 |
| CC Switch 统一管 **全局** Skill | `scan-unmanaged -a claude` → `import-from-apps ~/.claude/skills/find-docs -a claude` → `sync`（**须带目录**） |
| 只多装 Cursor 等 | `npx ctx7 setup --cli --claude --cursor`，不必 sync |

**不要** 以为 `skills sync` 会把 ctx7 刚写的 Skill **导入** CC Switch——`sync` 方向是 **SSOT → 各 Agent live 目录**。

---

## 规则里要求模型怎么做（`templates.ts` · FALLBACK_CLI）

装完后，`context7.md` 大致要求 agent：

1. **解析库 ID**  
   `npx ctx7@latest library <name> "<用户的完整问题>"`  
   除非用户已给出 `/org/project` 格式 ID。

2. **选库** — 按 name 精确匹配、description、Code Snippets 数量、Source Reputation（High/Medium）、Benchmark Score 选一条；不对就换带标点的全名（如 `next.js` 不是 `nextjs`）。

3. **拉文档**  
   `npx ctx7@latest docs <libraryId> "<用户的完整问题>"`  
   query 要具体，不要单词糊弄。

4. **用文档回答**，不要瞎编 API。

**硬限制（与 MCP 一致）：**

- 同一问题 **`library` / `docs` 最多 3 次**  
- query 里不要放 API Key、密码  
- **不要用于**：纯重构、从零写脚本、业务逻辑调试、code review（规则原文 *Do not use for*）

**限额报错时**：告知用户 `npx ctx7 login` 或设置环境变量 **`CONTEXT7_API_KEY`**（见 dashboard）。

Codex 用户规则里还会加一段：Context7 CLI 要在 **sandbox 外**跑（`CODEX_CLI_SANDBOX_GUIDANCE`）。

---

## 命令详解

### `ctx7 library` — 对应 MCP 的 `resolve-library-id`

```bash
ctx7 library <libraryName> [query]
ctx7 library react
ctx7 library nextjs "app router middleware"
ctx7 library react --json
```

- **`libraryName`**：官方库名，保留标点（`Next.js` 不是 `nextjs`）  
- **`query`**：可选；有 query 时 API 会按相关性排序（与 MCP tool 的 `query` + `libraryName` 同义，见 `index.ts`）

TTY 下输出示例字段（`docs.ts` · `formatLibraryResult`）：

- `Context7-compatible library ID`（如 `/vercel/next.js`）  
- Description、Code Snippets 数  
- Source Reputation：High / Medium / Low / Unknown（由 `trustScore` 换算）  
- Benchmark Score、Versions 列表  

`--json` 输出原始 `results` 数组，方便脚本解析。

### `ctx7 docs` — 对应 MCP 的 `query-docs`

```bash
ctx7 docs <libraryId> "<query>"
ctx7 docs /vercel/next.js "middleware authentication"
ctx7 docs /facebook/react "useEffect cleanup" --json
```

- **`libraryId`**：必须 `/org/project` 或 `/org/project/version`  
- **`query`**：完整问题，不要用单个词  

### 认证

```bash
ctx7 login      # 浏览器 OAuth
ctx7 whoami
ctx7 logout
```

API Key：[context7.com/dashboard](https://context7.com/dashboard)。环境变量 **`CONTEXT7_API_KEY`** 可被 CLI 使用。

关闭遥测：`CTX7_TELEMETRY_DISABLED=1`（README）。

### 卸载 setup 产物

```bash
npx ctx7 remove --claude --cli      # 删 find-docs + context7 规则，不动 MCP
npx ctx7 remove --claude --all        # CLI + MCP 都删
npx ctx7 remove --claude --mcp        # 只删 MCP 那套（context7-mcp Skill）
```

全局装过 `npm install -g ctx7` 时另执行 `npm uninstall -g ctx7`；只用 `npx ctx7` 则不必。

---

## Prompt 怎么写（根 README）

触发文档检索：

```text
How do I set up Next.js 14 middleware? use context7
```

已知库 ID，跳过 `library`：

```text
Implement basic auth with Supabase. use library /supabase/supabase
```

句末 **`use context7`** 或规则自动触发均可。

---

## 和 MCP 两 tool 的对照

| | CLI（默认） | MCP（备选，见下节） |
|--|-----|---------------------|
| 查库 | `ctx7 library <name> [query]` | `resolve-library-id`：`libraryName` + `query`（必填） |
| 查文档 | `ctx7 docs <id> "<query>"` | `query-docs`：`libraryId` + `query` |
| 参数别名容错 | — | server 会把 `userQuery`、`libraryID` 等误名改写成 canonical |
| Remote | — | `https://mcp.context7.com/mcp`；OAuth 时为 `…/mcp/oauth` |
| stdio MCP | — | `npx -y @upstash/context7-mcp --api-key …` |

MCP 的选型逻辑（snippet 数、reputation、benchmark）与 CLI 规则 **FALLBACK_CLI** 一致，只是通道不同。

---

<a id="mcp-fallback"></a>

## 备选：MCP 模式（一般不必装）

**默认不要装。** 只有下面情况才考虑从 CLI 切到 MCP：

- 你的 Agent **不支持** Bash，或 Skill 触发不稳定，只能靠 MCP tool
- 团队规范要求 **Remote MCP + OAuth**，统一在 dashboard 管 policy
- 已用 `ctx7 setup --cli` 仍不满意，且愿意 **删掉 find-docs** 再试 MCP

### 安装（与 CLI 互斥）

```bash
# 先清掉 CLI 那套（若装过）
npx ctx7 remove --claude --cli

# Remote MCP + Skill context7-mcp + 规则
npx ctx7 setup --mcp --claude

# 或 stdio（本地进程，需 API Key）
npx ctx7 setup --mcp --claude --stdio --api-key YOUR_KEY
```

`setup --mcp` 写入（`agents.ts`）：

| 项 | 内容 |
|----|------|
| MCP | `~/.claude.json` → `mcpServers.context7`，Remote URL 见上表 |
| Skill | **`context7-mcp`**（不是 find-docs） |
| 规则 | 同一 `context7.md`，内容为 **FALLBACK_MCP** |

手动等价（Claude Code Remote + API Key）：

```bash
claude mcp add-json context7 '{"type":"http","url":"https://mcp.context7.com/mcp","headers":{"CONTEXT7_API_KEY":"YOUR_KEY"}}'
```

OAuth 时 URL 为 `https://mcp.context7.com/mcp/oauth`，由 `ctx7 login` / setup 交互处理。

### 两个 MCP tool（`packages/mcp/src/index.ts`）

**`resolve-library-id`** — 参数 `libraryName`、`query`（均必填）；同一问题最多 3 次。

**`query-docs`** — 参数 `libraryId`（`/org/project` 或带 version）、`query`；最多 3 次。除非用户已给出 library ID，否则先 resolve。

规则与 CLI 相同：不要用于纯重构、从零写脚本、业务逻辑调试、code review。

### 切回 CLI

```bash
npx ctx7 remove --claude --mcp
npx ctx7 setup --cli --claude
```

---

## 示例：手动跑通再交给 Claude

```bash
npx ctx7 library nextjs "App Router middleware JWT cookies"
# 记下输出里的 ID，例如 /vercel/next.js

npx ctx7 docs /vercel/next.js "middleware read JWT from cookies redirect login"
# 把输出贴进对话，或让 Claude 自己跑上述两条 Bash
```

会话里你可以说：

```text
用 ctx7 查 Next.js App Router 里 middleware 怎么读 cookie 里的 JWT。use context7
```

期望行为：Bash 调 `ctx7 library` → `ctx7 docs` → 按返回文档写代码，而不是 hallucinate `middleware.ts` API。

---

## 常见问题

| 现象 | 处理 |
|------|------|
| 仍瞎编 API | `/skills` 看 **find-docs** 是否 Enabled；检查 `~/.claude/rules/context7.md` |
| 同时有 find-docs 和 context7-mcp | `ctx7 remove --claude --all` 后只重装 `--cli` |
| `library` 无结果 | 换官方库名标点；加第二参数 query；`ctx7 login` |
| 想改用 MCP | 见上文「备选：MCP 模式」；先 `remove --cli` 再 `setup --mcp` |
