---
order: 8
date: 2026-06-27
---

# 自建 MCP

## 何时自建、何时用 CLI、何时用现成 MCP

判断顺序：**内置 / Bash 够不够** → **有官方 CLI + Skill 吗**（[03](03-Context7-CLI.md)、[04](04-Playwright-CLI.md)）→ **Registry / [07 章](07-生态索引.md) 有现成 MCP 吗**（05–06）→ 都没有且 tool 边界稳定，才 **自建**（本章）。

| 情况 | 建议 |
|------|------|
| 查公开库文档 | **ctx7 CLI**，不自建 |
| 公司内网工单 API，无 CLI | **自建** thin MCP（几个 tool 即可） |
| GitHub / Gitee Issue | **现成 Remote MCP**（[05 章](05-Gitee与GitHub-MCP.md)） |
| 把现有 REST 全量暴露 | **不要**——tool 过多会爆 context；拆成少量高频 tool 或写 Skill 约束调用 |

---

## 技术选型

| | TypeScript | Python |
|--|------------|--------|
| SDK | [`@modelcontextprotocol/sdk`](https://github.com/modelcontextprotocol/typescript-sdk) | [`mcp`](https://github.com/modelcontextprotocol/python-sdk)（PyPI） |
| 适合 | 与 Node 工具链、前端团队一致 | 数据 / ML / 已有 Python 服务 |
| Claude Code 连接 | stdio 最常见 | stdio（`uv run` / `uvx`） |
| 参考实现 | 各产品 MCP 多用 TS（如 Context7） | `servers` 里 [mcp-server-git](https://github.com/modelcontextprotocol/servers/tree/main/src/git) |

**传输：** 个人 / 团队内网 Agent 用 **stdio** 最简单（Claude Code 起子进程）。要对多客户端、远程托管再用 **Streamable HTTP**（见 [01 · MCP 通道](01-概念与选型.md#mcp-通道)）。Claude Code 官方配置示例见 [Agent 书第 15 章](/AI/Agent/ClaudeCode/第15章%20MCP.html)。

---

## 最小 TypeScript Server（stdio）

目录示例：

```
hello-mcp/
├── package.json
├── tsconfig.json
└── src/index.ts
```

`package.json`（ESM）：

```json
{
  "name": "hello-mcp",
  "type": "module",
  "bin": { "hello-mcp": "./dist/index.js" },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "typescript": "^5.8.0",
    "tsx": "^4.19.0"
  }
}
```

`src/index.ts`（一个 echo tool）：

```typescript
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "hello-mcp",
  version: "0.1.0",
});

server.registerTool(
  "echo",
  {
    title: "Echo",
    description: "Return input text unchanged",
    inputSchema: {
      text: z.string().describe("Text to echo back"),
    },
  },
  async ({ text }) => ({
    content: [{ type: "text", text }],
  })
);

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**stdio 铁律：** 协议 JSON 走 **stdout**；日志、调试信息只写 **stderr**。往 stdout 打 `console.log` 会破坏 MCP 会话。

本地调试：

```bash
npm install
npx tsx src/index.ts
# 另开终端用 Inspector（见下）
```

---

## Python 方向（结构要点）

上游 reference [mcp_server_git](https://github.com/modelcontextprotocol/servers/tree/main/src/git) 模式：

- `mcp.server.Server` 注册 `list_tools` / `call_tool`
- `mcp.server.stdio.stdio_server()` 作 transport
- 参数用 **Pydantic** `BaseModel`，和 TS 侧 zod 同理

安装与运行通常：

```bash
uv init my-mcp && cd my-mcp
uv add mcp
uv run python -m my_mcp
```

细节跟 [Python SDK 文档](https://github.com/modelcontextprotocol/python-sdk) 走；不必为 hello world 再包一层 FastMCP，除非团队已在用。

---

## MCP Inspector

官方 [Inspector](https://github.com/modelcontextprotocol/inspector) 在 **不启动 Claude Code** 时测 tool 列表与调用：

```bash
npx @modelcontextprotocol/inspector npx tsx /path/to/hello-mcp/src/index.ts
```

浏览器里应能看到 `echo` tool，手动传 `{ "text": "hi" }` 验证返回。stdio server 调通后再写入 Claude 配置。

---

## 注册到 Claude Code

### 用户级（本机所有项目）

```bash
claude mcp add hello-mcp -- npx tsx /absolute/path/to/hello-mcp/src/index.ts
```

或 `claude mcp add-json` 写入 `~/.claude.json`（[02 章](02-安装与快速上手.md) 格式）。

### 项目级（团队共享）

仓库根 `.mcp.json`：

```json
{
  "mcpServers": {
    "hello-mcp": {
      "type": "stdio",
      "command": "npx",
      "args": ["tsx", "./tools/hello-mcp/src/index.ts"]
    }
  }
}
```

**密钥不要进 Git：** PAT、数据库 URL 用环境变量：

```json
{
  "mcpServers": {
    "hello-mcp": {
      "type": "stdio",
      "command": "npx",
      "args": ["tsx", "./tools/hello-mcp/src/index.ts"],
      "env": {
        "INTERNAL_API_TOKEN": "${INTERNAL_API_TOKEN}"
      }
    }
  }
}
```

把 `.env` 与含真实 token 的本地 override 加入 `.gitignore`。Remote HTTP 自建服务同理，header 用 `headersHelper` 脚本（Claude Code 文档 · [第 15 章](/AI/Agent/ClaudeCode/第15章%20MCP.html#安全特性)）。

### CC Switch

- **桌面 GUI**：MCP 面板 → **+** → 表单填 Server ID、Transport、Command/URL 等 → 保存 → `cc-switch mcp sync`。
- **CLI `cc-switch mcp add` / `mcp edit <id>`**：打开 **`$EDITOR` 外部编辑器**（默认常为 vim）编辑 JSON，**不是** GUI 那种逐步表单。保存退出后再 `mcp sync`。
- 与 `claude mcp add` 最终都写入 CC Switch 统一库，再 sync 到 `~/.claude.json` 等 live 文件。

---

## 验证清单

| 步骤 | 命令 / 操作 |
|------|-------------|
| Inspector 通 | `npx @modelcontextprotocol/inspector …` |
| 配置写入 | `claude mcp list` 出现 `hello-mcp` |
| 会话连接 | Claude Code 内 `/mcp`，该 server **Connected**，tool 数 > 0 |
| 最小 prompt | 「用 hello-mcp 的 echo，回传 ping」 |

`/mcp` 里 **0 tools** 或 **Disconnected**：先看 stderr 日志、命令路径是否绝对路径、Node 版本、env 是否缺失（[09 章排错](09-治理与排错.md#连接与零工具)）。

---

## 设计建议（少 tool、好描述）

1. **一个 tool 一件事**——名称动词开头，description 写清输入 / 输出 / 失败时怎么办。  
2. **返回体控大小**——列表 tool 加 `limit`；大 JSON 写文件路径让 Agent 用 Read。  
3. **读写在 tool 层分开**——Postgres Pro 的 `access-mode` 思路可借鉴（[06 章](06-Postgres-MCP.md)）。  
4. **OAuth / PAT**——能走 Remote + `claude mcp login` 就别让 Agent 摸明文 token。  
5. **版本**——`name` / `version` 与 npm 包一致，方便 `/mcp` 对账。

---

## 发布与 Registry（可选）

团队内自用：Git 仓库 + `.mcp.json` 足够。  
要对社区分发：按 [MCP Registry](https://registry.modelcontextprotocol.io/) 提交元数据，并在 README 写清 Claude Code 的 `claude mcp add` 一行命令（参考 [github-mcp-server install-claude.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-claude.md)）。
