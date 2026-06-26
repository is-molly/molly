---
order: 14
date: 2026-06-26
---

# Hooks

## 什么是 Hook

Hook 是 Claude Code 的**生命周期拦截机制**——在特定事件发生时自动执行一段 Shell 命令或向 LLM 发送评估请求。Hook 可以用于自动格式化、代码检查、通知、自定义审批流等。

Hook 配置在 `settings.json` 中的 `hooks` 块：

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here"
          }
        ]
      }
    ]
  }
}
```

## Hook 类型

### Bash 命令 Hook（`type: "command"`）

执行 Shell 脚本。适合**确定性规则**（如格式化、检查）。

### Prompt Hook（`type: "prompt"`）

向快速 LLM（Haiku）发送评估请求，由 LLM 返回结构化 JSON 决策。适合**上下文感知的决策**（如"任务是否完成"）。

**对比：**

| 特性 | Bash 命令 Hook | Prompt Hook |
|------|---------------|-------------|
| 执行方式 | 运行 Shell 脚本 | 查询 LLM |
| 决策逻辑 | 代码实现 | LLM 上下文评估 |
| 上下文感知 | 有限 | 自然语言理解 |
| 性能 | 快（本地执行） | 较慢（API 调用） |
| 适用场景 | 确定性规则 | 上下文感知决策 |

## 全部 Hook 事件

### 工具级事件

| 事件 | 触发时机 | 支持 Matcher |
|------|----------|-------------|
| **PreToolUse** | Claude 创建工具参数后、执行前 | 是（`Task`/`Bash`/`Glob`/`Grep`/`Read`/`Edit`/`Write`/`WebFetch`/`WebSearch`） |
| **PermissionRequest** | 用户被显示权限对话框时 | 是（同上） |
| **PostToolUse** | 工具执行成功后 | 是（同上） |
| **Notification** | Claude 发送通知时 | 是（`permission_prompt`/`idle_prompt`/`auth_success`/`elicitation_dialog`） |

### 会话级事件

| 事件 | 触发时机 | Matcher |
|------|----------|---------|
| **UserPromptSubmit** | 用户提交 prompt 后、Claude 处理前 | 无 |
| **SessionStart** | 新会话启动或恢复时 | `startup`/`resume`/`clear`/`compact` |
| **SessionEnd** | 会话结束时 | 无（`reason` 字段区分：`clear`/`logout`/`prompt_input_exit`/`other`） |
| **Stop** | 主 Claude 代理完成回复时 | 无（用户中断时不触发） |
| **SubagentStop** | 子代理完成回复时 | 无 |
| **PreCompact** | 上下文压缩前 | `manual`/`auto` |

### 文件与配置事件

| 事件 | 类型 | 触发时机 |
|------|------|----------|
| **CwdChanged** | 文件级 | 当前工作目录改变时 |
| **FileChanged** | 文件级 | 指定文件内容变化时 |
| **ConfigChange** | 配置级 | 配置变更时 |

### 工作区事件

| 事件 | 触发时机 |
|------|----------|
| **WorktreeCreate** | Git Worktree 创建时 |
| **WorktreeRemove** | Git Worktree 删除时 |

## Matcher 匹配规则

Matcher 用于 `PreToolUse`、`PostToolUse`、`Notification` 等工具级事件，决定 Hook 对哪些工具生效：

- 精确匹配：`Write` 仅匹配 Write 工具
- 正则：`Edit|Write` 或 `Notebook.*`
- 全部：`*` 或空字符串
- 大小写敏感

**不**使用 Matcher 的事件（`UserPromptSubmit`、`Stop`、`SubagentStop`）可省略该字段。

## Hook 输入/输出

### 输入（stdin JSON）

所有 Hook 接收公共字段：

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/dir",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/to/file.txt", "content": "file content" }
}
```

### 输出

**简单模式（退出码）**：
- `0`：允许操作继续
- `2`：阻止操作，将 stderr 作为原因显示给 Claude
- 其他非零：阻止操作

**JSON 模式**：
```json
{
  "decision": "approve" | "block",
  "reason": "说明",
  "continue": false,
  "stopReason": "显示给用户的消息",
  "systemMessage": "警告或上下文"
}
```

## 条件触发（`if` 字段）

v2.1.85 起支持 `if` 字段，使用权限规则语法过滤 Hook 触发条件：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(git *)",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Git command detected'"
          }
        ]
      }
    ]
  }
}
```

## 典型场景

### 代码自动格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write ${CLAUDE_PROJECT_DIR}/src/**/*.ts"
          }
        ]
      }
    ]
  }
}
```

### 提交前 Lint 检查

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(git commit *)",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint && npm run test"
          }
        ]
      }
    ]
  }
}
```

### SessionStart 环境设置

利用 `CLAUDE_ENV_FILE` 持久化环境变量：

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
  echo 'export PATH="$PATH:./node_modules/.bin"' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

## Plugin Hooks

Plugin 可以在 `hooks/hooks.json` 中声明 Hook。Plugin Hook 使用 `${CLAUDE_PLUGIN_ROOT}` 引用 Plugin 文件：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh"
          }
        ]
      }
    ]
  }
}
```

Plugin Hook 与用户/项目 Hook 并行运行——多个来源的同一事件 Hook 都会执行。

## 项目脚本路径

使用 `$CLAUDE_PROJECT_DIR` 引用项目中的 Hook 脚本：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-style.sh"
          }
        ]
      }
    ]
  }
}
```

> **资料来源**：[Claude Code 官方文档 - Hooks](https://code.claude.com/docs/en/hooks)
