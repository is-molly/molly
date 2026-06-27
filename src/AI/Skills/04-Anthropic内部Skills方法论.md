---
order: 4
date: 2026-06-27
---

# Anthropic 内部 Skills 方法论

2026 年 6 月 3 日，Anthropic 的 Claude Code 团队在官方博客上发表了《[Lessons from building Claude Code: How we use skills](https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills)》，首次系统性地公开了他们在内部使用 Skills 的经验。这篇文章披露了两个核心内容：**9 类 Skills 分类体系**和**9 条编写 Skill 的工程技巧**。

Anthropic 内部有数百个 Skills 在活跃使用中。经过系统梳理后发现，这些 Skills 自然聚类成了 9 个类型。最好的 Skill 干净地属于其中一类；试图兼顾多个类型的 Skill 会让 Agent 困惑。

> "After cataloging all of our internal skills at Anthropic, we noticed they cluster into nine categories. The best skills fit cleanly into one; the ones that try to do too much straddle several and confuse the agent."

本章内容基于这篇博客的完整原文。

## 4.1 Anthropic 的 9 类 Skills 分类体系

### 类型 1：Library & API Reference（库与 API 参考）

**定位**：教 Claude 如何正确使用某个库、CLI 或 SDK 的 Skill。

这些 Skill 面向的可以是内部私有库，也可以是 Claude 偶尔会出错的常用公开库。它们通常包含一个参考代码片段目录和一份"容易被踩到的坑"（gotchas）列表。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `billing-lib` | 内部账单库：边界情况、易错点等 |
| `internal-platform-cli` | 内部 CLI 包装器的每个子命令及使用场景示例 |
| `sandbox-proxy` | 配置组织的出口网关：哪些主机可达、如何调试"connection refused"、如何添加白名单 |

如果你团队有自己的内部 SDK、私有 npm 包或 CLI，为它们写一个 Library Reference Skill 回报很高。Claude 对通用库的知识通常足够，但对你内部的东西什么都不知道。

### 类型 2：Product Verification（产品验证）

**定位**：描述如何测试或验证代码是否正常工作的 Skill。

这类 Skill 常与 Playwright、tmux 等外部工具配合，用于端到端测试。Anthropic 特别指出：**验证类 Skill 对 Claude 输出质量的提升是所有类型中最为可衡量的**——投入一个工程师一周时间专门打磨验证 Skill 是完全值得的。如果你只能先做一个 Skill，Anthropic 的建议是：**从 Product Verification 开始，而不是从代码生成开始**。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `signup-flow-driver` | 在无头浏览器中走完注册→邮箱验证→引导流程，每个步骤都有状态断言 Hook |
| `checkout-verifier` | 用 Stripe 测试卡驱动结账 UI，验证发票实际落到了正确的状态 |
| `tmux-cli-driver` | 用于需要 TTY 的交互式 CLI 测试 |

具体做法：让 Claude 录制验证过程视频（这样你能看到它实际测试了什么）、在每个步骤强制执行程序化断言——这些通常通过在 Skill 中包含脚本来实现。

如果你的产品有注册、支付、关键用户路径，为它们写验证 Skill 能显著减少上线前的回归 Bug。Anthropic 的数据表明验证类 Skill 对输出质量的提升是所有类型里最可衡量的。

### 类型 3：Data Fetching & Analysis（数据获取与分析）

**定位**：连接到你的数据和监控栈的 Skill。

这类 Skill 可能包含获取数据的库（含凭证）、特定的 Dashboard ID、常用工作流指令或数据获取方法。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `funnel-query` | "join 哪些事件可以看到注册→激活→付费的漏斗" + 包含规范 user_id 的表 |
| `cohort-compare` | 比较两个群组的留存率或转化率，标记统计显著的差异，链接到群组定义 |
| `grafana` | 数据源 UID、集群名、问题→Dashboard 查询表 |
| `datadog` | 字段参考（`@request_id` vs `trace_id`）、服务列表、指标前缀规范 |

如果经常让 Claude 帮忙查数据、跑分析，把常用的查询模式、表结构、指标含义固化到 Skill，比每次对话解释高效得多。

### 类型 4：Business Process & Team Automation（业务流程与团队自动化）

**定位**：将重复性工作流自动化成一个命令的 Skill。

这类 Skill 通常是相对简单的指令，但可能依赖其他 Skill 或 MCP 连接。Anthropic 建议在这类 Skill 中用日志文件保存历史结果，帮助模型保持一致，并回顾之前的执行情况。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `standup-post` | 聚合你的 Ticket Tracker、GitHub 活动和之前的 Slack → 格式化站会报告，只显示增量变化 |
| `create-<ticket-system>-ticket` | 强制 Schema 校验（有效枚举值、必填字段）+ 创建后工作流（通知 reviewer、在 Slack 中链接） |
| `weekly-recap` | 已合并的 PR + 已关闭的 Ticket + 部署 → 格式化的周报 |

### 类型 5：Code Scaffolding & Templates（代码脚手架与模板）

**定位**：为代码库中特定功能生成框架模板的 Skill。

你可以将这些 Skill 与可组合的脚本搭配使用。当脚手架需要自然语言需求描述（不能完全用代码覆盖）时，这类 Skill 尤其有用。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `new-<framework>-workflow` | 生成新的 service/workflow/handler 并附带你的注解 |
| `new-migration` | 迁移文件模板 + 常见踩坑点 |
| `create-app` | 新内部应用，预配好你的认证、日志和部署配置 |

每当你需要重复做一些标准化的代码生成操作——比如 `create-react-component`、`new-api-route`、`add-database-migration`——都值得写一个 Scaffolding Skill。它保证生成的代码遵循团队规范，而不是 Claude 的默认写法。

### 类型 6：Code Quality & Review（代码质量与审查）

**定位**：在组织内强制执行代码质量标准和辅助代码审查的 Skill。

这类 Skill 可以包含确定性脚本或工具，以获得最大可靠性。可以配合 Hooks 自动运行，也可以集成到 GitHub Action 中。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `adversarial-review` | 启动一个全新视角的子 Agent 来挑剔代码→实现修复→迭代直到发现的问题退化为细枝末节 |
| `code-style` | 强制执行代码风格，特别是 Claude 默认做不到位的那些风格 |
| `testing-practices` | 怎么写测试、测试什么的指令 |

实际用法：`testing-practices` 定义"好的测试长什么样"→ 代码生成时遵循 → `adversarial-review` 自动挑剔→ 循环修复。这是 Anthropic 内部最常见的质量保障流水线。

### 类型 7：CI/CD & Deployment（CI/CD 与部署）

**定位**：帮助获取、推送和部署代码的 Skill。可能引用其他 Skill 来收集数据。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `babysit-pr` | 监控 PR → 重试不稳定的 CI → 解决合并冲突 → 启用自动合并 |
| `deploy-<service>` | 构建 → 冒烟测试 → 渐进式流量上线（带错误率对比）→ 回归时自动回滚 |
| `cherry-pick-prod` | 隔离 worktree → cherry-pick → 冲突解决 → 带模板的 PR |

把"盯 PR、等 CI、手动 rerun、解决冲突"这条所有工程师都讨厌的流程自动化了。

### 类型 8：Runbooks（运维手册）

**定位**：接收一个症状（如 Slack 线程、告警、错误签名），走一遍多工具排查流程，产出结构化报告的 Skill。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `<service>-debugging` | 为你流量最高的服务映射：症状→工具→查询模式 |
| `oncall-runner` | 获取告警→检查常见问题→格式化发现 |
| `log-correlator` | 给定一个请求 ID，从可能接触过它的每个系统中拉取对应日志 |

### 类型 9：Infrastructure Operations（基础设施运维）

**定位**：执行常规维护和运维操作的 Skill。其中一些涉及破坏性操作，因此受益于护栏机制。

Anthropic 内部的真实案例：

| Skill 名称 | 用途 |
|-----------|------|
| `<resource>-orphans` | 查找孤儿 Pod/Volume → 发到 Slack → 冷却期 → 用户确认 → 级联清理 |
| `dependency-management` | 组织级的依赖审批工作流 |
| `cost-investigation` | "为什么我们的存储/出口账单飙升" + 具体的 bucket 和查询模式 |

这类 Skill 中有部分涉及破坏性操作（如删除资源），应该设置 `disable-model-invocation: true` 限制为仅手动调用，并使用 `allowed-tools` 限制权限范围。

### 分类体系地图

将 9 类按编程开发相关度排列：

| 类型 | 编程开发相关度 | 典型用户 |
|------|---------------|----------|
| Library & API Reference | ⭐⭐⭐⭐⭐ | 所有开发者 |
| Code Quality & Review | ⭐⭐⭐⭐⭐ | 所有开发者 |
| Code Scaffolding & Templates | ⭐⭐⭐⭐⭐ | 所有开发者 |
| Product Verification | ⭐⭐⭐⭐ | 全栈/前端开发者 |
| CI/CD & Deployment | ⭐⭐⭐⭐ | DevOps/SRE |
| Runbooks | ⭐⭐⭐ | SRE/On-call |
| Infrastructure Operations | ⭐⭐⭐ | SRE/Platform |
| Data Fetching & Analysis | ⭐⭐ | 数据工程师 |
| Business Process Automation | ⭐⭐ | TPM/EM |

## 4.2 Anthropic 的 9 条 Skill 编写工程技巧

以下是 Anthropic Claude Code 团队在内部编写数百个 Skill 后总结的最佳实践。

### 技巧 1：不要重述显而易见的事

Claude 已经会写代码，能读你的代码库。一个只是重述 Claude 默认行为的 Skill 只会增加上下文负担。发布知识型 Skill 时，专注于那些把 Claude 推出默认思维模式的信息。

[frontend-design skill](https://github.com/anthropics/skills/blob/main/skills/frontend-design/SKILL.md) 就是个好例子——它不是教 Claude 怎么写 HTML/CSS，而是教它避免 Inter 字体和紫色渐变这些"AI 通用审美"。这是 Claude 的默认行为做不到的。

### 技巧 2：构建 Gotchas 区

任何 Skill 中信号密度最高的内容是 Gotchas（易错点）区。这个区域应该通过收集 Claude 在使用你的 Skill 时反复遇到的失败点来逐步积累。

Anthropic 官方给出的 Gotchas 示例：

> - "`subscriptions` 表是 append-only 的。你要的是拥有最高 `version` 的那一行，而不是最近 `created_at` 的那一行。"
> - "这个字段在 API 网关中叫 `@request_id`，在账单服务中叫 `trace_id`。它们是同一个值。"
> - "即使 Stripe webhook 实际没有处理，预发布环境也返回 200。检查 `payment_events` 才能知道真实状态。"

重要的是：Gotchas 区不是一次性写成的。每次 Claude 在使用 Skill 时犯了错误，就把这个错误写进 Gotchas。日积月累，这是 Skill 最有价值的部分。

### 技巧 3：利用文件系统做渐进式披露

Skill 是一个文件夹，不只是 Markdown 文件。整个文件系统就是一种上下文工程和渐进式披露的形式。

做法很简单：

- 把详细的函数签名和使用示例拆分到 `references/api.md`
- 当最终输出是 Markdown 文件时，在 `assets/` 中放入模板文件供复制使用
- 可以有 `references/`、`scripts/`、`examples/` 等目录，帮助 Claude 更高效地工作

根本原则：告诉 Claude Skill 中有哪些文件，它会在合适的时机自行读取，不需要你把所有内容都塞进 SKILL.md。

### 技巧 4：避免铁轨式指令

Claude 一般会尽量遵守你给出的指令。正因为 Skill 的复用性很高，在指令中要格外小心**不要过于具体**。给 Claude 需要的信息，但保留足够灵活性让它适应实际情况。

**反例**：

```markdown
Step 1: Open src/index.ts
Step 2: Add import statement on line 5
Step 3: Modify function on line 42-67
```

这种"铁轨式"指令在代码结构稍有变化时就完全失效。更好的写法：

```markdown
Understand the current codebase structure first.
Then recommend the most appropriate place to add the new functionality.
Present your recommendation before making changes.
```

### 技巧 5：Description 写给模型看，不是写给人看

这是最容易被误解的一条。当 Claude Code 启动会话时，它构建所有可用 Skill 的 name + description 列表。Claude 扫描这个列表来决定"有没有 Skill 适合这个请求？"因此 description 不是一个供人阅读的摘要，而是一个**触发条件描述**。

好的 description 同时说明：

- Skill 做什么
- 什么场景下应该触发
- 相关的触发关键词

比如：

```yaml
description: Babysit your PR by monitoring CI, retrying flaky tests, and resolving merge conflicts. Use when you've opened a PR and want to babysit it until merge
```

关键词 "babysit" 被包含在 description 中，这意味着即使用户只说 "babysit my PR"，Claude 也能匹配到这个 Skill。

### 技巧 6：用文件做 Skill 记忆

某些 Skill 可以通过在自身内部存储数据来实现一种形式的记忆。可以简单到 append-only 文本日志文件或 JSON 文件，复杂到 SQLite 数据库。

比如一个 `standup-post` Skill 维护一个 `standups.log`，记录它写过的每一条站会报告。下次运行时，Claude 读取历史记录，就能知道从昨天到现在发生了什么变化。

**持久化路径**：Plugin 内的 Skill 可使用环境变量 `${CLAUDE_PLUGIN_DATA}` 获取稳定目录。个人 Skill 更常用 `${CLAUDE_SKILL_DIR}` 引用同目录下的脚本和模板。

### 技巧 7：存储可执行脚本和可生成代码

能交给 Claude 的最强大工具之一就是代码。给 Claude 提供脚本和库，让它把精力花在**编排组合**上——决定接下来做什么——而不是重复写样板代码。

比如在 `data-science` Skill 中，有一个函数库用于从事件源获取数据。Claude 可以在对话中动态生成脚本来组合这些功能，完成更高级的分析。对于像"上周二发生了什么？"这样的提示词特别有效。

### 技巧 8：使用按需 Hook

Skills 可以包含 Hook，这些 Hook 只在 Skill 被调用时激活，且只持续到会话结束。适合那些不想一直运行的、看法更强硬的 Hook，但在特定场景下有用。

比如（社区/gstack 实践，非 Anthropic 博客原文）：

- `/careful` — 拦截 `rm -rf`、`DROP TABLE`、`force push` 等危险命令
- 代码格式化 Hook — 只在代码生成 Skill 激活时运行

### 技巧 9：思考 setup 流程

某些 Skill 需要用户提供配置才能正确运行。一个好的模式是将配置信息存储在 Skill 目录的 `config.json` 中。如果配置尚未设置，Agent 可以向用户询问信息。

如果你需要 Agent 提供结构化的选择题，可以让 Claude 使用 `AskUserQuestion` 工具（Claude Code 专有，其他平台可能不可用）。

## 4.3 融会贯通：一个高质量 Skill 的标准

综合以上 9 类分类和 9 条技巧，一个高质量的 Skill 应该具备以下特征：

1. **清晰归属**：干净地落入一个分类，而不是试图兼顾多个
2. **好的 description**：让模型知道什么时候应该触发它
3. **Gotchas 区**：随着使用积累不断完善的易错点列表
4. **渐进式披露**：利用 references/、scripts/、assets/ 组织内容，而不是全部塞进 SKILL.md
5. **灵活性**：给 Claude 足够上下文，但不过度约束
6. **持久化**：如果需要"记住"之前的状态，使用文件来实现

> **2026 更新**：Anthropic 已将 skill-creator 升级为官方 Plugin，支持 Create / Eval / Improve / Benchmark 四模式。详见 [06 工具链 · skill-creator](06-工具链-元Skill与安装器.md)。

---

> **本章资料来源**：[Anthropic 官方博客《Lessons from building Claude Code: How we use skills》（2026-06-03）](https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills)、[Anthropic 官方 PDF《The Complete Guide to Building Skills for Claude》](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
