---
order: 12
date: 2026-06-27
---

# 开发高质量 Skill

> **阅读路径**：先会装、会选（[02](02-管理架构与快速上手.md)、[03](03-选型与组合方案.md)）→ [01 基础](01-基础与生态概览.md) → 本章。

## 本章定位

前面章节（[01–11](README.md)）讲的是「用什么」——生态、方法论、现有仓库。本章讲「怎么做」——从零开始写一个高质量的 Skill。

本章内容来源：
- Anthropic 官方 PDF《The Complete Guide to Building Skills for Claude》
- Anthropic 内部博客的 9 条工程技巧（[04 Anthropic 方法论](04-Anthropic内部Skills方法论.md) 已详述，本章聚焦实操）
- Superpowers 的 `writing-skills` Skill 源码
- mattpocock/skills 中 21 个活跃 Skill 的共同设计模式
- skill-creator Plugin 的 Eval/Benchmark 工作流

## Skill 开发的完整流程

### Step 1：确定类型归属

在动手之前，先想清楚你要创建的 Skill 属于 Anthropic 的 9 类中的哪一类。这决定了 Skill 的整体结构和指令风格。最好的 Skill 干净地属于一个类型——试图兼顾多个类型的 Skill 会让 Agent 困惑。

如果你无法把 Skill 干净地归入一个类型，可能意味着你试图让一个 Skill 做太多事情。拆成多个。

### Step 2：定义触发条件

这是开发 Skill 最关键的一步。想清楚：

- 用户在什么场景下会说些什么，Claude 该意识到该用这个 Skill 了？
- 触发的关键词和意图模式是什么？

把这些写成 `description` 字段。记住 Anthropic 的原则：**写给模型看，不是写给人看**。好的 description 同时包含"做什么"和"什么时候用"。

### Step 3：创建目录结构

最小结构：

```
my-skill/
└── SKILL.md
```

需要渐进式披露时的结构：

```
my-skill/
├── SKILL.md              # 主体指令
├── references/
│   └── gotchas.md        # 易错点（随着使用不断积累）
├── scripts/
│   └── helper.py         # 可执行脚本
└── assets/
    └── template.md       # 输出模板
```

### Step 4：编写 SKILL.md

**YAML frontmatter** 必须包含：

```yaml
---
name: my-skill
description: What it does and when to use it. Use when the user asks for X, mentions Y, or says Z.
when_to_use: Additional trigger phrases. Appended to description, shared 1536 char cap.
---
```

可选配置：

```yaml
disable-model-invocation: true    # 只允许手动 /command 调用
user-invocable: false             # 从 / 菜单隐藏，仅模型可调用
allowed-tools: Read, Grep         # 限制权限范围
disallowed-tools: AskUserQuestion # 禁止特定工具
model: sonnet
effort: high                      # 或 max
context: fork                     # 隔离子代理执行
agent: Explore
paths: "src/**/*.ts"              # 仅匹配文件时激活
```

**正文结构**：

1. **Overview** — 一句话说清这个 Skill 做什么
2. **Core principle** — 核心原则（如果有的话）
3. **When to Use** — 什么情况下用，什么情况下不用
4. **The Process** — 分步骤的详细流程
5. **Red Flags** — 如果出现这些信号，停下来
6. **Gotchas** — 常见易错点（随着使用积累）
7. **Integration** — 与其他 Skill、MCP、Hooks 的配合

### Step 5：用 TDD 方法测试 Skill

Superpowers 的 `writing-skills` Skill 提供了完整的 TDD-for-Skills 方法论（源码原文）。

**2026 补充**：官方 skill-creator Plugin 的 **Eval 模式**可自动化这一流程——生成 should-trigger / should-not-trigger 测试用例，测量触发命中率，并 A/B benchmark Skill vs 无 Skill 基线。两者可组合：TDD-for-Skills 定方法论，skill-creator Eval 定工具。

**核心原则**：如果你没有看到 Agent 在**没有** Skill 时失败，你就不知道 Skill 教的是对的东西。

**TDD 映射表**（来自 Superpowers `writing-skills/SKILL.md` 源码）：

| TDD 概念 | Skill 创建 |
|----------|-----------|
| **测试用例** | 用子 Agent 执行的压力场景 |
| **生产代码** | Skill 文档（SKILL.md） |
| **测试失败（RED）** | Agent 在没有 Skill 时违反规则（基线行为） |
| **测试通过（GREEN）** | Agent 在有 Skill 时遵循规则 |
| **重构** | 堵住漏洞的同时保持合规 |
| **先写测试** | 在写 Skill **之前**运行基线场景 |
| **亲眼看着失败** | 记录 Agent 违反规则时的精确借口 |
| **最小代码** | 写的 Skill 只针对那些具体的违规行为 |
| **亲眼看着通过** | 验证 Agent 现在遵循规则 |
| **重构循环** | 发现新的借口→堵住→重新验证 |

**实操步骤**：

1. 在**没有** Skill 的情况下，给 Claude 一个它会失败的场景（压力场景）
2. 记录 Claude 做错了什么，以及它是如何为自己辩护的（借口模式）
3. 写只针对这些具体违规行为的 Skill（不要过度设计）
4. 重新运行同样的场景——看 Claude 现在是否遵循规则
5. 如果 Claude 找到了新的违规方式（新的借口）→ 堵住它 → 重新验证

这种方法的精髓：Skill 不是一次性写成的完美文档，而是通过 RED-GREEN-REFACTOR 循环**逐步堵漏**的产物。首先让 Claude 在没有 Skill 时暴露错误（RED），记录精确的失败模式，写刚好够解决问题的 Skill（GREEN），然后反复发现新的漏洞→堵住→验证（REFACTOR）。

### Step 6：迭代

Anthropic 的建议：一个成熟 Skill 需要 3-5 次迭代。每次在真实使用中发现问题后，更新 Skill。最重要的是持续丰富 Gotchas 区——这是 Skill 质量的核心指标。

## 常见错误

下列错误来自社区反馈（码哥字节、Matt Pocock 等人的经验分享）：

| 错误 | 后果 | 修正 |
|------|------|------|
| description 写成"给人看的摘要" | 模型不知道什么时候该触发 | 包含触发条件和关键词 |
| 把所有内容塞进 SKILL.md | 上下文爆炸，每次触发都加载大量无关内容 | 拆分到 references/，按需加载 |
| 使用"铁轨式"指令（指定具体行号） | 代码稍有变动就失效 | 用原则性描述代替硬编码 |
| 重述 Claude 已经知道的事 | 增加上下文负担，零价值 | 只写 Claude 默认做不到的 |
| 没有 Gotchas 区 | Skill 反复在同一个坑里跌倒 | 每次踩坑就记下来 |
| 装太多 Skill | 描述预算被打爆，后半部分 Skill 全部失效 | 控制在 10-15 个 |
| 从不看 `/doctor` 或 `/skills` | 不知道哪些 Skill 被截断 | 定期检查；用 `/skills` 禁用低频 Skill |

## 项目级 vs 全局级的选择

| 场景 | 存放位置 | 原因 |
|------|----------|------|
| 团队共享的开发规范 | `.claude/skills/` | 通过 git 分发给所有团队成员 |
| 个人偏好的工作流 | `~/.claude/skills/` | 跨所有项目可用 |
| 试验中的 Skill | `~/.claude/skills/` | 先个人用，成熟后再推团队 |
| 与特定项目强绑定的 Skill | `.claude/skills/` | 切换到其他项目时不干扰 |

## 从现有 Skill 学习

学习编写 Skill 的最好方式是阅读高质量 Skill 的源码。按推荐阅读顺序：

1. **Karpathy 四原则**（最短，最精炼）— 只 4 条指令，但每条都精确命中了 LLM 的行为缺陷。适合理解"少即是多"。
2. **frontend-design**（设计类 Skill 范本）— 清晰定义了"做什么"和"绝对不要做什么"。适合学习"约束"怎么写。
3. **superpowers/test-driven-development**（流程类 Skill 范本）— 铁律 + 完整流程 + 15 种借口的反驳。适合学习"硬性约束"怎么写。
4. **superpowers/systematic-debugging**（方法论类 Skill 范本）— 4 阶段 + 安全阀机制。适合学习"防止模型走偏"。
5. **baoyu-infographic**（参数化 Skill 范本）— references/ 下多布局/风格文件。适合学习渐进式披露。
6. **skill-creator Eval 报告**（工具类范本）— 用数据验证 description 触发准确率。

---

> **本章资料来源**：[Anthropic 官方 PDF《The Complete Guide to Building Skills for Claude》](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)、[Anthropic 内部博客](https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills)、各 Skill 仓库源码
