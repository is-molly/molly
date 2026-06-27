---
order: 0
date: 2026-06-27
---

# Skills

> **Claude Code + CC Switch 用户**：从 [02 管理架构与快速上手](02-管理架构与快速上手.md) 读起——先搞清 **谁管 Skill、CC Switch 能管到哪**，再安装。

## 章节分类（统一维度）

全书教程按 **类型** 划分，不是按「是不是明星 repo」混排：

| 类型 | 章节                                                                        | 说明 |
|------|---------------------------------------------------------------------------|------|
| **上手** | [02](02-管理架构与快速上手.md) [03](03-选型与组合方案.md) [13](13-治理-调优与冲突.md)            | 管理架构、**选型+安装+编程工作流**、治理 |
| **原理** | [01](01-基础与生态概览.md) [04](04-Anthropic内部Skills方法论.md) [05](05-安装方式参考手册.md) | 是什么、怎么写、安装参数 |
| **工具链** | [06](06-工具链-元Skill与安装器.md)                                                | 装/造 Skill 的工具（skill-creator、npx skills） |
| **工作流方法论** | [07](07-工作流方法论-Superpowers.md) [08](08-工作流方法论-四套系统.md)                    | **一整条开发流水线**（多 Skill 协作） |
| **领域 Skill** | [09](09-领域Skill-前端与UI.md) [10](10-领域Skill-内容创作.md)                        | **垂直场景**能力包 |
| **生态索引** | [11](11-生态索引.md)                                                          | 仓库查表 |
| **自建** | [12](12-自建Skill-开发指南.md)                                                  | 写 SKILL.md |

> **07 vs 08 vs 09**：07/08 是「怎么组织一整次开发」；09/10 是「某一类任务用什么 Skill 包」。Superpowers 单独成章只因篇幅，**类型上仍属工作流方法论**。

---

## CC Switch 用户：谁管 Skill？

| 做什么 | 用谁 |
|--------|------|
| 装 **Plugin**（Superpowers、skill-creator 等含 hooks 的套装） | **Claude Code** `/plugin`（CC Switch **不能**代劳） |
| 装 **单体 Skill**、双通道仓库的 npx 路径（如 baoyu-skills） | **CC Switch** `install` + `sync`（推荐）；或 `npx skills add`（无 CC Switch 时）；**不要混装** |
| 会话里开关、看 budget | **Claude Code** `/skills`、`/doctor` |
| 换 API Provider | CC Switch 切换 + **`skills sync`** |

详见 [02 第二章](02-管理架构与快速上手.md)。

---

## 阅读路径

### A · 马上用（推荐）

`02 管理架构 → 03 选型（含各方案编程工作流）→ （Skill 多了）13 治理`

### B · 先理解再动手

`01 基础 → 04 方法论 → 02 → 03`

### C · 自己写 Skill

`01 → 12 自建 → 06 工具链`

### D · 深挖某一类

| 目标 | 章节 |
|------|------|
| Superpowers 流水线 | [07 工作流·Superpowers](07-工作流方法论-Superpowers.md) |
| GSD / gstack / Matt / Karpathy | [08 工作流·四套](08-工作流方法论-四套系统.md) |
| 前端 UI | [09 领域·前端](09-领域Skill-前端与UI.md) |
| 内容创作 | [10 领域·内容](10-领域Skill-内容创作.md) |
| 查仓库 | [11 索引](11-生态索引.md) |

---

> 数据基于 GitHub 源码（2026 年 6 月）。Claude Code 速览：[第 13 章](/AI/Agent/ClaudeCode/第13章%20Skills.html)
