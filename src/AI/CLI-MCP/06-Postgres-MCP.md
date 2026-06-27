---
order: 6
date: 2026-06-27
---

> 上游：[crystaldba/postgres-mcp](https://github.com/crystaldba/postgres-mcp)（PyPI **`postgres-mcp`**，Docker **`crystaldba/postgres-mcp`**）。2026-06 clone 核对。勿用已归档 npm `@modelcontextprotocol/server-postgres`。

# Postgres MCP

---

## 是什么

<a id="是什么"></a>

**Postgres MCP Pro**（[crystaldba/postgres-mcp](https://github.com/crystaldba/postgres-mcp)）是面向 **DBA / 性能调优** 的 MCP Server：不只跑 `SELECT`，而是封装 **`pg_stat_statements`、EXPLAIN、hypopg 假索引、工作负载索引推荐、库健康检查** 等能力，让 Agent 能回答「为什么慢、该加什么索引」。

| | Bash + `psql` | 归档 `@modelcontextprotocol/server-postgres` | **Postgres MCP Pro（本章）** |
|--|---------------|---------------------------------------------|------------------------------|
| 一条 ad-hoc SQL | ✓ | ✓ 只读 | ✓ `execute_sql`（restricted 模式受控） |
| EXPLAIN + 假索引对比 | 手工 | ✗ | ✓ `explain_query` |
| Top 慢查询 | 手工查视图 | ✗ | ✓ `get_top_queries` |
| 索引推荐 | ✗ | ✗ | ✓ `analyze_workload_indexes` |

**会话里长什么样：** 「这条查询为什么慢」→ **`explain_query`**；「库连接数、缓存命中率正常吗」→ **`analyze_db_health`**。

**何时不必装：** 项目不用 Postgres；或偶尔 `psql -c 'SELECT …'` 就够——走 Bash（[01 · CLI](01-概念与选型.md#cli-通道)）。

**连接方式：** 启动时固定 **`DATABASE_URI`**（stdio / Docker），**没有** Gitee 那种 Remote OAuth SaaS。生产库用 **`--access-mode=restricted`** 只读。

---

## 和 Bash / 其它 MCP 怎么分

| 任务 | 用谁 |
|------|------|
| 本地跑一条 SQL、看表结构 | **Bash** `psql`（[02 章](01-概念与选型.md)） |
| 应用 ORM 改 schema、写 migration | **内置 Edit** + 项目 migration 工具 |
| 慢查询 Top N、`pg_stat_statements` 分析 | **Postgres MCP** `get_top_queries` |
| EXPLAIN + hypothetical index（hypopg） | **Postgres MCP** `explain_query` |
| 工作负载索引推荐（DTA 算法） | **Postgres MCP** `analyze_workload_indexes` |
| 缓存命中率、vacuum、连接数健康检查 | **Postgres MCP** `analyze_db_health` |
| Supabase **控制台**管项目、分支、Auth | [Supabase MCP](https://github.com/supabase-community/supabase-mcp)（管理 API，不是本章） |
| Neon **控制台**管 branch / compute | [Neon MCP](https://github.com/neondatabase-labs/mcp-server-neon) |

**选型：** 一条 SQL 能答 → Bash + `psql`；要 EXPLAIN、索引、健康检查 → 本章 Postgres MCP Pro；管 Supabase / Neon **控制台** → 各自 MCP，不是本章替代品。

**没有 Remote OAuth SaaS**：连接串在启动 MCP 时固定（`DATABASE_URI` 或 positional URL）。适合 **连你本机 / 内网 / 开发库**；生产库务必用 **`restricted`** 只读模式。

---

## 和归档 reference server 的区别

| | `@modelcontextprotocol/server-postgres`（归档） | Postgres MCP Pro（本章） |
|--|--------------------------------------------------|---------------------------|
| Schema | MCP **Resources** 暴露 | **Tools**：`list_schemas`、`get_object_details` 等 |
| SQL | 只读查询 | `execute_sql` + 安全解析 |
| 调优 | 无 | `explain_query`、`analyze_*_indexes`、`get_top_queries`、`analyze_db_health` |
| 算法 | — | 索引搜索（Anytime 类贪心 + hypopg 模拟） |

---

## 传输方式

| 方式 | 说明 |
|------|------|
| **stdio（默认）** | Claude Code / CC Switch 启动子进程；最常用 |
| **SSE** | `--transport=sse`，`http://host:8000/sse`，多客户端共享同一 server |
| **streamable-http** | 源码支持（`server.py` `--transport=streamable-http`） |

Claude Code 日常用 **stdio + Docker 或 uvx** 即可。

---

## 安装（Claude Code）

### 前置

1. 可用的 **`DATABASE_URI`**（`postgresql://user:pass@host:5432/dbname`）
2. Docker **或** Python 3.12+（上游推荐 Docker）
3. 索引/慢查询分析：库上启用扩展 **`pg_stat_statements`**、**`hypopg`**（见下节）

### 推荐：Docker + restricted（连生产或共享库）

```bash
claude mcp add postgres \
  -e DATABASE_URI=postgresql://USER:PASS@localhost:5432/mydb \
  -- docker run -i --rm -e DATABASE_URI crystaldba/postgres-mcp --access-mode=restricted
```

开发库可改 **`--access-mode=unrestricted`**（允许写 SQL，destructive）。

Docker 镜像会把容器内的 `localhost` 映射到宿主机（Mac/Windows 用 `host.docker.internal`；Linux 用 `172.17.0.1` 等，见 README）。

### 备选：uvx / pipx

```bash
claude mcp add postgres \
  -e DATABASE_URI=postgresql://USER:PASS@localhost:5432/mydb \
  -- uvx postgres-mcp --access-mode=restricted
```

或 `pipx` / `uv run postgres-mcp`（README 各有一段 Claude Desktop JSON，Claude Code 用 `claude mcp add` 等价）。

### CC Switch

stdio + Docker 与 Gitee Remote 不同：**GUI** 里添加 **本地命令** 型 MCP（command/args/env）；**CLI `mcp add`** 则在外部编辑器里填同类 JSON。然后 `mcp sync`。

`DATABASE_URI` 用环境变量或 CC Switch 密钥字段，**勿提交 git**。也可直接用上文 **`claude mcp add`**，再 `cc-switch mcp import --app claude` 迁入 CC Switch。

### 验证

```bash
claude mcp list
claude mcp get postgres
```

会话 `/mcp`：`postgres` 已连接，约 **9** 个 tool。试：

```text
列出数据库里所有 schema。
```

应对应 **`list_schemas`**。

---

## Access Mode（必懂）

| 模式 | 参数 | 适用 |
|------|------|------|
| **unrestricted** | `--access-mode=unrestricted` | 开发库；`execute_sql` 可读写；tool 标注 destructive |
| **restricted** | `--access-mode=restricted` | 生产/ staging 只读；只读事务 + SQL 解析拒绝 `COMMIT`/`ROLLBACK` 绕过；有执行时间限制 |

源码（`server.py`）：restricted 下 `execute_sql` 描述为 read-only，`readOnlyHint=True`；unrestricted 为 `destructiveHint=True`。

**不要用 unrestricted 连含真实用户数据的生产库**，除非你能接受 Agent 误 `UPDATE`/`DELETE`。

---

## 可选扩展（索引与慢查询）

在数据库执行（需足够权限；RDS/Cloud SQL 通常已带扩展包）：

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS hypopg;
```

| 扩展 | 作用 |
|------|------|
| `pg_stat_statements` | `get_top_queries`、工作负载分析 |
| `hypopg` | `explain_query` / 索引推荐里模拟「假如加了这条索引」 |

自管 Postgres：`pg_stat_statements` 可能要在 `shared_preload_libraries` 里预加载；`hypopg` 有时要系统包安装。无扩展时，部分 tool 会报错或能力受限——先 `list_objects` / `execute_sql` 查 `pg_extension`。

---

## 九个 MCP Tool（README · MCP Server API）

Postgres MCP Pro **只用 Tools**，不用 Resources（与 reference server 对比，README 刻意选型以兼容更多 MCP client）。

| Tool | 干什么 |
|------|--------|
| `list_schemas` | 所有 schema |
| `list_objects` | 某 schema 下 table / view / sequence / extension |
| `get_object_details` | 表/视图的列、约束、索引 |
| `execute_sql` | 执行 SQL（restricted 下只读） |
| `explain_query` | EXPLAIN；可带 hypothetical indexes |
| `get_top_queries` | 按 `pg_stat_statements` 报最慢 SQL |
| `analyze_workload_indexes` | 分析工作负载，推荐索引 |
| `analyze_query_indexes` | 对最多 **10** 条指定 SQL 推荐索引 |
| `analyze_db_health` | 缓存、连接、约束、索引健康、vacuum、序列、复制等 |

### 典型调用顺序

**排查慢应用：**

1. `analyze_db_health` 或 `get_top_queries`
2. 对可疑 SQL → `explain_query`
3. 要自动索引方案 → `analyze_workload_indexes` 或 `analyze_query_indexes`

**写业务 SQL 前看 schema：**

1. `list_schemas` → `list_objects` → `get_object_details`
2. 再 Edit 代码或 `execute_sql`（验证用 restricted）

---

## 对话示例

### 示例 A：数据库健康

**你说：**

```text
检查 Postgres 数据库健康状况，有没有索引膨胀、缓存命中率低或 vacuum 问题。
```

**期望：** `analyze_db_health`（可带 `health_type` 参数筛某一类检查，见 tool schema）。

### 示例 B：最慢查询

**你说：**

```text
这个库里最慢的 SQL 是哪些？怎么加速？
```

**期望：** `get_top_queries` → 对 Top 语句 `explain_query` → 必要时 `analyze_query_indexes`。

### 示例 C：优化指定 SQL

**你说：**

```text
优化这条查询：
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id WHERE o.created_at > '2024-01-01';
```

**期望：** `explain_query` → `analyze_query_indexes(queries=[...])`；输出索引建议与 hypopg 模拟结果（扩展已装时）。

### 示例 D：ORM 页面慢（README Demo 场景）

**你说：**

```text
movie 列表页 ORM 查询要 3 秒，数据库是本地 Postgres，帮我找缺失索引。
```

**期望：** schema 工具定位表 → `get_top_queries` 或你贴 ORM 生成的 SQL → `analyze_query_indexes`。**改代码**仍用 Read/Edit，不要指望 MCP 直接改 repo。

---

## 安全与治理

| 风险 | 做法 |
|------|------|
| 连接串进 git | `DATABASE_URI` 只放 env / CC Switch 密钥；`.mcp.json` 含 URI 则 **gitignore** |
| Agent 删数据 | 生产用 **`restricted`**；或 DB 用户只读权限 |
| 长查询拖垮库 | restricted 有执行时间上限（README） |
| 多库切换 | 本 server **启动时绑一个 URI**；换库要改配置或再起一个 MCP 实例 |
| 凭证经 LLM | 不要把 URI 贴在对话里；用 env 注入 |

实验功能 **Index Tuning by LLM** 需 `OPENAI_API_KEY`（README Technical Notes）；与 deterministic DTA 算法并列，非默认路径。

---

## 常见问题

| 现象 | 处理 |
|------|------|
| MCP 连上但 tool 报 DB 错误 | `DATABASE_URI` 是否正确；从 Claude 宿主机能否 `psql` 连通 |
| Docker 连不上 localhost DB | 用 `host.docker.internal:5432`（Mac/Win）或宿主机 IP |
| `get_top_queries` 空 | 是否 `CREATE EXTENSION pg_stat_statements` 并有查询负载 |
| 索引推荐不可用 | 是否安装 `hypopg` |
| 和 `psql` 重复 | 简单 ad-hoc 查询用 Bash；调优/EXPLAIN 用 MCP |
| 想要 Supabase 管项目 | 另装 Supabase MCP，不是本章替代品 |
| 归档 npm postgres server | 迁移到 **crystaldba/postgres-mcp** |