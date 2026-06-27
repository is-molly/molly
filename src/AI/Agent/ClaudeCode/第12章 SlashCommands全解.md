---
order: 12
date: 2026-06-26
---

# Slash Commands 全解

Slash Commands（斜杠命令）是 Claude Code 交互界面中的内建命令，以 `/` 开头。Claude Code 支持**命令补全**——输入 `/` 后可按 Tab 浏览所有可用命令。

本章覆盖全部内建命令（部分内建 Skills 如 `/batch` `/loop` 见第 13 章）。

## 会话管理

| 命令 | 功能 | 用法 |
|------|------|------|
| `/clear` | 清空当前对话上下文，从零开始 | `/clear` |
| `/compact` | 压缩对话上下文，释放 Token 空间 | `/compact` |
| `/context` | 查看当前上下文使用情况 | `/context` |
| `/cost` | 查看当前会话 Token 消耗与费用（**API 用户**；Pro/Max 订阅用户不适用） | `/cost` |
| `/recap` | 查看当前会话摘要 | `/recap` |
| `/copy` | 复制最近一段对话内容到剪贴板 | `/copy` |
| `/resume` | 恢复之前的会话 | `/resume` |

## 配置与个性化

| 命令 | 功能 | 用法 |
|------|------|------|
| `/config` | 打开交互式设置界面（或 `/config key=value` 直接设置） | `/config` 或 `/config thinking=false` |
| `/model` | 切换模型（打开选择器或直接指定） | `/model` 或 `/model opus` |
| `/effort` | 调整 Effort 等级（打开滑块或直接指定） | `/effort` 或 `/effort xhigh` |
| `/color` | 设置会话颜色（不加参数随机选择） | `/color` |
| `/init` | 分析项目并自动生成 CLAUDE.md | `/init` |
| `/memory` | 编辑记忆文件 | `/memory` |
| `/permissions` | 管理权限规则 | `/permissions` |
| `/statusline` | 自定义状态栏 | `/statusline` |

## 代理管理

| 命令 | 功能 | 用法 |
|------|------|------|
| `/agents` | 打开子代理管理界面（查看、创建、编辑、删除） | `/agents` |
| `claude agents` | 查看所有会话的代理视图（研究预览） | 终端执行 `claude agents` |

## 插件与扩展

| 命令 | 功能 | 用法 |
|------|------|------|
| `/plugin` | 插件管理（安装、发现、更新） | `/plugin` 或 `/plugin install <name>` |
| `/mcp` | MCP Server 管理（查看状态、工具计数） | `/mcp` |
| `/sandbox` | 沙箱配置 | `/sandbox` |

## 代码与开发工作流

| 命令 | 功能 | 用法 |
|------|------|------|
| `/plan` | 以 Plan Mode 处理下一条 prompt（只读分析） | `/plan 重构认证模块` |
| `/branch [name]` | **会话分支**——在当前对话点 fork，保留原会话 | `/branch try-approach` |
| `/fork <directive>` | 后台 subagent 继承对话执行任务（v2.1.161+） | `/fork 跑完整测试套件` |
| `/background` | 将整个会话 detach 为 background agent | `/background` |
| `/btw <question>` | 旁路提问——**不写入主对话历史** | `/btw 这个 API 叫什么？` |

## 调试与反馈

| 命令 | 功能 | 用法 |
|------|------|------|
| `/bug` | 报告 Bug | `/bug` |
| `/feedback` | 提交反馈 | `/feedback` |
| `/help` | 显示帮助信息 | `/help` |
| `/doctor` | 诊断 Claude Code 安装状态 | `/doctor` |
| `/debug [描述]` | 启用调试日志（已有会话中途启用） | `/debug` |

## 内建 Skills（以 `/` 命令形式）

内建 Skills 是 Claude Code 自带的、以 `/` 命令形式调用的 Skills。与内建命令不同，Skills 是基于 prompt 的——Claude 根据详细的 Playbook 使用自己的工具来编排工作流。

| 内建 Skill | 功能 |
|------------|------|
| `/batch <指令>` | 大规模代码变更编排（研究代码库、拆分工作、在隔离 worktree 中并行执行） |
| `/claude-api` | 加载 Claude API 参考文档（Python/TypeScript/Java/Go/Ruby/C#/PHP/cURL + Agent SDK） |
| `/debug [描述]` | 启用调试日志（见上文） |
| `/loop [间隔] <prompt>` | 循环执行一个 prompt（如每 5 分钟检查一次部署状态） |
| `/simplify [关注点]` | 审查最近修改文件的质量和复用性，并行派 3 个审查 subagent，汇总结果并修复 |

### `/batch` 详解

```
/batch migrate src/ from Solid to React
```

- 研究代码库，将工作拆分为 5-30 个独立单元
- 展示计划，等待你批准
- 批准后，每个单元在隔离的 git worktree 中启动一个后台代理
- 每个代理实现自己的单元、运行测试、创建 PR
- 需要 git 仓库

### `/loop` 详解

```
/loop 5m check if the deploy finished
```

- 在会话保持打开期间按指定间隔重复执行 prompt
- 间隔支持 `s`（秒）、`m`（分）、`h`（小时）
- 适用于轮询部署、追踪 PR 状态、定期重新运行 Skill 等场景

## 自定义 Slash Commands

在 `.claude/commands/` 目录下创建 markdown 文件即可定义自定义命令。从某个版本开始，自定义命令已合并进 Skills 体系（`.claude/skills/` 目录），但 `commands/` 中的文件仍然有效。

详见第 13 章 Skills。

## 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Shift+Tab` | 切换权限模式 |
| `Option+T` / `Alt+T` | 切换 Extended Thinking |
| `Ctrl+O` | 切换详细思考模式 |
| `Ctrl+B` | 将**当前 Bash 命令或 subagent** 移到后台（不是整个会话；全会话用 `/background`） |
| `Ctrl+V` | 粘贴图片 |

> **资料来源**：[Claude Code Skills 文档](https://code.claude.com/docs/en/skills)、[CLI Reference](https://code.claude.com/docs/en/cli-reference)、[Commands](https://code.claude.com/docs/en/commands)
