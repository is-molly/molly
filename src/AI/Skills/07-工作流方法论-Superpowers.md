---
order: 7
date: 2026-06-27
---

# 工作流方法论 · Superpowers

> **章节类型：工作流方法论**（一整条 AI 驱动开发流水线，不是单个领域 Skill）。  
> 同类章节：[08 四套系统](08-工作流方法论-四套系统.md)（GSD / gstack / mattpocock / Karpathy）。  
> **组合安装 + 与 frontend-design 串联的编程工作流** → [03 方案 A](03-选型与组合方案.md#方案-a-工作流)。  
> 安装：Plugin → Claude Code `/plugin`（见 [02 管理架构](02-管理架构与快速上手.md)）。

## 项目概述

Superpowers 是目前 Claude Code 生态中 Stars 数最高（~239K，2026-06）的 Skill 项目。它不是单个 Skill，而是一个**完整的软件开发方法论**，被封装为 Claude Code 的 Plugin（在 Plugin 体系下打包了 Skills + Hooks + Agents + Commands）。

- **仓库**：[obra/superpowers](https://github.com/obra/superpowers)
- **作者**：Jesse Vincent（[obra](https://github.com/obra)，Prime Radiant 创始人，开源工单系统 Request Tracker 的作者）
- **Stars**：~239K（2026-06，持续增长）
- **协议**：MIT
- **首批入选** Anthropic 官方 Plugin Marketplace（2026 年 1 月）
- **支持平台**：Claude Code / Codex / Cursor / OpenCode / Gemini CLI / GitHub Copilot CLI / Antigravity / Factory Droid / Kimi Code / Pi 等

Jesse Vincent 在 2025 年 10 月的博客中描述了 Superpowers 的起源：他在用 Claude Code 开发 Request Tracker 时，反复遇到同一个模式——Claude 跳过了需求理解直接开始写代码、跳过测试、产出的代码语法正确但架构错误。问题不在 Claude 的能力，而在于缺少一个**专业开发流程**。Superpowers 就是这个流程，以 Skills 的形式编码。

## 设计哲学

Superpowers 有四条核心原则：

- **Test-Driven Development** — 永远先写测试
- **Systematic over ad-hoc** — 流程优于猜测
- **Complexity reduction** — 简单是首要目标
- **Evidence over claims** — 验证后再确认成功

这四条原则贯穿 Superpowers 的每一个 Skill。它不是建议——是硬性约束。

> "The goal is workflows clear enough for an enthusiastic junior engineer with poor taste, no judgement, no project context, and an aversion to testing to follow."
> —— Jesse Vincent, Superpowers README

这句话解释了 Superpowers 的设计标准：连一个"没有品味、没有判断力、不了解项目、讨厌写测试但充满热情的初级工程师"都能照着执行。正是因为这种极端约束，Superpowers 的指令才能被 Claude 可靠地执行。

## 架构设计

### Plugin 结构

Superpowers 的完整目录结构：

```
superpowers/
├── .claude-plugin/          # Claude Code Plugin 清单（注册到 Marketplace）
├── .codex/                  # Codex 安装说明
├── .cursor-plugin/          # Cursor Plugin 清单
├── .opencode/               # OpenCode 安装说明
├── skills/                  # 15 个核心 Skill 的 SKILL.md
│   ├── brainstorming/
│   ├── using-git-worktrees/
│   ├── writing-plans/
│   ├── subagent-driven-development/
│   ├── executing-plans/
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   ├── verification-before-completion/
│   ├── dispatching-parallel-agents/
│   ├── requesting-code-review/
│   ├── receiving-code-review/
│   ├── using-superpowers/
│   ├── writing-skills/
│   ├── finishing-a-development-branch/
│   └── getting-started/
├── hooks/                   # Hook 脚本（会话启动注入）
├── agents/                  # 子 Agent 定义（实现器、审查器）
├── commands/                # Slash Commands（已弃用，迁移到了 Skills）
├── scripts/                 # 辅助脚本
├── tests/                   # 自测
└── docs/                    # 文档
```

### Bootstrap 机制：1% 规则

Superpowers 的核心机制不是通过用户手动调用，而是通过 `session-start` Hook 自动注入。当 Claude Code 启动时，Hook 注入类似这样的引导指令：

```
<EXTREMELY_IMPORTANT>You have Superpowers. RIGHT NOW, go read:
@~/.claude/plugins/cache/Superpowers/skills/getting-started/SKILL.md
</EXTREMELY_IMPORTANT>
```

Claude 读取 `getting-started` Skill 后，理解了自己拥有 Superpowers 技能库。从那一刻起，Claude 在每个操作前都会检查"有没有 Superpowers 的 Skill 适用于当前情况"。这被称为 **1% 规则**：

> 如果至少有 1% 的几率某个 Skill 可能适用，必须调用它。

这意味着 Superpowers 的 Skills 不是被动等待召唤——它们是主动介入的。Claude 会被强制进入 Superpowers 的方法论流程。

### Skills 分类

Superpowers 的 15 个 Skills 分成四组：

**测试组**（Testing）
| Skill | 职责 |
|-------|------|
| `test-driven-development` | RED-GREEN-REFACTOR 循环，附 testing-anti-patterns 参考文件 |

**调试组**（Debugging）
| Skill | 职责 |
|-------|------|
| `systematic-debugging` | 四阶段根因分析 |
| `verification-before-completion` | 在声称"修好了"之前验证真的修好了 |

**协作组**（Collaboration）——这是 Superpowers 的核心，包含完整开发工作流的每一步
| Skill | 职责 |
|-------|------|
| `brainstorming` | 苏格拉底式需求引导 |
| `writing-plans` | 编写详细实施计划 |
| `using-git-worktrees` | 创建隔离 feature 分支 |
| `subagent-driven-development` | 主执行模式：每任务派一个新 Agent → 两阶段审查 |
| `executing-plans` | 备选执行模式：批次执行 + 人类检查点 |
| `dispatching-parallel-agents` | 并行子 Agent 调度 |
| `requesting-code-review` | 代码审查发起 |
| `receiving-code-review` | 审查反馈响应 |
| `finishing-a-development-branch` | 完成后合并/PR/清理 |

**元 Skills**（Meta）
| Skill | 职责 |
|-------|------|
| `writing-skills` | 遵循 Superpowers 规范编写新 Skill |
| `using-superpowers` | 介绍 Skills 体系自身 |
| `getting-started` | 首次安装的引导 Skill |

## 逐个 Skill 详细教学

以下基于 Superpowers 实际的 SKILL.md 源码编写。每个 Skill 的指令逻辑、检查点、决策门都来自源码原文。

### brainstorming — 编码前的强制性思考

**自动触发条件**：任何涉及"创建功能、构建组件、添加功能、修改行为"的操作。

**核心流程**：

1. **理解想法** — 先检查当前项目状态（文件、文档、最近提交），然后**一次一个问题**引导你细化需求。优先使用选择题（减少你的回答负担），每次只能问一个问题。
2. **探索方案** — 提出 2-3 种不同方案，对话式地给出每个方案的权衡，最后给出推荐方案和理由。
3. **呈现设计** — 将设计方案分成 200-300 字的小块呈现，每块之后停顿询问"这部分看起来对吗？"。覆盖：架构、组件、数据流、错误处理、测试策略。
4. **文档化** — 将验证过的设计写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`，提交到 git。

**关键原则**（来自源码原文）：

- 一次只问一个问题——不要用多问题轰炸
- 优先选择题——比开放式问题更容易回答
- 无情地应用 YAGNI——从所有设计中删除不必要的功能
- 在确定方向前始终提出 2-3 种替代方案
- 增量验证——分段呈现设计，每段验证
- 灵活应变——如果某些内容说不通，随时回头澄清

**工作流衔接**：

brainstorming 完成后，Skill 会问："Ready to set up for implementation?" 然后自动引导进入 `using-git-worktrees` 创建隔离工作区 → `writing-plans` 编写详细计划。

### test-driven-development — 铁律级 TDD 执行

**自动触发条件**：实现任何功能或修复 Bug 之前。

**核心指令**（源码原文）：

> "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST"
> 没有先写失败测试，就没有生产代码。

如果在测试之前写了代码？删掉。从零开始。不看它、不"参考"它、不"适配"它——删掉就是删掉。

**RED-GREEN-REFACTOR 完整流程**：

**第一步 — RED：写失败测试**

写一个最小化测试，展示应该发生什么。

要求：
- 一个行为
- 清晰的命名
- 真实代码（除非无法避免才 mock）

```
✅ 好的测试：
test('retries failed operations 3 times', async () => { ... })
测试了真实行为

❌ 坏的测试：
test('retry works', async () => { ... })
名字模糊，测试 mock 而不是真实代码
```

**第二步 — Verify RED：亲眼看着它失败**

这是强制步骤，永远不能跳过。

```bash
npm test path/to/test.test.ts
```

确认：
- 测试失败（不是报错）
- 失败信息符合预期
- 失败原因是因为功能还没实现（不是笔误）

**如果测试通过了？** 说明在测试一个已经存在的功能——修正测试。
**如果测试报错了？** 修正错误，重新运行直到正确地失败。

**第三步 — GREEN：最小代码**

写刚好能让测试通过的最简单代码。不添加功能、不重构其他代码、不超过测试范围。

```
✅ 好的实现：
刚好让测试通过，没有多余

❌ 坏的实现：
加了一堆 options 参数——YAGNI（You Aren't Gonna Need It）
```

**第四步 — Verify GREEN：亲眼看着它通过**

```bash
npm test path/to/test.test.ts
```

确认：
- 测试通过
- 其他测试仍然通过
- 输出干净（无错误、无警告）

**第五步 — REFACTOR：清理**

只在变绿之后重构：
- 消除重复
- 改进命名
- 提取辅助函数

保持测试绿色。不添加新行为。

**15 种常见借口的反驳**（全部来自源码原文）

| 借口 | 反驳 |
|------|------|
| "太简单了不需要测试" | 简单代码也会坏。测试只花 30 秒。 |
| "我之后再加测试" | 之后加的测试一开始就通过。立即通过说明不了任何问题。 |
| "代码之后的测试能达到同样目的" | 不。之后加的测试回答"代码做了什么"。先写的测试回答"应该做什么"。 |
| "已经手动测试过了" | 临时操作 ≠ 系统化。没有记录，不能重跑。 |
| "删掉已经花了 X 小时的工作很浪费" | 沉没成本谬误。保留你无法信任的代码才是真正的浪费。 |
| "留着当参考，先写测试" | 你会"适配"它——那就是之后加测试。删掉就是删掉。 |
| "需要先探索一下" | 可以。扔掉探索代码，用 TDD 重新开始。 |
| "TDD 会拖慢我" | TDD 比调试快。务实 = 先测试。 |
| "手动测试更快" | 手动不能证明边界情况。每次代码变动你都要重新手动测试。 |
| "现有代码没有测试" | 你正在改进它——先给现有代码加测试。 |

**TDD 铁律的最终表述**：

```
生产代码 → 测试存在并且先失败了
否则 → 不是 TDD
```

### systematic-debugging — 四阶段根因分析

**自动触发条件**：遇到任何 Bug、测试失败、意外行为。

**核心指令**：

> "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"
> 完成 Phase 1 之前，不能提出任何修复方案。

**Phase 1：根因调查**（在尝试任何修复之前）

1. **仔细读错误信息** — 不要跳过错误或警告。它们通常包含了精确的解决方案。完整读取堆栈跟踪。
2. **稳定复现** — 能可靠触发吗？精确步骤是什么？每次都发生吗？如果不可复现 → 收集更多数据，不要猜测。
3. **检查最近的改动** — 什么改动可能导致这个问题？Git diff、最近提交、新依赖、配置变化、环境差异。
4. **在多组件系统中收集证据** — 在每个组件边界处添加诊断插桩，追踪数据从哪一层开始出错。
5. **追踪数据流** — 错误值从哪里来？谁用它调用了这里？一直向上追踪直到找到源头。**在源头修复，不要治标。**

**Phase 2：模式分析**

1. 找到相似的正确代码
2. 完整读取参考实现（不要略读）
3. 列出正常和出错之间的所有差异
4. 理解依赖关系

**Phase 3：假说与测试**

1. 形成单一假说："我认为 X 是根因，因为 Y"
2. 做最小改动来测试假说
3. 一次只测一个变量
4. 如果不知道，直接说"我不理解 X"——不要假装理解

**Phase 4：实施修复**

1. 创建复现 Bug 的失败测试
2. 实施单一修复
3. 验证修复
4. **如果尝试了 3 次以上修复都失败了 → 停止并质疑架构本身**

**关键安全阀**：如果尝试了 3 次以上修复全部失败，这可能是架构层面的问题，而不是实现 Bug。Superpowers 会阻止你继续试第 4 次，强制你停下来质疑"是不是从根本上走错了方向"。

### subagent-driven-development — 多 Agent 驱动开发

**自动触发条件**：当实施计划准备就绪，且任务是独立的。

这是 Superpowers 的**默认执行模式**（备选是 `executing-plans`，用于需要人类检查点的场景）。

**核心机制**：

```
对每个任务：
  1. 派发一个全新的实现子 Agent
  2. 子 Agent 问问题 → 你回答 → 子 Agent 实现 + 自测 + 提交 + 自查
  3. 派发 spec 合规审查子 Agent → 确认代码符合设计文档
    如果不通过 → 实现 Agent 修复 → 重新审查
  4. 派发代码质量审查子 Agent → 审查代码质量
    如果不通过 → 实现 Agent 修复 → 重新审查
  5. 标记任务完成
  6. 下一个任务
```

**关键设计决策**：

- **每个任务一个全新的子 Agent**——不会上下文污染，每个 Agent 只专注于一个任务
- **两阶段审查**——先检查是否符合设计文档（没有多做事也没有少做事），再检查代码质量
- **审查必须通过才能进入下一个任务**——不能跳过
- **控制器预提取所有任务内容**——子 Agent 不需要读计划文件，控制器直接提供完整上下文

**效率提升**：
- 子 Agent 天然遵循 TDD
- 每个任务都是全新的上下文（不会混乱）
- 并行安全（子 Agent 之间不会互相干扰）
- 子 Agent 可以在工作中提问

**绝对禁止的行为**：
- 在主分支上开始实现
- 跳过任何审查步骤
- 在有问题未修复的情况下继续
- 并行派发多个实现子 Agent（会冲突）
- 让子 Agent 自己去读计划文件（控制器必须提供完整文本）
- spec 审查没通过就进行代码质量审查（顺序错误）

### writing-plans — 为"零上下文工程师"编写实施计划

**触发条件**：设计方案已经批准，需要详细的实施计划。

**核心指令**：

> "编写全面的实施计划，假设工程师对代码库零上下文，品味堪忧。记录他们需要知道的一切。"

**任务粒度标准**：每个步骤是"一个动作（2-5 分钟）"：
- "写失败测试" — 一个步骤
- "运行以确认测试失败" — 一个步骤
- "实现最小代码通过测试" — 一个步骤
- "运行测试确认通过" — 一个步骤
- "提交" — 一个步骤

**计划文档格式**：

每个计划必须以固定的 Header 开头：

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans
> or superpowers:subagent-driven-development to implement this plan

**Goal:** [一句话描述目标]
**Architecture:** [2-3 句话描述方案]
**Tech Stack:** [关键技术和库]
```

每个任务的结构：

```markdown
### Task N: [组件名]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the failing test**
[完整测试代码]

**Step 2: Run test to verify it fails**
Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**
[完整实现代码]

**Step 4: Run test to verify it passes**
Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```

**关键要求**：
- 始终使用精确文件路径
- 计划中包含完整代码（而不是"添加验证"这种模糊描述）
- 精确的命令和预期输出

**执行交接**：计划完成后，Skill 给出两个选项：
1. **Subagent-Driven（当前会话）** — 派发子 Agent 执行，每任务两阶段审查
2. **Parallel Session（新会话）** — 在新会话中用 `executing-plans` 批次执行

### 其他重要 Skill

**using-git-worktrees** — 为每个 Feature 创建隔离的 Git Worktree 分支。运行项目初始化、验证测试基线。这保证了主分支始终干净。

**executing-plans** — 备选执行模式。与 SDD 的区别是：在主会话中批次执行，每批后有**人类检查点**。适合需要密切监督的任务。

**dispatching-parallel-agents** — 当多个任务完全不互相依赖时，并行派发子 Agent。用于独立任务（如同时进行前端和后端变更）。

**requesting-code-review** — 在任务之间触发。按照计划的严重性等级报告问题。Critical 问题阻塞进度。

**receiving-code-review** — 指导 Agent 如何响应审查反馈。

**finishing-a-development-branch** — 所有任务完成后：验证测试、提出选项（合并/PR/保留/丢弃）、清理 Worktree。

**verification-before-completion** — 在声称"修好了"之前，确认真的修好了。这是 `systematic-debugging` 的配套 Skill。

**writing-skills** — 如何按照 Superpowers 规范创建新 Skill。包含测试方法论。

## 完整工作流串联

Superpowers 最强大的地方是 Skill 之间的自动串联。从模糊想法到合并分支的全流程：

```
你: "帮我做一个用户认证功能"
    ↓
[brainstorming 自动触发]
Claude: "这个功能是给什么类型的用户使用的？
         A) 企业内部员工  B) 外部客户  C) 两者都有"
你: "B"
Claude: "需要支持哪些登录方式？
         A) 邮箱+密码  B) 手机验证码  C) OAuth  D) 全部"
...（苏格拉底式对话，一次一个问题）
Claude: "以下是设计方案，第一部分：架构"
    ↓ (你确认)
[设计文档写入 docs/plans/2026-06-27-auth-design.md]
    ↓
Claude: "Ready to set up for implementation?"
    ↓
[using-git-worktrees 自动触发]
→ 创建 `feature/user-auth` 分支
→ 隔离 Worktree
→ 运行测试确认基线
    ↓
[writing-plans 自动触发]
→ 将设计文档拆分为 2-5 分钟粒度的任务
→ 每个任务含完整代码、测试、验证步骤
→ 写入 docs/plans/2026-06-27-auth-plan.md
    ↓
你: "go"
    ↓
[subagent-driven-development 自动触发]
Task 1: 创建 User 模型
  → 派发实现 Agent
  → 实现 Agent 完成后
  → 派发 spec 审查 Agent → ✅ 符合设计
  → 派发代码质量审查 Agent → ✅ 通过
  → 标记完成

Task 2: 实现注册 API
  → 派发实现 Agent
  → ...（同样流程）
  ...（自动执行所有任务）
    ↓
[finishing-a-development-branch 自动触发]
→ 所有测试通过
→ 选项：merge / PR / keep / discard
→ 清理 Worktree
```

从"帮我做用户认证"到可合并的代码，你只需要回答 brainstorming 的问题确认设计方向，然后说"go"。这就是 Superpowers 的核心价值。

## 安装

> 通用规则（官方 vs 作者 Market、装完怎么验）→ [05 · 多路径安装](05-安装方式参考手册.md#多路径安装怎么选装完怎么验)

### Claude Code — 推荐（官方 Market）

**用这个**：Anthropic 官方收录，一条命令，与 [obra/superpowers README](https://github.com/obra/superpowers) 的 “Official Marketplace” 一致。

```
/plugin install superpowers@claude-plugins-official
/reload-plugins
/skills
/doctor
```

**怎么算装好**：新开或当前会话里说 `help me plan a feature`。Superpowers 生效时 Claude **不会直接写代码**，而是先走 brainstorming 问需求；若直接开写，说明 Plugin / hooks 未加载 → 再 `/reload-plugins` 或重启会话。

### Claude Code — 备选（作者 Market）

**何时用**：`/plugin install superpowers@claude-plugins-official` 报找不到包；或你已在用 `obra/superpowers-marketplace` 里的其他 Plugin，想统一从一个 Market 更新。

**与官方 Market 的区别**：装的是**同一 Superpowers**，只是 catalog 来源不同（作者维护的 [superpowers-marketplace](https://github.com/obra/superpowers-marketplace)）。**不要两个 Market 各装一遍。**

```
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
/reload-plugins
/skills
/doctor
```

### 其他 Agent（与 Claude Code 安装互不同步）

每个工具**单独装**；Claude Code 里装好不会自动出现在 Codex / Cursor。

| 平台 | 推荐命令 |
|------|----------|
| Cursor | `/add-plugin superpowers` 或 Marketplace 搜 superpowers |
| Codex CLI | `/plugins` → 搜 `superpowers` → Install |
| GitHub Copilot CLI | `copilot plugin marketplace add obra/superpowers-marketplace` → `copilot plugin install superpowers@superpowers-marketplace` |
| Gemini CLI | `gemini extensions install https://github.com/obra/superpowers` |
| OpenCode | 按仓库 `.opencode/INSTALL.md` |

CC Switch **不能**代装 Superpowers Plugin。

## 更新

```
/plugin update superpowers
```

---

> **本章资料来源**：[obra/superpowers GitHub 仓库](https://github.com/obra/superpowers)，所有 Skill 的教学内容均基于实际 SKILL.md 源码
