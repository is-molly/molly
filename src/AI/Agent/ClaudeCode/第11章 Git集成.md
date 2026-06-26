---
order: 11
date: 2026-06-26
---

# Git 集成

## Claude 直接操作 Git

Claude Code 原生支持 Git——它可以自主完成提交、创建分支、推送、创建 PR 等完整的 Git 工作流。你只需要描述意图，Claude 会完成所有 Git 操作。

### 创建提交

```
claude "commit my changes with a descriptive message"
```

```
帮我提交当前的修改，写一个 conventional commit 消息
```

Claude 会：
1. 检查当前修改（`git status` / `git diff`）
2. 分析改动内容
3. 生成符合规范的提交消息
4. 执行 `git add` 和 `git commit`

### 创建分支

```
claude "create a branch for the new auth feature"
```

或使用 `/branch` 命令：

```
/branch feat/oauth2-integration
```

### 创建 Pull Request

```
claude "push and create a PR for my changes"
```

Claude 会生成 PR 标题和描述，包含改动摘要和测试说明。

## 代码审查（Code Review）

### GitHub Code Review

Claude Code 提供自动化的 PR 审查能力。配置 GitHub Actions 后，每次 PR 提交会自动触发 Claude 进行 Code Review。

### @claude on GitHub

在 GitHub PR 或 Issue 中 `@claude`，可以触发 Claude Code 进行审查、回答问题或生成修复。

### 内置 Code Review 工作流

```
帮我 review 一下当前分支的所有改动
```

Claude 会检查：
- 代码质量和可读性
- 安全漏洞
- 潜在 Bug
- 测试覆盖
- 与项目编码规范的一致性

## CI/CD 集成

### GitHub Actions

```yaml
# .github/workflows/claude-code-review.yml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### GitLab CI/CD

类似地支持 GitLab CI/CD 集成，详情见 [官方文档](https://code.claude.com/docs/en/gitlab-ci-cd)。

## 提交消息风格

Claude Code 默认使用 Conventional Commits 规范生成提交消息：

```
type(scope): 简短的变更描述

可选的详细描述

可选的 footer（如 BREAKING CHANGE）
```

你可以通过 CLAUDE.md 定制提交消息的偏好格式：

```markdown
# Git 提交规范
- 遵循 Conventional Commits
- type 使用: feat, fix, refactor, docs, test, chore
- scope 使用英文小写
- 描述使用中文
```

## CLI 标志相关

| 命令/标志 | 说明 |
|-----------|------|
| `claude "commit..."` | 启动交互会话并请求提交 |
| `claude -p "commit..."` | 非交互模式提交 |
| `/branch <name>` | 创建分支 |
| `claude --worktree` | 在独立的 git worktree 中启动（第 17 章详述） |
| `claude project purge` | 删除项目的所有 Claude Code 状态（v2.1.126+） |

## 注意事项

1. **Claude 不会执行危险的 Git 操作**：force push、推送到 main/master、reset --hard 等操作需要显式确认或特定的权限配置。

2. **Auto Mode 下的 Git 操作**：分类器会阻止危险的源码控制操作（force push、直接推送 main），但允许推送到当前分支或 Claude 自己创建的分支。

3. **提交前验证**：建议通过 Hooks（第 14 章）配合 lint 和测试，确保 Claude 的每次提交都满足质量标准。

> **资料来源**：[Claude Code 官方文档 - Overview](https://code.claude.com/docs/en/overview)、[GitHub Actions](https://code.claude.com/docs/en/github-actions)、[Code Review](https://code.claude.com/docs/en/code-review)
