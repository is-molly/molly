---
order: 20
date: 2026-06-26
---

# Dynamic Workflows

## 什么是 Dynamic Workflows

Dynamic Workflows（动态工作流）是 Claude Code 的一项能力，Claude 可以**自主编写 JavaScript 编排脚本**，为当前任务动态生成定制的 Agentic Harness（代理编排层）。

传统 Claude Code Harness 是为编码任务设计的。Dynamic Workflows 让 Claude 为不同类型的任务（研究、安全分析、竞品评估等）**动态创建**专用的编排逻辑。

## 解决什么问题

长时间、大规模、高度结构化或对抗性任务中，单个上下文窗口会遇到三个典型失败模式：

| 问题 | 表现 |
|------|------|
| **Agentic Laziness** | 长期任务中 Claude 未完成全部工作就声明任务完成（如安全审查 50 项里只处理了 35 项） |
| **Self-preferential Bias** | Claude 倾向于偏好自己的结果，尤其是在被要求验证自己的产出时 |
| **Goal Drift** | 多轮对话后逐渐偏离原始目标，尤其是 compact 后细节丢失 |

Dynamic Workflows 通过**用独立的 Subagent 编排任务**来解决这些问题——每个 Subagent 有独立的上下文窗口和专注的独立目标。

## 工作原理

Dynamic Workflow 执行一个 JavaScript 文件，其中包含一组用于生成和协调 Subagent 的特殊函数。支持标准 JS（JSON、Math、Array 等）用于数据处理。

工作流可以决定：
- 每个 Subagent 使用哪个模型
- Subagent 是否在独立 worktree 中运行

如果工作流被中断（用户操作或退出终端），恢复会话后工作流可以从中断点继续。

## 示例 Prompts

```
This test fails maybe 1 in 50 runs. Set up a workflow to reproduce it. Form competing theories about the race, and don't stop until one theory survives the evidence.

Using a workflow, go through my last 50 sessions and mine them for corrections I keep making and turn the recurring ones into CLAUDE.md rules.

Use a workflow to dig through #incidents in Slack for the past six months and find recurring root causes where nobody has filed a ticket.

Take my business plan and run a workflow where different agents tear it apart from an investor's, a customer's, and a competitor's perspective.

Here's a folder of 80 resumes, use a workflow to rank them for the backend role and double-check the top ten.

I need a name for this CLI tool. Use a workflow to brainstorm a bunch of options and run a tournament to pick the top 3.

Go through my blog post draft and verify every technical claim against the codebase using a workflow, I don't want to ship anything wrong.
```

## 静态 vs 动态工作流

| | 静态工作流（Skills/Hooks） | 动态工作流 |
|------|------|------|
| 定义方式 | 提前写好的 SKILL.md / Hook | Claude 自主编写 JS 编排脚本 |
| 适用场景 | 重复性任务 | 一次性复杂任务 |
| Token 开销 | 低 | 高 |
| 灵活度 | 固定 | 极高 |

**使用建议**：Dynamic Workflows 会产生更多 Token 消耗，最适合复杂的、高价值的任务。对于可重复的任务，优先使用 Skills。

> **资料来源**：[Claude Code 官方博客 - Dynamic Workflows](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
