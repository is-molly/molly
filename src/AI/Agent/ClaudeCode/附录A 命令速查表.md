---
order: 29
date: 2026-06-26
---

# 附录A：命令速查表

## 全部 Slash Commands

### 会话管理

| 命令 | 功能 |
|------|------|
| `/clear` | 清空对话上下文 |
| `/compact` | 压缩上下文 |
| `/context` | 查看上下文使用 |
| `/cost` | 查看 Token 费用 |
| `/recap` | 查看会话摘要 |
| `/copy` | 复制内容到剪贴板 |
| `/resume` | 恢复之前会话 |

### 配置

| 命令 | 功能 |
|------|------|
| `/config` | 打开设置（或 `/config key=value` 直接设置） |
| `/model` | 切换模型 |
| `/effort` | 调整 Effort 等级 |
| `/color` | 设置会话颜色 |
| `/init` | 自动生成 CLAUDE.md |
| `/memory` | 编辑记忆文件 |
| `/permissions` | 管理权限规则 |
| `/statusline` | 自定义状态栏 |

### 工作流

| 命令 | 功能 |
|------|------|
| `/plan` | Plan Mode 处理下一条 prompt |
| `/branch` | 创建 Git 分支 |
| `/btw` | 后台执行（By The Way） |
| `/agents` | 管理子代理 |
| `/batch` | 大规模并行代码迁移 |
| `/loop` | 循环执行 prompt |
| `/simplify` | 审查和优化代码质量 |
| `/claude-api` | 加载 Claude API 文档 |

### 扩展与调试

| 命令 | 功能 |
|------|------|
| `/plugin` | 管理插件/市场 |
| `/mcp` | 查看 MCP Server 状态 |
| `/sandbox` | 配置沙箱 |
| `/debug` | 启用调试日志 |
| `/bug` | 报告 Bug |
| `/feedback` | 提交反馈 |
| `/help` | 显示帮助 |
| `/doctor` | 诊断安装状态 |

## CLI 命令

| 命令 | 功能 |
|------|------|
| `claude` | 启动交互会话 |
| `claude "prompt"` | 带初始 prompt 启动 |
| `claude -p "prompt"` | 非交互模式 |
| `claude -c` | 继续最近会话 |
| `claude -r "session"` | 恢复指定会话 |
| `claude update` | 更新版本 |
| `claude --version` | 查看版本 |
| `claude agents` | 查看代理视图 |
| `claude project purge` | 清除项目状态 |

## 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Shift+Tab` | 切换权限模式 |
| `Option+T` / `Alt+T` | 切换 Extended Thinking |
| `Ctrl+O` | 切换详细思考 |
| `Ctrl+B` | 后台化任务 |
| `Ctrl+V` | 粘贴图片 |
