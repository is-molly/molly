---
order: 0
date: 2026-06-27
---

# CLI 与 MCP

Claude Code 接外部能力，常见三条路：**内置工具**、**终端命令（常配合 Skill）**、**MCP Server**。多数任务用前两条就够；只有连平台 API、OAuth 这类事才值得上 MCP。

## 主栈一览（先认产品，再装）

| 产品 | 是什么 | Agent 里典型用法 | 形态 | 深潜 |
|------|--------|------------------|------|------|
| **Context7** | [context7.com](https://context7.com) 索引的**库官方文档**（按版本），防 API 幻觉 | 「查 Next.js App Router 怎么写 middleware」→ Bash 跑 `ctx7 library` / `ctx7 docs` | **CLI + Skill**（`find-docs`） | [03](03-Context7-CLI.md) |
| **Playwright CLI** | Microsoft 给 **coding agent** 用的浏览器终端工具（不是主写 E2E 测试套件） | 「打开 demo 站、点按钮、截图验证 UI」→ `playwright-cli open` / `snapshot` / `click` | **CLI + Skill** | [04](04-Playwright-CLI.md) |
| **Gitee MCP** | Gitee 官方的 **Remote MCP**，调 Open API v5（Issue / PR / 读远程文件等） | 「在我有权限的仓库开一个 bug Issue」→ MCP tool `create_issue` | **MCP**（HTTP + PAT） | [05](05-Gitee与GitHub-MCP.md) |
| **Postgres MCP Pro** | 带 **EXPLAIN / 慢查询 / 索引推荐** 的 Postgres MCP（不是裸 `SELECT` 包装） | 「这条 SQL 为什么慢、缺什么索引」→ `explain_query` / `analyze_workload_indexes` | **MCP**（stdio + 连接串） | [06](06-Postgres-MCP.md) |

GitHub 托管协作与 Gitee 同类，见 [05 · GitHub](05-Gitee与GitHub-MCP.md#github-mcp)。**02 章**只给最短安装命令；**不要跳过**上表对应深潜章的「是什么 / 怎么用」。

---

## 章节分类

| 类型 | 章节 | 说明 |
|------|------|------|
| **上手** | [01 概念与选型](01-概念与选型.md) · [02 安装与快速上手](02-安装与快速上手.md) | 三层、选型表；02 为**安装速查**（产品说明见 README 主栈 + 03–06） |
| **CLI 深潜** | [03 Context7](03-Context7-CLI.md) · [04 Playwright](04-Playwright-CLI.md) | 默认 CLI；章末各有一节 MCP **备选** |
| **MCP 深潜** | [05 Gitee/GitHub](05-Gitee与GitHub-MCP.md) · [06 Postgres](06-Postgres-MCP.md) | Agent 主路径就是 MCP 的产品 |
| **索引与收尾** | [07 生态索引](07-生态索引.md) · [08 自建 MCP](08-自建MCP.md) · [09 治理与排错](09-治理与排错.md) | 查地图、自建、排错 |

> **03 / 04 vs 05–06**：Context7、Playwright 上游推荐 **CLI + Skill**；Gitee、Postgres（调优）等写 **MCP 深潜**。

不单独写 Context7 MCP、Playwright MCP 章——默认 CLI；备选说明在 03、04 章末。

---

## CC Switch 用户：谁管什么

| 要做的事 | 用谁 |
|----------|------|
| 装 / 同步 MCP | GUI 表单；或 `cc-switch mcp add`（**外部编辑器**）→ `mcp sync`；或 `claude mcp add-json` |
| 装 ctx7、Playwright Skill | ctx7 → `~/.claude/skills/`；Playwright → **`cd ~ && playwright-cli install --skills`**（勿在项目根直接 install） |
| 会话里看 MCP | `/mcp` |
| OAuth、`add-json` | `claude mcp`（[02 章](02-安装与快速上手.md)） |

---

## 怎么读

**推荐（先懂再装）：** [README 主栈一览](#主栈一览先认产品再装) → [01 概念](01-概念与选型.md) → 按需读 [03](03-Context7-CLI.md)–[06](06-Postgres-MCP.md) 的 **「是什么」** → [02 安装](02-安装与快速上手.md)

**先装再用：** [02 安装](02-安装与快速上手.md) → 遇不懂再回 [03](03-Context7-CLI.md)–[06](06-Postgres-MCP.md)

**已在用 Skills 03 方案 A：** [Skills 03](../Skills/03-选型与组合方案.md#方案-a单人全栈) + [01 · 与 Skills 03 方案](01-概念与选型.md#和-skills-03-怎么加-cli--mcp)

---

## 不写什么

Filesystem MCP、Sequential Thinking MCP、废弃 npm `@modelcontextprotocol/server-github`、awesome 列表式堆砌；也不单独开「git / kubectl 命令大全」——走 Bash（[01 章 · 内置与 CLI](01-概念与选型.md)）。

Claude Code：[第 15 章 MCP](/AI/Agent/ClaudeCode/第15章%20MCP.html)
