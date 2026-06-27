---
order: 17
date: 2026-06-27
---

# Worktrees 与并行会话

Git **worktree** 让同一仓库有多份独立工作区。Claude Code 用 `--worktree` 把每个会话隔离到独立 checkout，**并行改文件不冲突**。

Worktree 解决**文件隔离**；任务协调靠 subagent、Agent Teams 或 background agents——见 [Run agents in parallel](https://code.claude.com/docs/en/run-agents-in-parallel)。

## 为什么并行

单会话瓶颈：
- **时间**：任务串行
- **Context 污染**：长会话质量下降

Anthropic 内部与社区重度用户常见模式：**3–5 个并行会话**，各守一条 worktree。

## Claude Code 启动 Worktree

```shell
claude --worktree                    # 自动命名，如 bright-running-fox
claude --worktree feature-auth       # 指定名
claude -w "#1234"                    # 从 PR #1234 创建
claude -w "https://github.com/org/repo/pull/1234"
```

- 默认路径：`<repo>/.claude/worktrees/<name>/`
- 默认分支：`worktree-<name>`，从 **`origin/默认分支`** 分出（干净树）
- 将 `.claude/worktrees/` 加入 **`.gitignore`**

**首次**在某目录用 `--worktree` 前，须先 `claude` 一次并接受 **workspace trust**。

### `worktree.baseRef`

```json
{
  "worktree": { "baseRef": "head" }
}
```

| 值 | 行为 |
|----|------|
| `"fresh"`（默认） | 从 remote 默认分支，无本地未推送脏状态 |
| `"head"` | 从当前本地 HEAD，含 feature 分支与未推送提交 |

Subagent `isolation: worktree` 共用此设置。

### 会话中切换

Claude 可用 **`EnterWorktree`** 工具创建或切入已有 worktree（`.claude/worktrees/` 下）。

## `.worktreeinclude`

Worktree 是**新 checkout**，`.env` 等 gitignore 文件不会自动存在。项目根添加：

```text
.env
.env.local
config/secrets.json
```

语法同 `.gitignore`；**仅复制同时被 gitignore 的文件**。适用于 `--worktree`、subagent worktree、Desktop 并行会话。

## Subagent Worktree 隔离

```yaml
---
name: bulk-refactor
isolation: worktree
---
```

每个 subagent 获临时 worktree，**无改动则自动清理**。

## Background 与 Worktree

Background 会话（`/background`、`claude --bg`）在改文件前通常**移入** `.claude/worktrees/` 隔离。例外：
- 已在 linked worktree 内
- `worktree.bgIsolation: "none"` 关闭隔离

## `/batch` 大规模并行

```
/batch migrate src/ from Solid to React
```

1. 研究代码库 → 拆 5–30 单元
2. 展示计划 → 你批准
3. 每单元：**隔离 worktree + 后台 agent** → 实现、测试、开 PR
4. 需要 git 仓库

## 清理

| 情况 | 行为 |
|------|------|
| 无未提交/未跟踪/新 commit | 退出时**自动删除** worktree 与分支（有命名时可能 prompt） |
| 有改动 | prompt 保留或删除 |
| `-p --worktree` | **不**自动清理，需 `git worktree remove` |
| 孤儿 worktree | `cleanupPeriodDays` 过期 sweep；运行中 agent 会 `git worktree lock` |

```shell
git worktree list
git worktree remove .claude/worktrees/feature-auth
git worktree remove --force <path>   # 有未提交改动时
```

## 手动 Git Worktree

```shell
git worktree add ../project-feature-a -b feature-a
cd ../project-feature-a && claude
git worktree remove ../project-feature-a
```

每个 worktree 需**单独**装依赖、配虚拟环境。

## 非 Git VCS

Mercurial / Perforce / SVN：配置 **`WorktreeCreate` / `WorktreeRemove` hooks** 替代 git 逻辑；此时 **不**处理 `.worktreeinclude`，需在 hook 内复制本地配置。

## 并行实操布局

```
Tab1 [红] claude -w plan-auth      → Plan Mode + Opus
Tab2 [蓝] claude -w impl-auth      → acceptEdits 实施
Tab3 [绿] claude -w review         → Sonnet 审查
Tab4 [黄] claude -w logs           → 只读分析日志
```

Shell 别名示例：

```bash
alias cw='claude --worktree'
```

## 会话迁移

```shell
claude --teleport      # Web → 本地
/desktop               # 本地 → Desktop App
claude agents          # 统一查看 background 会话
```

## 最佳实践

1. **任务边界清晰**——一会话一目标
2. **避免同文件并行写**
3. **复杂任务先 Plan** 再拆 worktree 执行
4. **合并前**在 agent view 删除 session 前先 push/merge worktree 改动
5. **Monorepo**：配合 nested skills、path-scoped rules（第 6、13 章）

> **资料来源**：[Worktrees](https://code.claude.com/docs/en/worktrees)、[Common workflows](https://code.claude.com/docs/en/common-workflows)、[Agent view](https://code.claude.com/docs/en/agent-view)
