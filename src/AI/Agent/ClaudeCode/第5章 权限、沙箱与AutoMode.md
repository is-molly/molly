---
order: 5
date: 2026-06-26
---

# 权限、沙箱与 Auto Mode

## 权限模式

Claude Code 提供了六种权限模式，影响 Claude 在执行操作前是否需要征求你的同意，从而在便利性和安全性之间做出不同的权衡。

### 六种模式速览

| 模式 | Claude 不经确认可以做什么 | 适用场景 |
|------|---------------------------|----------|
| `default` | 读取文件 | 入门使用、敏感操作 |
| `acceptEdits` | 读取和编辑文件（受保护目录除外） | 迭代开发时审查代码 |
| `plan` | 读取文件（只读） | 探索代码库、制定重构方案 |
| `auto` | 所有操作，由后台分类器安全审核 | 长时间任务、减少确认疲劳 |
| `bypassPermissions` | 所有操作（受保护目录除外） | 仅限隔离容器和虚拟机 |
| `dontAsk` | 仅预先批准的工具 | 非交互式环境（CI 等） |

### 受保护路径

无论何种模式（`bypassPermissions` 除外），对以下路径的**写入**不会自动批准，防止损坏仓库状态与 Claude Code 自身配置：

- `.git/`、`.config/git/`、`.vscode/`、`.idea/`、`.husky/`、`.cargo/`、`.devcontainer/`、`.yarn/`、`.mvn/`
- **`.claude/` 整体受保护**，唯一例外是 **`.claude/worktrees/`**（Claude 存放 git worktree 的位置）
- 另有受保护文件：`.gitconfig`、`.zshrc`、`.npmrc`、`.mcp.json`、`.claude.json` 等（见官方 [Permissions](https://code.claude.com/docs/en/permissions) 完整列表）

> 旧文档曾写 `.claude/commands/`、`.claude/skills/` 可写——**已过时**。写入 `.claude/` 会触发确认；会话内可选择「允许 Claude 编辑自身设置」以跳过后续 `.claude/` 写入提示。

### 切换模式

**会话中切换**：按 `Shift+Tab` 在 `default` → `acceptEdits` → `plan` → `auto` 之间循环切换，状态栏会显示当前模式。

> v2.1.111 起 **`--enable-auto-mode` 已移除**；`auto` 默认出现在 `Shift+Tab` 循环中。启动时可用 `--permission-mode auto`。`bypassPermissions` 需以 `--permission-mode bypassPermissions` 启动才会出现在循环中。

**启动时指定**：

```shell
claude --permission-mode plan
claude --permission-mode acceptEdits
```

**设为默认**：

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

**非交互模式**（`-p` 模式同样支持）：

```shell
claude -p "refactor auth" --permission-mode acceptEdits
```

## Auto Mode（自动模式）

Auto Mode 是 Claude Code 的一项研究预览功能——Claude 可以跳过权限确认弹窗自主执行操作，同时有一个**后台分类器**在每次操作执行前审核安全性。

### 可用条件

- 需要 Team、Enterprise 或 API 计划
- 需要管理员在 [Claude Code 管理后台](https://claude.ai/admin-settings/claude-code) 启用
- 需要 Claude Sonnet 4.6 或 Opus 4.6
- 不支持 Haiku、Claude 3 系列模型或第三方 API 提供方

### 分类器如何工作

Auto Mode 的分类器运行在 **Claude Sonnet 4.6** 上（即使主会话使用其他模型）。

**决策顺序**（第一条匹配即生效）：

1. 匹配你的 allow/deny 规则的操作直接放行或拒绝
2. 只读操作和工作目录内的文件编辑自动批准
3. 其余操作送入分类器判断
4. 分类器拒绝时，Claude 收到原因并尝试替代方案

**关键安全设计**：
- 分类器**永远看不到工具结果**——注入到文件或网页中的恶意指令无法直接影响分类器
- 服务器端探针扫描进来的工具结果，标记可疑内容
- 你的 CLAUDE.md 内容会被送给分类器，所以项目指令中描述的操作会被纳入判断

### 默认的阻止规则

分类器默认信任你的工作目录和当前 Git 仓库的远程地址。以下操作默认被阻止：

- 下载并执行代码（`curl | bash` 类操作）
- 向外部端点发送敏感数据
- 生产环境部署和数据库迁移
- 对象存储的批量删除
- 授予 IAM 或仓库权限
- 修改共享基础设施
- 不可逆地删除会话开始前就存在的文件
- 危险的源码控制操作（force push、直接推 main）

### 回退机制

如果分类器连续拒绝 3 次或累计拒绝 20 次，Auto Mode 暂停并恢复手动确认模式。

管理员可通过 `autoMode.environment` 设置将公司内部的仓库、内部服务等标记为受信任基础设施。

## 权限规则

### 通过 `/permissions` 或 settings.json 配置

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Read(~/.zshrc)"
    ],
    "deny": [
      "Bash(curl *)",
      "Read(./.env)",
      "Read(./secrets/**)"
    ],
    "ask": []
  }
}
```

### 规则语法（Permission Rule Syntax）

```
工具名(模式)
```

- `Bash(npm run *)`：所有以 `npm run` 开头的命令
- `Read(./.env)`：精确匹配 `.env`
- `Read(./.env.*)`：匹配 `.env.local`、`.env.production` 等
- `Read(./secrets/**)`：secrets 目录及其所有子目录
- `Edit`：所有 Edit 操作
- `*`：所有工具

### 各模式的行为差异

| 模式 | allow 规则 | deny 规则 | ask 规则 | 未匹配的操作 |
|------|-----------|----------|---------|-------------|
| `default` | 自动批准 | 阻止（提示 Claude 原因） | 弹出确认框 | 弹出确认框 |
| `acceptEdits` | 自动批准 | 阻止 | 弹出确认框 | 自动批准（受保护目录除外） |
| `plan` | 自动批准 | 阻止 | 弹出确认框 | 弹出确认框（不可编辑） |
| `auto` | 自动批准 | 阻止 | - | 分类器判断 |
| `dontAsk` | 自动批准 | 阻止 | 视为 deny | 自动拒绝 |

### 权限规则的合并行为

不同作用域的权限规则**合并**而非覆盖：
- User + Project + Local 的 allow/deny/ask 规则都会生效
- Managed 规则优先级最高且不可被覆盖

## 沙箱（Sandbox）

Claude Code 的沙箱功能使用操作系统级原语为所有 Bash 命令提供文件系统和网络隔离。

### 原理

**文件系统隔离**：
- 默认：对工作目录及子目录有读写权限
- 默认：对整个计算机有只读权限（某些被拒绝的目录除外）
- 不可修改工作目录外的文件

**网络隔离**：
- 只有批准的域名可以访问
- 新的域名请求触发权限提示
- 所有子进程继承相同的隔离边界

**操作系统级强制执行**：
- macOS：使用 Seatbelt 框架
- Linux / WSL2：使用 [bubblewrap](https://github.com/containers/bubblewrap)
- WSL1 不支持（缺少所需内核特性）

### 安装（Linux / WSL2）

```shell
# Ubuntu/Debian
sudo apt-get install bubblewrap socat

# Fedora
sudo dnf install bubblewrap socat
```

macOS 无需额外安装。

### 启动沙箱

```
/sandbox
```

在打开的菜单中选择沙箱模式即可。

### 两种沙箱模式

| 模式 | 行为 |
|------|------|
| **Auto-allow** | 沙箱内命令自动批准，失败的退回到正常权限流程 |
| **Regular permissions** | 所有 Bash 命令都走标准权限流程 |

两种模式的文件系统和网络隔离相同，区别仅在于是否自动批准沙箱内的命令。

### 安全价值

即使 Claude Code 被提示注入攻击成功操纵，沙箱仍能保证系统安全：

- 无法修改 `~/.bashrc` 等关键配置文件
- 无法修改 `/bin/` 下的系统文件
- 无法将数据外泄到攻击者控制的服务器
- 无法从未授权的域名下载恶意脚本
- 所有越界尝试被 OS 级阻止并通知用户

### 已知限制

- `docker` 命令在沙箱内不兼容（建议在 `excludedCommands` 中配置）
- `watchman` 在沙箱内不兼容（Jest 用户建议 `jest --no-watchman`）
- 沙箱提供一种逃生舱机制：命令因沙箱限制失败时，Claude 会分析失败并可以用 `dangerouslyDisableSandbox` 参数在沙箱外重试（走正常权限流程）

### 开源沙箱运行时

沙箱运行时已作为 npm 包开源，可用于你自己的 Agent 项目：

```shell
npx @anthropic-ai/sandbox-runtime <command-to-sandbox>
```

> **资料来源**：[Claude Code Permission Modes](https://code.claude.com/docs/en/permission-modes)、[Sandboxing](https://code.claude.com/docs/en/sandboxing)
