---
order: 5
date: 2026-06-27
---

> Gitee：[oschina/mcp-gitee](https://github.com/oschina/mcp-gitee)（2026-06 clone 核对）。GitHub：[github/github-mcp-server](https://github.com/github/github-mcp-server)。**勿用** 已废弃 npm `@modelcontextprotocol/server-github`。

# 代码托管 MCP：Gitee 与 GitHub

---

## 是什么

<a id="是什么"></a>

**代码托管 MCP** 让 Agent 通过 **MCP tool** 操作 Gitee / GitHub **平台 API**——Issue、PR、Review、读远程文件、通知等。它**不替代 git**：本地 `commit` / `push` 仍用 Bash。

| 产品 | 上游 | 传输 | Agent 侧 |
|------|------|------|----------|
| **Gitee MCP** | [oschina/mcp-gitee](https://github.com/oschina/mcp-gitee) | **Remote HTTP** + 私人令牌 | 约 24 个 tool（可 Header 收窄） |
| **GitHub MCP** | [github/github-mcp-server](https://github.com/github/github-mcp-server) | Remote OAuth 或 Docker stdio | 100+ tool（可按 toolsets 缩） |

**会话里长什么样（Gitee）：** 「列出我有权限的仓库」→ tool **`list_user_repos`**；「在 foo/bar 开一个 bug Issue」→ **`create_issue`**。主用 Gitee 时通常 **只常开 Gitee Remote**；GitHub 按需再装。

**和 `gh` CLI：** GitHub 用户部分 Issue/PR 可用 **`gh`**（Bash）；Gitee 官方主推 **Remote MCP**。详见 [git 与 MCP 的分工](#git-与-mcp-的分工)。

---

## git 与 MCP 的分工

**git / Bash** 管本地仓库；**MCP** 管托管平台上的 Issue、PR、通知等 API 对象。

| 任务 | git / Bash | Gitee/GitHub MCP |
|------|------------|------------------|
| 改文件、commit、push | ✓ | — |
| 看本地 diff、log | ✓ | — |
| 创建 / 更新 Issue | — | ✓ |
| 创建 / 合并 PR、Review | — | ✓ |
| 读远程仓库文件内容 | 可 `git show` | ✓ `get_file_content`（Gitee） |
| 搜 Gitee 开源库 | — | ✓ `search_open_source_repositories` |

GitHub 用户还可用官方 **`gh` CLI** 做部分 Issue/PR 操作；Gitee 官方主推 **Remote MCP**。

---

## Gitee Remote MCP

### 连接参数（上游 README + `docs/install/claude.md`）

| 项 | 值 |
|----|-----|
| URL | `https://api.gitee.com/mcp` |
| 类型 | HTTP（Claude Code JSON 里 `"type": "http"`） |
| 鉴权 | Header `Authorization: Bearer <私人令牌>` |
| 令牌申请 | https://gitee.com/profile/personal_access_tokens |

### 推荐：CC Switch

**桌面 GUI：** MCP 面板 → **+** → Custom → Transport **http** → URL `https://api.gitee.com/mcp` → Header `Authorization: Bearer <token>` → 勾选 Claude → 保存。

**CLI `cc-switch mcp add`：** 会打开 **`$EDITOR`（常为 vim）** 编辑 JSON 模板，不是逐步问答向导。填好 `id`、`name`、`url`、`headers` 后保存退出，再 `cc-switch mcp sync`。字段示例见 [02 · Gitee 安装](02-安装与快速上手.md#gitee)。

**已有 Claude 配置：** `cc-switch mcp import --app claude` 从 `~/.claude.json` 导入，再在 CC Switch 里改。

项目级共享：在仓库根建 `.mcp.json`，令牌用 `${GITEE_TOKEN}` 或本机环境变量，并把 `.mcp.json` 加入 `.gitignore`（若含明文则勿提交）。

**验证** — 新开 Claude Code 会话，`/mcp` 里 `gitee` 工具数 > 0；或 `claude mcp get gitee`。

**收窄 tool（推荐）** — Header 加 `X-MCP-Enabled-Tools`（见 [推荐 tool 子集](#推荐-tool-子集)）。

### 令牌权限

Gitee MCP 调的是 Open API v5；令牌需覆盖你要让 Agent 做的操作。经验法则：

| 你要 Agent 做的事 | 令牌侧通常需要 |
|------------------|----------------|
| 列仓库、读文件 | 仓库 / 项目 相关读权限 |
| Issue 列表、创建、关闭 | Issue 读写 |
| PR 创建、合并、Review | Pull Request 读写 |
| 评论 Issue/PR | 评论 / 通知 类权限（与 Issue/PR 配套） |
| 仅只读协作 | **不要**勾选创建仓库、删仓库等写权限；用 `X-MCP-Enabled-Tools` 再收窄 |

创建时在 Gitee 令牌页 **最小权限** 勾选；某 tool 报 403 再补对应 scope。令牌泄露时在 Gitee 后台 **立即吊销** 并换新 token，再 `mcp sync`。

### 推荐 tool 子集

24 个 tool 全开会占 context；按场景在 Header 里白名单：

**单人全栈（Issue + PR + 读仓库，默认推荐）**

```text
list_user_repos,get_file_content,list_repo_issues,get_repo_issue_detail,create_issue,update_issue,list_repo_pulls,get_pull_detail,create_pull,create_comment,list_comments,get_user_info
```

**只读看板（省 token）**

```text
list_user_repos,get_file_content,list_repo_issues,list_repo_pulls,get_pull_detail,list_user_notifications
```

**含合并 PR（需你接受误 merge 风险时再开）**

在上面的基础上加：`merge_pull,manage_pull_review,update_pull,get_diff_files`

配置写在 `headers.X-MCP-Enabled-Tools`，逗号分隔、**区分大小写**（与 [工具列表](#gitee-工具列表) 一致）。

### 备选：`claude mcp add-json`

```bash
claude mcp add-json gitee '{"type":"http","url":"https://api.gitee.com/mcp","headers":{"Authorization":"Bearer YOUR_GITEE_TOKEN"}}'
```

项目共享（令牌仍用环境变量，勿提交）：

```bash
claude mcp add-json gitee '…' --scope project
```

### 验证

```bash
claude mcp list
claude mcp get gitee
```

`/mcp` 中 `gitee` 工具数应 **大于 0**。可试：「列出我有权限的 Gitee 仓库」→ 应调用 **`list_user_repos`**。

### 本地 stdio（一般不必，Remote 优先）

上游支持 Go 二进制 `mcp-gitee`，flag（README **Command-line Options**）：

| Flag / 环境变量 | 含义 |
|-----------------|------|
| `--token` / `GITEE_ACCESS_TOKEN` | 令牌 |
| `--transport` | `stdio`、`sse`、`http`（默认 stdio） |
| `--enabled-toolsets` / `ENABLED_TOOLSETS` | 白名单 tool 名，逗号分隔 |
| `--disabled-toolsets` / `DISABLED_TOOLSETS` | 黑名单 |

---

## Gitee 工具列表

名称以下游 README **Available Tools** 为准（2026-06，共 24 个）：

<a id="gitee-工具列表"></a>

| Tool | 分类 | 说明 |
|------|------|------|
| `list_user_repos` | Repository | 列出当前用户有权限的仓库 |
| `get_file_content` | Repository | 读仓库内文件内容 |
| `create_repo` | Repository | 创建仓库（用户/组织/企业） |
| `fork_repository` | Repository | Fork |
| `create_release` | Repository | 创建 Release |
| `list_releases` | Repository | 列 Release |
| `search_open_source_repositories` | Repository | 搜 Gitee 开源库 |
| `search_files_by_content` | Repository | 按内容搜文件 |
| `compare_branches_tags` | Repository | 比较分支/标签/commit |
| `list_repo_pulls` | Pull Request | 列 PR |
| `merge_pull` | Pull Request | 合并 PR |
| `create_pull` | Pull Request | 创建 PR |
| `update_pull` | Pull Request | 更新 PR |
| `get_pull_detail` | Pull Request | PR 详情 |
| `get_diff_files` | Pull Request | PR diff 文件列表 |
| `manage_pull_review` | Pull Request | 通过或取消 Review |
| `create_comment` | Comment | 评论 Issue 或 PR |
| `list_comments` | Comment | 列评论 |
| `create_issue` | Issue | 创建 Issue |
| `update_issue` | Issue | 更新 Issue |
| `get_repo_issue_detail` | Issue | Issue 详情 |
| `list_repo_issues` | Issue | 列 Issue |
| `get_user_info` | User | 当前认证用户 |
| `search_users` | User | 搜用户 |
| `list_user_notifications` | Notification | 用户通知 |

源码里 tool 注册在 `mcp-gitee/operations/` 各子目录；名称与上表一致（如 `operations/repository/list_user_repos.go`）。

### 缩小 tool 面（Remote）

README **Per-Request Tool Filtering**：HTTP header

- 白名单：`X-MCP-Enabled-Tools: list_user_repos,list_repo_issues,create_issue`
- 黑名单：`X-MCP-Disabled-Tools: create_repo,merge_pull`（README 示例里出现的 `delete_repo` **不在**当前 24 tool 列表中，勿照抄无效名）
- 同时存在时 **白名单优先**

配置示例：

```json
{
  "mcpServers": {
    "gitee": {
      "type": "http",
      "url": "https://api.gitee.com/mcp",
      "headers": {
        "Authorization": "Bearer …",
        "X-MCP-Enabled-Tools": "list_user_repos,list_repo_issues,create_issue,list_repo_pulls,create_pull"
      }
    }
  }
}
```

---

<a id="github-mcp"></a>

## GitHub MCP（按需）

### Remote（上游 `docs/installation-guides/install-claude.md`）

| 项 | 值 |
|----|-----|
| URL | `https://api.githubcopilot.com/mcp` |
| 鉴权 | `Authorization: Bearer <GitHub PAT>` |

Claude Code ≥ 2.1.1：

```bash
claude mcp add-json github '{"type":"http","url":"https://api.githubcopilot.com/mcp","headers":{"Authorization":"Bearer YOUR_GITHUB_PAT"}}'
```

Legacy（≤ 2.1.0）：

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp -H "Authorization: Bearer YOUR_GITHUB_PAT"
```

本地 Docker（备选）：

```bash
claude mcp add github -e GITHUB_PERSONAL_ACCESS_TOKEN=YOUR_GITHUB_PAT -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

### 默认 toolsets（未指定时）

`github-mcp-server` README：**default** = `context` + `repos` + `issues` + `pull_requests` + `users`。

可用 toolset 还包括（README **Available Toolsets** 自动生成列表）：`actions`、`code_security`、`dependabot`、`discussions`、`notifications`、`orgs`、`projects`、`git`、`gists`、`labels`、`stargazers`、`secret_protection`、`security_advisories` 等；`all` 表示全部。

Remote 额外：`copilot`、`copilot_spaces`、`github_support_docs_search`。

缩小示例：

**本地 Docker / binary** — 环境变量 `GITHUB_TOOLSETS`（或 `--toolsets` flag）：

```bash
GITHUB_TOOLSETS="repos,issues,pull_requests" …
```

**Remote HTTP** — 请求头 `X-MCP-Toolsets`（与本地 `GITHUB_TOOLSETS` 等价，见上游 `docs/remote-server.md`）：

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp",
      "headers": {
        "Authorization": "Bearer …",
        "X-MCP-Toolsets": "repos,issues,pull_requests"
      }
    }
  }
}
```

只读可加 URL 后缀 `/readonly`，或 header 里设 read-only 相关字段（见 upstream `remote-server.md`）。具体 tool 名（100+）见上游 README **Tools** 折叠块；写 automation 时以 `claude mcp get github` / `/mcp` 列出的为准。

### GitHub 与 `gh` CLI

| | `gh` CLI | GitHub MCP |
|--|----------|------------|
| 安装 | `brew install gh` 等 | Remote 或 Docker |
| 典型 | 脚本化 PR、Issue | Agent tool 调用、toolset 筛选 |
| Token | `gh auth login` | PAT 在 MCP header |

两个平台都在用时：**不要** Gitee + GitHub MCP 全 toolset 同时常开。

---

## Gitee 与 GitHub 对照

| | Gitee | GitHub |
|--|-------|--------|
| 上游 | oschina/mcp-gitee | github/github-mcp-server |
| Remote URL | `https://api.gitee.com/mcp` | `https://api.githubcopilot.com/mcp` |
| 工具数（README 级） | 24 个命名 tool | 按 toolset，默认 5 组 |
| 收窄方式 | `X-MCP-Enabled-Tools` 等 | `GITHUB_TOOLSETS` / header |
| 废弃包 | — | `@modelcontextprotocol/server-github` |

---

## 上游文档里的 Issue → PR 流程（Gitee）

README **Practical scenario** 三步（配图在仓库 `docs/images/`）：

1. `list_repo_issues` 取 Issue  
2. 本地改代码 + `create_pull`  
3. `create_comment` + 关闭 Issue（`update_issue`）

会话里可以一句描述让 Claude 串起来，但 **改代码仍用 Edit/Bash**，只有平台操作走 MCP。

---

## 对话示例

### 示例 A：列 Issue 并实现

**你说：**

```text
在 Gitee 仓库 owner/repo 里列 open 的 Issue，选最旧的一个，按描述改代码，完成后开 PR 并在 Issue 里说明。
```

**期望分工：**

| 步骤 | 通道 | Tool / 动作 |
|------|------|-------------|
| 列 Issue | MCP | `list_repo_issues` |
| 读 Issue 详情 | MCP | `get_repo_issue_detail` |
| 改本地代码 | 内置 | Read / Edit |
| 提交 | Bash | `git commit` / `git push` |
| 开 PR | MCP | `create_pull` |
| 留言 | MCP | `create_comment`；可选 `update_issue` 关 Issue |

### 示例 B：只看 PR diff，不动手合并

**你说：**

```text
列出 owner/repo 上 open 的 PR，把最新一个的 diff 文件列表给我，不要合并。
```

期望：`list_repo_pulls` → `get_pull_detail` / `get_diff_files`；**不应**调用 `merge_pull`（tool 子集里也可去掉 merge 相关 tool）。

### 示例 C：远程读文件（未 clone）

**你说：**

```text
从 Gitee 读 owner/repo 默认分支上的 src/config.ts，不要 clone 整个仓库。
```

期望：`get_file_content`；若本地已有 clone，Agent 应优先 **Read** 而不是 MCP。

---

## 常见问题

| 现象 | 处理 |
|------|------|
| `/mcp` 工具数为 0 | Bearer 是否正确；URL 是否为 `https://api.gitee.com/mcp`；是否 `cc-switch mcp sync`；Claude 是否新开会话 |
| 401 / 403 | 令牌过期或 scope 不足；在 Gitee 后台检查权限或换 token |
| Agent 用 MCP 读本地已有文件 | 提醒「仓库已 clone，请 Read/grep」；或收窄掉 `get_file_content` |
| 和 GitHub MCP 重复占 context | 主 Gitee 只 enable gitee；GitHub 项目再临时开 github |
| 误创建仓库 / 误 merge | 收窄 `X-MCP-Enabled-Tools`；去掉 `create_repo`、`merge_pull` |
| 读文件用 MCP 还是 git | 已 clone → Read/grep；未 clone 或看远程默认分支 → `get_file_content` |
