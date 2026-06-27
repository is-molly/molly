---
order: 6
date: 2026-06-26
---

# CLAUDE.md 与项目知识

## CLAUDE.md 的本质

CLAUDE.md 不是写给 Claude 的"提示词"，而是 Claude 对项目的**持久知识库**。每次启动会话时，Claude 自动加载所有的 CLAUDE.md 文件。Claude 在执行长期自主任务时，还可以自己编辑 CLAUDE.md，把解决过程中的经验写进去，供后续会话使用。

## 四级记忆体系

Claude Code 支持四个层次的 CLAUDE.md，从最广泛到最具体：

| 记忆类型 | 存储位置 | 用途 | 共享范围 |
|----------|----------|------|----------|
| **Enterprise 策略** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux: `/etc/claude-code/CLAUDE.md`<br>Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | 组织级的编码规范、安全策略 | 全组织所有用户 |
| **User 记忆** | `~/.claude/CLAUDE.md` | 个人的全局偏好（所有项目生效） | 仅自己 |
| **Project 记忆** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 项目共享的架构、规范、工作流 | 团队成员（通过 git） |
| **Project 本地记忆** | `./CLAUDE.local.md` | 个人项目偏好（建议加入 `.gitignore`） | 仅自己 |

层级越高优先级越高，更具体的记忆叠加在更广泛的记忆之上。

## 递归查找

Claude Code 会递归向上查找 CLAUDE.md，从当前工作目录到根目录 `/` 的每一层父目录都会被检查，这意味着你可以在大型项目中灵活组织：

```
project/
├── CLAUDE.md              ← 项目通用记忆
├── packages/
│   ├── frontend/
│   │   └── CLAUDE.md      ← 前端子项目的记忆
│   └── backend/
│       └── CLAUDE.md      ← 后端子项目的记忆
```

在 `packages/frontend/` 运行 `claude`，两个 CLAUDE.md 都会被加载。

此外，Claude 还会在当前工作目录下的**子目录树**中发现嵌套的 CLAUDE.md，但不会在启动时加载，只在实际操作对应子树的文件时才包含。

## CLAUDE.md 的内容结构

### `/init` 自动生成

在项目目录中运行以下命令，Claude 会分析项目并自动生成初始的 CLAUDE.md：

```
/init
```

### 推荐的记忆内容

- **常用命令**（构建、测试、lint），避免 Claude 反复搜索
- **代码风格偏好**和命名规范
- **重要架构模式**和设计决策
- **项目特定的约定**

### 记忆最佳实践

- **具体明确**："使用 2 空格缩进" 好于 "格式化好代码"
- **用结构组织**：每条记忆用 bullet point 写，相关记忆分组在标题下
- **定期回顾**：项目演化时更新记忆，确保 Claude 始终有最新信息

### 示例

```markdown
# 项目架构
- 使用 Feature-Sliced Design 分层
- 状态管理统一使用 Zustand

# 编码规范
- TypeScript 严格模式
- 组件使用函数式 + Hooks
- 禁止使用 any 类型

# 常用命令
- 开发: `npm run dev`
- 测试: `npm run test`
- Lint: `npm run lint`
- 构建: `npm run build`

# Git 工作流
- 遵循 Conventional Commits
- PR 标题格式: `type(scope): 描述`
- @docs/git-instructions.md
```

## 快速添加记忆：`#` 快捷方式

在 Claude Code 中输入以 `#` 开头的内容，会被直接存储为记忆：

```
# 测试文件统一放在 __tests__ 目录下
```

Claude 会提示选择存储到哪个记忆文件。

## 用 `/memory` 编辑记忆

```
/memory
```

在系统编辑器中打开记忆文件进行更全面的编辑。

## CLAUDE.md Imports（引用导入）

CLAUDE.md 文件可以通过 `@path/to/file` 语法导入其他文件：

```markdown
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
- @~/.claude/my-project-instructions.md
```

支持相对路径和绝对路径。导入最多递归 **4 层**（4 hops）。`@path` 在 Markdown 代码块内不会被解析为导入。

> 跨 git worktree 共享个人指令时，推荐 `@~/.claude/my-project-instructions.md` 而非仅存在于单个 worktree 的 `CLAUDE.local.md`。

### 排除特定 CLAUDE.md

在 settings.json 中可通过 `claudeMdExcludes` 跳过某些 CLAUDE.md：

```json
{
  "claudeMdExcludes": ["**/vendor/**/CLAUDE.md"]
}
```

## `.claude/rules/` 路径规则（推荐）

大型项目应将指令拆到 `.claude/rules/*.md`，支持递归子目录。每条规则可带 `paths` frontmatter，**仅在 Claude 操作匹配文件时加载**，比全量 CLAUDE.md 更省 context：

```yaml
---
paths:
  - "src/api/**"
  - "src/types/**"
---
# API 层编码规范
- 所有 endpoint 必须带 request validation
```

- 无 `paths` 的规则：启动时加载，优先级同 `.claude/CLAUDE.md`
- 用户级规则：`~/.claude/rules/`（先于项目规则加载，项目规则优先级更高）
- 支持 symlink 跨项目共享规则

## 组织级记忆

企业可以通过配置管理系统（MDM、Group Policy、Ansible 等）将一个 `CLAUDE.md` 文件部署到所有开发者机器的对应位置。

另外，在 Managed settings 中可以设置 `claudeMd` 字段注入组织级指令：

```json
{
  "claudeMd": "Always run make lint before committing. Follow SOC2 requirements."
}
```

> **资料来源**：[Claude Code 官方文档 - Memory](https://code.claude.com/docs/en/memory)
