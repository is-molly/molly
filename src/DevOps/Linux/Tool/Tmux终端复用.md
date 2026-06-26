---
order: 1
date: 2026-06-26
---

# Tmux终端复用

> 基于 tmux 3.6b（2026 年 5 月发布），本文持续追踪最新版本。
> 查看当前版本：`tmux -V`

---

## 1. 概述

### 1.1 什么是 tmux

tmux 是一个**终端复用器**（terminal multiplexer），核心功能是从一个终端屏幕创建、访问和控制多个虚拟终端。它基于 C/S（Client/Server）架构，包含一个后台 server 进程和若干 client 进程。

官方定义：

> tmux is a terminal multiplexer: it enables a number of terminals to be created, accessed, and controlled from a single screen.

### 1.2 核心价值

- **会话持久化**：SSH 断开或终端窗口关闭后，运行的程序不受影响，重新连接即可恢复现场
- **多任务并行**：一个窗口内切分多个面板，同时运行不同任务
- **可脚本化**：通过命令行构建和操控复杂开发环境
- **协作支持**：多人同时 attach 同一 session，适合结对编程和远程演示

### 1.3 与 GNU Screen 的对比

| 特性 | tmux | GNU Screen |
|------|------|------------|
| 窗口分割 | 支持任意方向、任意尺寸 | 有限的分割能力 |
| 配置文件语法 | 清晰，支持细粒度控制 | 较晦涩 |
| 插件生态 | 活跃（TPM + 众多插件） | 较少 |
| 活跃度 | 持续更新（~每年一个大版本） | 更新缓慢 |

---

## 2. 架构

### 2.1 层级结构

tmux 由四个层级组成，从外到内依次是：

```
Server（单一后台进程）
  └── Session（会话，可多个）
        └── Window（窗口，可多个）
              └── Pane（面板，可多个）
```

### 2.2 Server

tmux 启动后，会在后台运行一个 server 进程，管理所有 session。server 进程的 socket 文件默认位于 `/tmp/tmux-<UID>/default`。所有 client 通过该 socket 与 server 通信。

```shell
ps aux | grep tmux
```

输出中会看到 `tmux: server` 进程。最后一个 session 被 kill 后，server 进程自动退出。

### 2.3 Session（会话）

一个 session 是一个独立的工作环境，包含一组 window。session 的核心价值是**持久化**——即便 client 断开连接（detach），session 及其内部运行的所有程序仍然在后台继续运行。

- 每个 session 有唯一名称（可自定义或自动分配）
- 同一个 session 可被多个 client 同时 attach（适合结对编程或演示）
- session 之间完全隔离，互不干扰

### 2.4 Window（窗口）

一个 session 包含多个 window，类似浏览器中的标签页。每个 window 占满整个屏幕，一次只能看到一个 window。

- 每个 window 有独立编号（从 0 开始）和可选名称
- window 可被多个 session 同时链接（link/unlink），实现 session 间的资源共享

### 2.5 Pane（面板）

每个 window 可被分割成多个 pane。每个 pane 是一个独立的伪终端（pty），运行着独立的 shell 或程序。

- 默认情况下整个 window 是一个 pane
- 可垂直分割（左右各一个 pane）或水平分割（上下各一个 pane）
- 面板之间可调整大小、可相互切换焦点
- 3.5+ 支持**镜像布局**：主面板可放在底部或右侧（代替传统的顶部或左侧）
- 3.6+ 支持**原生滚动条**（`pane-scrollbars` 选项）

---

## 3. 安装与启动

### 3.1 安装

**macOS**

```shell
brew install tmux
```

**Ubuntu/Debian**

```shell
sudo apt install tmux
```

**CentOS/RHEL**

```shell
sudo yum install tmux
# 或
sudo dnf install tmux
```

**从源码编译（获取最新版本）**

```shell
git clone https://github.com/tmux/tmux.git
cd tmux
sh autogen.sh
./configure && make
sudo make install
```

### 3.2 启动

直接执行 `tmux`：

```shell
tmux
```

此时会创建一个新的 session（默认名称从 0 开始编号），并进入 tmux 环境。底部会出现一个状态栏，显示 session 名、window 列表、当前时间等信息。

### 3.3 退出 tmux

**分离（detach）—— 推荐**

分离会让 tmux client 退出，但 session 和内部运行的程序继续在后台运行。分离后可以随时重新 attach 回来。

```shell
快捷键：Ctrl+b d
# 或执行命令
tmux detach-client
```

**关闭（kill）**

关闭当前 window 或 session，内部运行的所有程序会被终止。

```shell
exit
# 或
Ctrl+d
# 或
Ctrl+b x   # 确认后关闭当前 pane
```

### 3.4 重新连接（attach）

```shell
tmux attach-session
# 简写
tmux attach
# 或
tmux a
```

如果只有一个 session，直接 attach 即可。若有多个 session，需要指定名称：

```shell
tmux a -t <session-name>
```

---

## 4. 前缀键

tmux 的快捷键采用"前缀键 + 命令键"的组合方式。默认前缀键是 `Ctrl+b`。

使用方式：先按住 `Ctrl`，再按 `b`，然后松开，再按其他键。

### 为什么要用前缀键

- 避免与终端中其他应用的快捷键冲突
- 设计哲学：先告诉 tmux"我要操作你了"，再告诉它"做什么"

### 操作示例

| 动作 | 完整按键序列 |
|------|-------------|
| 新建 window | `Ctrl+b` 然后 `c` |
| 切换到下一个 window | `Ctrl+b` 然后 `n` |
| 垂直分割 pane | `Ctrl+b` 然后 `%` |
| 水平分割 pane | `Ctrl+b` 然后 `"` |

> `"` 需要按 `Shift+'`，即和双引号同一个键。

---

## 5. 命令执行方式

tmux 支持三种执行命令的方式：

### 5.1 快捷键

按下 `Ctrl+b` 后，再按命令键。这是日常使用最频繁的方式。

### 5.2 命令行模式

`Ctrl+b :` 进入命令行模式，然后输入 tmux 命令。

```
Ctrl+b :resize-pane -D 10
```

这种方式适合执行不常用或参数较复杂的命令。

### 5.3 shell 命令

在 shell 中直接执行 tmux 命令。适合脚本化操作。

```shell
tmux new-session -s dev -n editor
tmux split-window -h
```

---

## 6. 配置

### 6.1 配置文件路径

tmux 的配置文件默认位于 `~/.tmux.conf`。若文件不存在，手动创建即可。

部分版本还支持 XDG 规范路径 `~/.config/tmux/tmux.conf`。

### 6.2 基础配置

```shell
# 开启鼠标支持（点击切换 pane、拖动调整大小、滚轮滚动历史）
set -g mouse on

# 回滚缓冲区大小（默认 1000，调大可查看更多历史输出）
set -g history-limit 100000

# 终端类型（3.6+ 推荐 tmux-256color）
set -g default-terminal "tmux-256color"

# 状态栏刷新间隔（秒）
set -g status-interval 2

# 窗口自动重命名（根据当前运行的进程名自动更新 window 名称）
set -wg automatic-rename on
```

### 6.3 修改前缀键

```shell
# 将前缀键改为 Ctrl+a
unbind C-b
set -g prefix C-a
bind C-a send-prefix
```

### 6.4 重新加载配置

```shell
tmux source-file ~/.tmux.conf
# 在 tmux 内执行
Ctrl+b :source-file ~/.tmux.conf
```

配置快捷键一键重载：

```shell
# 在 .tmux.conf 中添加
bind r source-file ~/.tmux.conf \; display "Config reloaded!"
```

之后按 `Ctrl+b r` 即可重载配置并显示提示。

### 6.5 tmux 3.6+ 新配置特性

```shell
# ---- 滚动条（3.6+）可选值：off / modal / on ----
# on：始终显示；modal：仅复制模式显示；off：不显示（默认）
set -g pane-scrollbars on
set -g pane-scrollbars-position right    # left / right（默认 right）

# ---- pane 边框风格（3.6+）----
# 可选值（默认 single）：single / double / heavy / simple / number / spaces
# double、heavy 需要终端支持 UTF-8
set -g pane-border-lines single

# ---- 活跃 pane 指示器（3.6+）----
# 可选值：colour（默认）/ arrows / both / off
set -g pane-border-indicators colour

# ---- 布局 ----
set -g window-option tiled-layout-max-columns 4   # tiled 布局最大列数（3.6+）

# ---- 其他 ----
set -g allow-set-title off                        # 禁止应用修改 pane 标题（3.5+）
```

---

## 7. Session 管理

### 7.1 创建

```shell
# 创建新 session
tmux new -s <session-name>

# 创建时同时指定 window 名
tmux new -s <session-name> -n <window-name>

# 后台创建（不自动 attach）
tmux new -s <session-name> -d
```

### 7.2 查看与切换

```shell
# 列出所有 session
tmux ls
# 或
tmux list-sessions
```

输出示例：

```
dev: 3 windows (created Wed Jun 26 10:30:45 2026)
logs: 1 windows (created Wed Jun 26 11:15:22 2026)
```

在 tmux 内部切换：

```
Ctrl+b s          # 弹出列表，交互选择
Ctrl+b (          # 上一个 session
Ctrl+b )          # 下一个 session
```

在 shell 中切换：

```shell
tmux switch -t <session-name>
```

### 7.3 重命名

```
Ctrl+b $
```

或在 shell 中：

```shell
tmux rename-session -t <old-name> <new-name>
```

### 7.4 删除

```shell
tmux kill-session -t <session-name>

# 删除所有 session（server 退出）
tmux kill-server
```

---

## 8. Window 管理

### 8.1 创建

在 tmux 内部：

```
Ctrl+b c
```

从 shell 中创建：

```shell
tmux new-window -t <session-name> -n <window-name>
```

### 8.2 切换

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b n` | 下一个 window |
| `Ctrl+b p` | 上一个 window |
| `Ctrl+b 0-9` | 跳到指定编号 |
| `Ctrl+b f` | 搜索 window 名并跳转 |
| `Ctrl+b w` | 列出所有 window 交互选择 |

### 8.3 重命名

```
Ctrl+b ,
```

### 8.4 移动

```shell
# 将 window 移到另一个 session
tmux move-window -t <target-session> -s <source-session>:<window-id>

# 在 tmux 内部将当前 window 变成独立 session
Ctrl+b :
move-window -t <new-session-name>
```

### 8.5 链接与取消链接

```shell
# 将 window 链接到另一个 session（共享）
tmux link-window -t <target-session>:<window-id> -s <source-session>:<window-id>

# 取消链接（仅从当前 session 移除，不关闭 window）
tmux unlink-window -t <session>:<window-id>
```

### 8.6 关闭

```
Ctrl+b &
```

或从 shell：

```shell
tmux kill-window -t <session>:<window-id>
```

---

## 9. Pane 管理

### 9.1 分割

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b %` | 垂直分割（左右） |
| `Ctrl+b "` | 水平分割（上下） |

从 shell 中分割：

```shell
tmux split-window -h -t <target>   # 垂直分割
tmux split-window -v -t <target>   # 水平分割
```

### 9.2 切换焦点

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b 方向键` | 向对应方向切换 |
| `Ctrl+b o` | 按顺序循环切换 |
| `Ctrl+b q` | 显示 pane 编号，按数字跳转 |

### 9.3 调整大小

**使用快捷键微调**（每次 1 个单元格）：

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b Ctrl+方向键` | 向对应方向调整大小 |

**使用命令行模式指定步长**：

```
Ctrl+b :resize-pane -D 10    # 向下扩大 10
Ctrl+b :resize-pane -U 10    # 向上扩大 10
Ctrl+b :resize-pane -L 10    # 向左扩大 10
Ctrl+b :resize-pane -R 10    # 向右扩大 10
```

**使用鼠标**：若开启 `set -g mouse on`，可直接拖拽 pane 边界调整大小。

### 9.4 布局

`Ctrl+b 空格` 循环切换预设布局：

| 布局 | 说明 |
|------|------|
| `even-horizontal` | 水平均匀分布 |
| `even-vertical` | 垂直均匀分布 |
| `main-horizontal` | 主面板在上，其余在下 |
| `main-vertical` | 主面板在左，其余在右 |
| `main-horizontal-mirror` | 主面板在下，其余在上（**3.5+**） |
| `main-vertical-mirror` | 主面板在右，其余在左（**3.5+**） |
| `tiled` | 瓷砖式均分 |

手动设置布局：

```
Ctrl+b :select-layout <layout-name>
```

### 9.5 放大与还原

`Ctrl+b z` 将当前 pane 临时放大到整个 window，再次按则还原。

### 9.6 关闭

```
Ctrl+b x        # 确认后关闭
exit            # 在 pane 中执行
Ctrl+d          # 在 pane 中执行
```

### 9.7 转换为 Window

`Ctrl+b !` 将当前 pane 剥离为独立 window。

### 9.8 同步输入

在同一个 window 的所有 pane 中同时执行相同的输入，适合批量操作多台服务器。

```shell
# 开启
Ctrl+b :setw synchronize-panes on

# 关闭
Ctrl+b :setw synchronize-panes off

# 查看当前状态
Ctrl+b :setw synchronize-panes?
```

### 9.9 滚动条（3.6+）

tmux 3.6 引入原生 pane 滚动条支持。`pane-scrollbars` 有三个可选值：

| 值 | 效果 |
|----|------|
| `off` | 不显示（默认） |
| `modal` | 仅进入复制模式时显示 |
| `on` | 始终显示 |

```shell
# 始终显示滚动条
set -g pane-scrollbars on

# 滚动条位置：left / right（默认 right）
set -g pane-scrollbars-position right
```

启用后，pane 右侧会出现滚动条，可拖拽或点击滚动。

---

## 10. 复制模式

tmux 的复制模式用于浏览历史输出并复制文本。

### 10.1 进入复制模式

```
Ctrl+b [
```

### 10.2 Vi 风格操作（推荐）

在 `.tmux.conf` 中添加：

```shell
set -g mode-keys vi
```

进入复制模式后：

| 按键 | 作用 |
|------|------|
| `h/j/k/l` | 左/下/上/右移动 |
| `Ctrl+u/d` | 上/下半页翻动 |
| `g/G` | 跳转到顶部/底部 |
| `/` | 向下搜索 |
| `?` | 向上搜索 |
| `n/N` | 下一个/上一个搜索结果 |
| `空格` | 开始选择文本 |
| `回车` | 复制选中的文本并退出复制模式 |

### 10.3 Emacs 风格操作（默认）

| 按键 | 作用 |
|------|------|
| `Ctrl+f/b` | 前/后移动 |
| `Ctrl+n/p` | 下/上移动 |
| `Ctrl+v/Alt+v` | 下/上半页翻动 |
| `空格` | 开始选择文本 |
| `回车` | 复制选中文本并退出复制模式 |

### 10.4 粘贴

```
Ctrl+b ]
```

### 10.5 复制模式增强（3.5+）

- **超链接显示**（3.5+）：复制模式中可识别并显示超链接
- **搜索计数**（3.5+）：`search_count` 和 `search_count_partial` 格式变量
- **`-d` 标志**（3.5+）：`copy-mode -d` 支持在复制模式中翻页
- **`selection-mode` 命令**（3.6+）：显式设置复制模式的选择模式
- **不区分大小写搜索**（3.6+）：全小写字符串时自动不区分大小写

### 10.6 系统剪贴板

配合 `tmux-yank` 插件，选中文本后按 `y`（vi 模式）即可复制到系统剪贴板。

---

## 11. 状态栏

### 11.1 默认状态栏

tmux 底部状态栏默认显示：

- 左侧：当前 session 名
- 中间：window 列表（当前 window 高亮）
- 右侧：当前时间

### 11.2 自定义样式

```shell
# 状态栏颜色
set -g status-bg colour235
set -g status-fg colour250

# 设置左右两栏最大长度
set -g status-left-length 100
set -g status-right-length 100

# 自定义内容
set -g status-left "[#S] "
set -g status-right "#{prefix_highlight} | %Y-%m-%d %H:%M"
```

### 11.3 多行状态栏（3.6+）

tmux 3.6 引入更友好的默认第二行和第三行状态栏：

```shell
# 使用更丰富的默认状态栏
set -g status-format[1] "..."
```

### 11.4 主题模式（3.6+）

tmux 3.6 支持 Mode 2031 自动报告明/暗主题：

- 自动检测终端背景色并报告主题
- 对不支持该转义序列的终端，tmux 会从背景色猜测主题

---

## 12. 脚本化

tmux 支持通过 shell 命令构建完整的开发环境，适合制作一键启动脚本。

### 12.1 常用构建命令

```shell
# 后台创建 session
tmux new-session -d -s mysession -n mywindow

# 在指定位置执行命令
tmux send-keys -t mysession:mywindow 'cd /project' C-m

# 垂直分割 pane
tmux split-window -h -t mysession:mywindow

# 水平分割 pane
tmux split-window -v -t mysession:mywindow

# 创建新 window
tmux new-window -t mysession -n another

# 最终 attach 到 session
tmux attach -t mysession
```

### 12.2 实战脚本

```shell
#!/bin/bash
# 一键启动开发环境

SESSION="dev"
PROJECT_DIR="/Users/xxx/project"

# 若 session 已存在则 attach
tmux has-session -t $SESSION 2>/dev/null
if [ $? -eq 0 ]; then
    tmux attach -t $SESSION
    exit 0
fi

# 创建 session
tmux new-session -d -s $SESSION -n editor

# 在第一个 pane 打开 vim
tmux send-keys -t $SESSION:editor "cd $PROJECT_DIR" C-m
tmux send-keys -t $SESSION:editor "vim" C-m

# 分割出第二个 pane 跑构建命令
tmux split-window -h -t $SESSION:editor
tmux send-keys -t $SESSION:editor.1 "cd $PROJECT_DIR" C-m
tmux send-keys -t $SESSION:editor.1 "npm run dev" C-m

# 创建第二个 window 跑日志
tmux new-window -t $SESSION -n logs
tmux send-keys -t $SESSION:logs "tail -f /var/log/app.log" C-m

# attach
tmux attach -t $SESSION
```

### 12.3 C-m 说明

`C-m` 代表 Ctrl+m，即回车键（Enter）。在 `send-keys` 中发送 `C-m` 相当于按了一次回车。也可用 `Enter` 代替。

---

## 13. 插件系统

tmux 的插件通过 **TPM**（Tmux Plugin Manager）管理。

### 13.1 安装 TPM

```shell
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

### 13.2 配置插件

在 `~/.tmux.conf` 中添加插件列表：

```shell
# 插件管理器本身
set -g @plugin 'tmux-plugins/tpm'

# 插件列表
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @plugin 'tmux-plugins/tmux-yank'

# 初始化 TPM（必须放在最后）
run '~/.tmux/plugins/tpm/tpm'
```

### 13.3 管理插件

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b I` | 安装配置文件中列出的所有插件（大写 I） |
| `Ctrl+b U` | 更新所有插件 |
| `Ctrl+b Alt+u` | 删除未在配置文件中列出的插件 |

### 13.4 核心插件详解

#### tmux-resurrect

**作用**：将当前 session 的状态保存到磁盘，包括 window 布局、pane 布局、当前路径、运行中的程序。

```shell
# 手动保存
Ctrl+b Ctrl+s

# 手动恢复
Ctrl+b Ctrl+r
```

**可保存的内容**：
- window 和 pane 的布局
- 每个 pane 的当前工作目录
- 每个 pane 中运行的程序的启动命令（如 `vim file.txt`）

**注意**：`tmux-resurrect` 并非"复活"进程，而是记录启动命令，恢复时重新执行该命令。

#### tmux-continuum

**作用**：自动化引擎，自动保存和恢复。

```shell
# 自动保存间隔（分钟），设为 0 则关闭
set -g @continuum-save-interval '15'

# 开机自动恢复（tmux 启动时自动读取最近的快照）
set -g @continuum-restore 'on'

# 恢复时是否显示状态信息
set -g @continuum-restore-status 'on'
```

#### tmux-yank

**作用**：将 tmux 复制模式中选中的文本同步到系统剪贴板。

```shell
set -g @plugin 'tmux-plugins/tmux-yank'
```

使用方式：进入复制模式选中文本后，按 `y` 即可复制到系统剪贴板（vi 模式）。

**macOS 依赖**：`pbcopy`（macOS 自带）

**Linux 依赖**：

```shell
sudo apt install xclip   # Ubuntu/Debian
sudo yum install xclip   # CentOS/RHEL
```

### 13.5 其他常用插件

| 插件名 | 作用 |
|--------|------|
| `tmux-cpu` | 状态栏显示 CPU 使用率 |
| `tmux-battery` | 状态栏显示电池电量（macOS/Linux） |
| `tmux-prefix-highlight` | 按下前缀键时高亮状态栏 |
| `tmux-sessionist` | 增强 session 管理功能 |
| `tmux-pain-control` | 优化 pane 切换和调整大小的快捷键 |
| `catppuccin/tmux` | 流行的主题插件，支持丰富的自定义 |

---

## 14. 会话共享

tmux 支持多人同时 attach 同一个 session，适合结对编程或远程演示。

### 14.1 共享方式

用户 A 创建 session：

```shell
tmux new -s shared
```

用户 B attach 同一个 session：

```shell
tmux a -t shared
```

两人看到同一个屏幕，任何一方的输入和操作都会实时同步显示。

### 14.2 只读模式

如果只想让别人看，不让他操作：

```shell
tmux attach -t shared -r
```

### 14.3 权限控制（3.4+）

tmux 3.4 引入了增强的访问控制列表（ACL），支持多用户环境的细粒度权限控制：

```shell
# 设置 server 访问控制
tmux set -g server-access-control on
tmux add -g access-control-list <user>:<permissions>
```

### 14.4 多人协作的最佳实践

- 使用一个独立的 session 用于协作，不要和个人开发 session 混用
- 开启 `synchronize-panes` 前确认所有人都知道
- 协作结束后，保持 session 供他人继续使用，或直接 kill

---

## 15. 调试与故障排查

### 15.1 查看日志

```shell
# server 日志（写入当前目录 tmux-server.log）
tmux -v

# 更详细
tmux -vv

# client 日志
tmux -v attach
```

### 15.2 常用调试命令

```shell
# 查看所有 session
tmux ls

# 查看 session 详细信息
tmux list-sessions -F '#{session_name}: #{session_windows} windows'

# 查看 window 信息
tmux list-windows -t <session>

# 查看 pane 信息
tmux list-panes -t <session>:<window>

# 显示所有可用的格式变量
tmux list-format
```

### 15.3 常见问题

#### 终端颜色异常

表现为 vim 颜色不对、程序输出颜色缺失。

检查：

```shell
echo $TERM
# 应输出 tmux-256color
```

在 `.tmux.conf` 中设置：

```shell
set -g default-terminal "tmux-256color"
```

#### 复制粘贴不工作

可能原因：
- 鼠标模式干扰：`set -g mouse on` 可能会影响选择行为
- 缺少 `tmux-yank` 插件
- 系统剪贴板工具缺失（Linux 需安装 xclip）

#### 自动恢复失败

检查：

```shell
# 确认已配置 @continuum-restore 'on'
grep continuum-restore ~/.tmux.conf

# 确认至少保存过一次
ls -la ~/.tmux/resurrect/
# 应看到 latest 文件
```

#### SSH 断开后无法 attach

tmux 本身不受 SSH 影响。重新 SSH 后执行 `tmux a` 即可。

若 SSH 断开后 tmux 进程消失，检查是否在 tmux 会话内执行了 `exit` 关闭了最后一个 window。

#### 窗口卡死或无响应

可能是某个 pane 内的程序卡住。尝试：

```
Ctrl+b 方向键     # 切换焦点到其他 pane
Ctrl+b x          # 强制关闭卡住的 pane
```

或从外部 kill 整个 session：

```shell
tmux kill-session -t <session>
```

---

## 16. 版本演进

### 版本时间线

| 版本 | 发布日期 | 主要变化 |
|------|---------|---------|
| 3.2 | 2022 年 4 月 | 基础功能完善 |
| 3.2a | 2022 年 6 月 | 错误修复 |
| 3.3 | 2022 年 6 月 | 安全修复（缓冲区溢出） |
| 3.3a | 2022 年 6 月 | 错误修复 |
| **3.4** | **2024 年 2 月** | **SIXEL 图像支持、多用户 ACL、性能优化** |
| **3.5** | **2024 年 9 月** | **扩展键支持、超链接显示、镜像布局、command-error hook、escape 10ms** |
| 3.5a | 2024 年 10 月 | 错误修复 |
| **3.6** | **2025~2026** | **原生滚动条、Mode 2031 主题、selection-mode、tiled-layout-max-columns、增强 Unicode 处理、OSC 52 检测、nice 默认状态栏** |
| 3.6a | 2026 | 错误修复 |
| **3.6b** | **2026 年 5 月** | **当前最新**（缓冲区溢出修复、鼠标拖拽修复等） |

### 3.4 → 3.5 关键变更

- **扩展键支持**：重构键处理，更贴合 xterm 行为，支持 mode 1 和 mode 2
- **`extended-keys-format`**：新选项控制键格式
- **自动清除弹出/菜单**：进入命令提示符时自动清除覆盖层
- **复制模式超链接**：支持显示和交互超链接
- **镜像布局**：`main-horizontal-mirror`、`main-vertical-mirror`
- **`command-error` hook**：命令失败时触发
- **`allow-set-title`**：禁止应用修改 pane 标题
- **默认 escape 时间降至 10ms**

### 3.5 → 3.6 关键变更

- **原生滚动条**：`pane-scrollbars` 选项
- **Mode 2031 主题**：自动报告明/暗主题
- **`selection-mode` 命令**：显式控制复制模式的选择模式
- **图片保存/恢复**：在 alternate screen 中保存和恢复图片
- **`tiled-layout-max-columns`**：tiled 布局最大列数
- **`pane-border-lines spaces`**：空格边框
- **增强 Unicode 处理**：韩文、区域指示符、表情符号修饰符
- **OSC 52 检测**：通过设备属性报告检测剪贴板支持
- **更友好的默认状态栏**
- **`noattr` 样式属性**：可配置是否忽略属性
- **格式变量增强**：支持 else if、排序、布尔表达式加强

---

## 17. 附录

### 17.1 完整配置示例

```shell
# ==========================================
# Tmux 配置文件 ~/.tmux.conf
# 适用于 tmux 3.6+
# ==========================================

# ---- 基础设置 ----
set -g mouse on
set -g history-limit 100000
set -g default-terminal "tmux-256color"
set -g status-interval 2
set -wg automatic-rename on

# ---- 前缀键：改为 Ctrl+a ----
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# ---- 快捷键 ----
# 重载配置
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# 分割 pane（更顺手）
bind | split-window -h
bind - split-window -v

# 调整大小（Shift+方向键，每次 5 个单元格）
bind -r C-Up resize-pane -U 5
bind -r C-Down resize-pane -D 5
bind -r C-Left resize-pane -L 5
bind -r C-Right resize-pane -R 5

# ---- 复制模式 ----
set -g mode-keys vi

# ---- 状态栏样式 ----
set -g status-left-length 100
set -g status-right-length 100
set -g status-left "[#S] "
set -g status-right "%Y-%m-%d %H:%M"

# ---- 插件管理 ----
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @plugin 'tmux-plugins/tmux-yank'
set -g @plugin 'tmux-plugins/tmux-prefix-highlight'

# 自动保存与恢复
set -g @continuum-save-interval '15'
set -g @continuum-restore 'on'

# 初始化 TPM
run '~/.tmux/plugins/tpm/tpm'
```

### 17.2 快捷键速查卡

#### Session

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b d` | 分离当前 session |
| `Ctrl+b $` | 重命名当前 session |
| `Ctrl+b s` | 列出所有 session（可切换） |
| `Ctrl+b (` | 切换到上一个 session |
| `Ctrl+b )` | 切换到下一个 session |

#### Window

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b c` | 新建 window |
| `Ctrl+b n` | 切换到下一个 window |
| `Ctrl+b p` | 切换到上一个 window |
| `Ctrl+b 0-9` | 切换到指定编号的 window |
| `Ctrl+b ,` | 重命名当前 window |
| `Ctrl+b f` | 在所有 window 中搜索文本 |
| `Ctrl+b w` | 列出所有 window（可交互选择） |
| `Ctrl+b &` | 关闭当前 window（需确认） |

#### Pane

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b %` | 垂直分割 pane |
| `Ctrl+b "` | 水平分割 pane |
| `Ctrl+b 方向键` | 切换焦点到对应方向的 pane |
| `Ctrl+b o` | 按顺序切换 pane |
| `Ctrl+b q` | 显示 pane 编号（按编号可快速切换） |
| `Ctrl+b 空格` | 循环切换 pane 布局 |
| `Ctrl+b z` | 放大/还原当前 pane |
| `Ctrl+b x` | 关闭当前 pane（需确认） |
| `Ctrl+b !` | 将当前 pane 变成独立 window |

#### 其他

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b [` | 进入复制模式 |
| `Ctrl+b ]` | 粘贴 |
| `Ctrl+b :` | 进入命令行模式 |
| `Ctrl+b ?` | 显示所有快捷键列表 |
| `Ctrl+b t` | 显示时钟 |

#### 插件快捷键

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+b I` | 安装插件（大写 I） |
| `Ctrl+b U` | 更新插件 |
| `Ctrl+b Ctrl+s` | 保存 session 状态（resurrect） |
| `Ctrl+b Ctrl+r` | 恢复 session 状态（resurrect） |

### 17.3 术语对照

| 英文 | 中文 | 说明 |
|------|------|------|
| server | 服务器 | 后台管理所有 session 的进程 |
| session | 会话 | 独立的工作环境，可持久化 |
| window | 窗口 | 类似浏览器标签页 |
| pane | 面板 | 窗口内分割的子终端 |
| attach | 连接 | 连接到已存在的 session |
| detach | 分离 | 断开 client 但不终止 session |
| prefix key | 前缀键 | 快捷键的前导键（默认 Ctrl+b） |
| copy mode | 复制模式 | 浏览历史输出和复制文本的模式 |
| command mode | 命令行模式 | 输入 tmux 命令的模式 |
| TPM | Tmux 插件管理器 | Tmux Plugin Manager |
| SIXEL | 图像格式 | 终端中显示位图图像的编码标准（3.4+） |
| OSC 52 | 转义序列 | 用于访问系统剪贴板的终端协议（3.6+） |
| Mode 2031 | 主题模式 | 自动报告终端明/暗主题的协议（3.6+） |
