---
order: 28
date: 2026-06-26
---

# Agent SDK 与多仓库

## Claude Code Agent SDK

Claude Code 提供 Python 和 TypeScript SDK，让你在自己的应用中以编程方式使用 Claude Code 的能力：

- 自定义编排逻辑
- 完全控制工具访问和权限
- 构建自己的 Agentic 系统

详见 [Agent SDK 文档](https://platform.claude.com/docs/en/agent-sdk/overview)。

## 非交互模式（`-p`）

`-p`（print）模式是 Claude Code 的**脚本化调用**方式：

```shell
# 基础用法
claude -p "explain this function"

# 管道输入
cat logs.txt | claude -p "找出所有 ERROR 级别的日志并分析原因"

# JSON 输出（适合程序解析）
claude -p "列出 src/ 下所有导出的函数" --output-format json

# JSON Schema 约束输出
claude -p "分析代码" --json-schema '{"type":"object","properties":{"issues":{"type":"array"}}}'

# 限制 Token 预算
claude -p "修复测试" --max-budget-usd 5.00

# 限制回合数
claude -p "分析代码" --max-turns 3

# 禁用会话持久化
claude -p "生成代码" --no-session-persistence

# 继续上一次对话
claude -c -p "检查上次修改是否有类型错误"
```

## `--bare` 模式

在仓库的**任意子目录**中运行 Claude Code，而不需要是 Git 仓库根目录：

```shell
claude --bare
```

## `--add-dir` 多仓库工作

授予 Claude Code 访问额外目录的权限（验证每个路径是否存在且为目录）：

```shell
claude --add-dir ../libs ../shared-utils
```

`--add-dir` 目录中自动发现的配置项（Skills 的 `.claude/skills/` 除外）不会被加载。

## `--channels` MCP 通道

启用 Channels（将外部事件推送到 Claude Code 会话）：

```shell
claude --channels
```

## 系统提示词控制

```shell
# 替换整个系统提示词
claude --system-prompt "You are a Python expert who only writes type-annotated code"

# 追加到默认提示词（推荐，保留 Claude Code 能力）
claude --append-system-prompt "Always use TypeScript and include JSDoc comments"

# 从文件加载
claude -p --system-prompt-file ./prompts/code-review.txt "Review this PR"
```

## `--settings` 与 `--setting-sources`

加载额外设置文件或限制设置来源：

```shell
# 加载额外设置
claude --settings ./ci-settings.json -p "run tests"

# 限制设置来源
claude --setting-sources user,local
```

## `--output-format` 输出格式

| 格式 | 用途 |
|------|------|
| `text`（默认） | 人类可读文本 |
| `json` | JSON 输出，适合程序解析 |
| `stream-json` | 流式 JSON，适合实时处理（配合 `--include-partial-messages`） |

## `--session-id` 指定会话 ID

```shell
claude --session-id "550e8400-e29b-41d4-a716-446655440000"
```

## `--tools` 与 `--allowedTools`/`--disallowedTools`

```shell
# 限制可用工具
claude --tools "Bash,Edit,Read"

# 预批准工具（跳过确认）
claude --allowedTools "Bash(git log *)" "Bash(git diff *)" "Read"

# 移除工具
claude --disallowedTools "WebSearch" "WebFetch"
```

> **资料来源**：[Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)、[Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)
