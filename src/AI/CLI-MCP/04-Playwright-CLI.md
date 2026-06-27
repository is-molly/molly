---
order: 4
date: 2026-06-27
---

> 上游：[microsoft/playwright-cli](https://github.com/microsoft/playwright-cli)（npm **`@playwright/cli`**）。Skill 内容来自仓库 `skills/playwright-cli/SKILL.md` 与 `references/`。Playwright MCP 仅作备选，见本章末尾。

# Playwright CLI

---

## 是什么

<a id="是什么"></a>

**Playwright CLI**（npm **`@playwright/cli`**，命令名 `playwright-cli`）是 Microsoft 给 **coding agent** 用的浏览器自动化**终端工具**。它驱动真实 Chromium/Firefox/WebKit，在终端输出 **Snapshot（YAML 无障碍树 + ref）**，Agent 用 `click e15` 这类 ref 操作页面。

| | Playwright CLI（本章） | Playwright MCP | Playwright Test |
|--|------------------------|----------------|-----------------|
| 面向 | Agent 会话里**临时验 UI** | 同上，走 MCP tool | 项目 **CI/E2E 测试套件** |
| 进 context | 单次命令输出 | 常驻 tool schema + 返回体 | 一般不直接给 Agent |
| 本书 | **默认** | 章末备选 | 不写（用项目自己的 `npx playwright test`） |

**会话里长什么样：** 「用 playwright-cli 打开 todomvc 演示站，加一条 todo 并截图」→ Agent 跑 `playwright-cli open …` → `snapshot` 拿 ref → `fill` / `click` → `screenshot`。页面一变 ref 会变，需再 snapshot。

上游 README 强调：相对 MCP，CLI 对 coding agent 更 **token-efficient**——不必常驻二十多个 `browser_*` tool 定义。

Skill 来自仓库 `skills/playwright-cli/SKILL.md`；全局安装：`cd ~ && playwright-cli install --skills`（见 [安装](#安装与-skill-落盘)）。

---

<a id="安装与-skill-落盘"></a>

## 安装与 Skill 落盘

### 要求

- Node.js ≥ 18

### 推荐：全局 Skill（默认，与 ctx7 的 `~/.claude/skills/` 一致）

上游 **`playwright-cli install --skills` 没有 `--global` 开关**。Skill 总是写到 **当前工作目录** 下的 `.claude/skills/playwright-cli/`（见上游 `skillCheck.js`）。要在 **Claude Code 全局** 生效，必须先 **`cd ~`** 再 install：

```bash
npm install -g @playwright/cli@latest
cd ~ && playwright-cli install --skills
```

装完路径应为 **`~/.claude/skills/playwright-cli/`**（任意项目开 Claude Code 都能用）。终端大致：

```text
✅ Workspace initialized at `/Users/you`.
✅ Skills installed to `.claude/skills/playwright-cli`.
… FFmpeg … / Found chrome …
```

说明：`Workspace initialized` 会在 **当前 cwd**（这里是 `$HOME`）留下 Playwright CLI 的 workspace 配置（如 `~/.playwright/`），与 Skill 目录是两回事。

**CC Switch 用户（可选）：**

```bash
cc-switch skills scan-unmanaged -a claude
cc-switch skills import-from-apps ~/.claude/skills/playwright-cli -a claude
cc-switch skills sync
```

`import-from-apps` **必须带目录**；裸跑会报 `Please provide at least one directory`。

### 备选：仅当前仓库（团队要把 Skill 进 git 时）

在**项目根**执行 `playwright-cli install --skills` → Skill 落在 **`<repo>/.claude/skills/playwright-cli/`**，只有在这个仓库里开 Claude Code 才加载。**不要**在文档默认路径里从项目根直接跑，除非你就是想项目级安装。

### 误装到项目里怎么改

<a id="误装到项目里怎么改"></a>

若已在某仓库（如 molly）里跑过 install，可先删项目内 Skill，再按上面全局装一遍：

```bash
# 在误装的仓库里
rm -rf .claude/skills/playwright-cli
# 可选：若不需要项目级 playwright workspace，再删 .playwright/

cd ~ && playwright-cli install --skills
```

### 装完检查

| 目标 | 检查 |
|------|------|
| 全局 | `ls ~/.claude/skills/playwright-cli/SKILL.md`；任意目录开 Claude Code → `/skills` 有 **playwright-cli** |
| 项目 | 仅在该 repo 根下 `/skills` 可见 |

`install --skills` 路径规则（由 **cwd** 决定）：

| 布局 | 实际路径 |
|------|----------|
| `--skills` / `--skills=claude` | `<cwd>/.claude/skills/playwright-cli/` |
| `--skills=agents` | `<cwd>/.agents/skills/playwright-cli/` |

目录内：**`SKILL.md`** + **`references/`**（10 个子文档）。

### 与 CC Switch（5.8.x）

<a id="与-cc-switch"></a>

| Skill 位置 | CC Switch |
|------------|-----------|
| **全局** `~/.claude/skills/playwright-cli` | 可选 `import-from-apps` 上路径 → `sync` |
| **项目** `<repo>/.claude/skills/…` | 一般**不要** import；`scan-unmanaged` 也扫不到项目目录 |

### 无全局命令时

Skill 里写：先试 `npx --no-install playwright-cli --version`；有本地包则全程 `npx playwright-cli …`，否则 `npm install -g @playwright/cli@latest`。

### 不用 Skill、只靠 help

README 允许只装 CLI：

```text
Test the "add todo" flow on https://demo.playwright.dev/todomvc using playwright-cli.
Check playwright-cli --help for available commands.
```

有 Skill 时触发更稳；生产环境仍推荐 `install --skills`（等价 `playwright-cli install --skills=claude`；Copilot 等布局用 `--skills=agents`）。

### 与上游对齐

命令以 **`playwright-cli --help`** 为准（npm `@playwright/cli`）。下文按 2026-06 上游 README + `--help` 整理；发新版后先跑 help 再改 Skill 提示。

**全局选项**（所有子命令可用）：

| 选项 | 作用 |
|------|------|
| `-s=<session>` | 指定 session 名（默认 `default`） |
| `--json` | 整条输出包成 JSON |
| `--raw` | 只输出结果值，便于管道 / diff |
| `--config <path>` | 配置文件（默认 `.playwright/cli.config.json`） |
| `--help [command]` | 子命令帮助 |
| `--version` | 版本号 |

---

## 核心工作流：snapshot + ref

Playwright CLI 和 MCP 版最大差别在 **元素定位方式**：Agent 用 Bash 跑 `open` / `goto` 等命令 → CLI 驱动浏览器 → 终端输出里带 **Snapshot** 和 **ref**（如 `e15`）→ Agent 用 `click e15` 操作 → 页面变化后 **ref 会变**，需要再 `snapshot`，不要死磕旧 ref。

### 每次命令后的输出结构

`goto`、`click` 等命令执行后，终端通常包含：

```text
### Page
- Page URL: https://example.com/
- Page Title: Example Domain
### Snapshot
[Snapshot](.playwright-cli/page-2026-02-14T19-22-42-679Z.yml)
```

Snapshot 是 YAML 无障碍树片段；每个可交互节点带 **ref**（如 `e15`）。Agent **应用 ref 操作**，不要猜 CSS——除非 snapshot 不够用。

### 按需 snapshot

```bash
playwright-cli snapshot                          # 默认：时间戳文件名
playwright-cli snapshot --filename=after.yaml    # 固定文件名，便于 diff
playwright-cli snapshot "#main"                  # 只截某元素
playwright-cli snapshot --depth=4                # 限制深度，省 token
playwright-cli snapshot e34                      # 某 ref 子树
playwright-cli snapshot --boxes                  # 附带 [box=x,y,w,h]
```

### 三种 targeting 方式（优先级）

1. **ref**（默认）— `playwright-cli click e15`
2. **CSS** — `playwright-cli click "#main > button.submit"`
3. **Playwright locator 字符串** — `playwright-cli click "getByRole('button', { name: 'Submit' })"`
4. **test id** — `playwright-cli click "getByTestId('submit-button')"`

ref 在页面 DOM 大变后会失效；下一步应再 `snapshot`，不要死磕旧 ref。

### `--raw` 与 `--json`

- **`--raw`**：去掉 Page 状态、生成代码、Snapshot 段落，只留命令结果值，方便管道：

```bash
playwright-cli --raw eval "JSON.stringify(performance.timing)" | jq '.loadEventEnd - .navigationStart'
playwright-cli --raw snapshot > before.yml
playwright-cli click e5
playwright-cli --raw snapshot > after.yml
diff before.yml after.yml
TOKEN=$(playwright-cli --raw cookie-get session_id)
```

- **`--json`**：整条回复包成 JSON（如 `playwright-cli list --json`）。

---

## 命令分组（与 `playwright-cli --help` 对齐）

### Core

```bash
playwright-cli open [url]              # 见下方 open 参数
playwright-cli attach [name]           # 附着已有浏览器，见 attach 参数
playwright-cli detach                  # 从 attach 会话脱离，外部浏览器继续跑
playwright-cli goto <url>
playwright-cli close
playwright-cli delete-data             # 删当前 session 的 profile 数据
playwright-cli type "search query"
playwright-cli click e3
playwright-cli dblclick e7
playwright-cli fill e5 "user@example.com"
playwright-cli fill e5 "user@example.com" --submit   # fill 后 Enter
playwright-cli drag e2 e8
playwright-cli drop e4 --path=./image.png
playwright-cli drop e4 --data="text/plain=hello world"
playwright-cli hover e4
playwright-cli select e9 "option-value"
playwright-cli upload ./document.pdf
playwright-cli check e12 / uncheck e12
playwright-cli snapshot [target]       # 见上文 snapshot 节
playwright-cli eval "document.title"
playwright-cli eval "el => el.textContent" e5
playwright-cli dialog-accept [text] / dialog-dismiss
playwright-cli resize 1920 1080
```

### Save as

```bash
playwright-cli screenshot [ref]
playwright-cli screenshot --filename=shot.png
playwright-cli pdf
playwright-cli pdf --filename=page.pdf
```

### Navigation / Keyboard / Mouse

```bash
playwright-cli go-back | go-forward | reload
playwright-cli press Enter | press ArrowDown
playwright-cli keydown Shift | keyup Shift
playwright-cli mousemove 150 300
playwright-cli mousedown [right] | mouseup [right]
playwright-cli mousewheel 0 100
```

### Tabs

```bash
playwright-cli tab-list
playwright-cli tab-new [url]
playwright-cli tab-close [index]
playwright-cli tab-select 0
```

### Storage / Network / DevTools

Storage：`state-save` / `state-load`、`cookie-*`、`localstorage-*`、`sessionstorage-*`（见 SKILL.md Storage 节）。

**Network — 抓包与 mock：**

```bash
playwright-cli requests                    # 列请求（带序号）
playwright-cli request 5                   # 单条详情
playwright-cli request-headers 5
playwright-cli request-body 5
playwright-cli response-headers 5
playwright-cli response-body 5             # 二进制会落盘并打印路径
playwright-cli network-state-set offline   # 或 online
playwright-cli route "**/*.jpg" --status=404
playwright-cli route "https://api.example.com/**" --body='{"mock": true}'
playwright-cli route-list
playwright-cli unroute "**/*.jpg"
```

**DevTools / 录屏 / 调试：**

```bash
playwright-cli console [min-level]         # error|warning|info|debug
playwright-cli run-code "async page => …"
playwright-cli run-code --filename=snippet.ts
playwright-cli tracing-start / tracing-stop
playwright-cli video-start [filename]
playwright-cli video-chapter "checkout flow"
playwright-cli video-show-actions          # 后续动作在录屏上标注
playwright-cli video-hide-actions
playwright-cli video-stop
playwright-cli show                        # 可视化 dashboard（见下节 Monitoring）
playwright-cli show --annotate             # UI 评审：用户画框标注
playwright-cli pause-at <location>         # 测试调试：跑到断点暂停
playwright-cli resume
playwright-cli step-over
playwright-cli generate-locator e5
playwright-cli highlight e5 [--style=…]
playwright-cli highlight e5 --hide
playwright-cli highlight --hide            # 隐藏页上全部高亮
```

### Install / Session 管理

```bash
playwright-cli install                     # 初始化 workspace
playwright-cli install --skills            # 在 **cwd** 下写 .claude/skills/；全局须先 cd ~
playwright-cli install --skills=agents     # .agents/skills/
playwright-cli install-browser [browser]   # 安装浏览器二进制
playwright-cli list
playwright-cli close-all
playwright-cli kill-all                    # 强杀僵尸进程
```

子场景详见 Skill 内 `references/`：

| 文件 | 内容 |
|------|------|
| `playwright-tests.md` | 跑、调试 Playwright 测试 |
| `request-mocking.md` | 路由 mock |
| `running-code.md` | `run-code` |
| `session-management.md` | session / profile |
| `spec-driven-testing.md` | plan / generate / heal |
| `storage-state.md` | cookie、localStorage 持久化 |
| `test-generation.md` | 从操作生成测试 |
| `tracing.md` | trace 文件 |
| `video-recording.md` | 录屏 |
| `element-attributes.md` | `eval` 读 DOM 属性 |

---

## Session 与浏览器选项

默认 profile **在内存**，`close` 后 cookie 丢；`--persistent` / `--profile` 落盘。

### `open` 参数

```bash
playwright-cli open [url]
playwright-cli open --headed
playwright-cli open --browser=chrome|firefox|webkit|msedge
playwright-cli open --persistent
playwright-cli open --profile=/path/to/profile
playwright-cli open --config=my-config.json
```

### `attach` 参数

```bash
playwright-cli attach --extension=chrome      # Playwright MCP Bridge 扩展
playwright-cli attach --cdp=chrome            # 本机 Chrome/Edge channel
playwright-cli attach --cdp=http://localhost:9222
playwright-cli attach --endpoint=<playwright-server-url>
playwright-cli detach                           # 脱离 attach，浏览器不关
```

工作区固定 session（README）：

```bash
PLAYWRIGHT_CLI_SESSION=todo-app claude .
```

有界面调试与 **Monitoring dashboard**：

```bash
playwright-cli open https://demo.playwright.dev/todomvc/ --headed
playwright-cli show                     # 网格预览各 session + 远程接管；Esc 释放输入
```

上游 README：**Session grid** 按 workspace 分组 live screencast；点进 **Session detail** 可 back/forward/reload、地址栏导航、点击 viewport 接管键鼠。

```bash
playwright-cli -s=example open https://example.com --persistent
playwright-cli -s=example click e6
playwright-cli -s=example close
playwright-cli -s=example delete-data
```

---

## 配置文件

默认读 **`.playwright/cli.config.json`**（也可用 `--config path`）。

主要字段（README schema 摘要）：

| 字段 | 作用 |
|------|------|
| `browser.browserName` | `chromium` / `firefox` / `webkit` |
| `browser.isolated` | 内存 profile，不落盘 |
| `browser.userDataDir` | 持久化目录 |
| `browser.launchOptions` | `headless`、`channel`、`executablePath` 等 |
| `browser.contextOptions` | `viewport` 等 |
| `browser.cdpEndpoint` / `cdpHeaders` / `cdpTimeout` | 连已有 Chromium |
| `browser.remoteEndpoint` | 连 Playwright Server |
| `browser.initPage` / `initScript` | 每个 page 加载前注入脚本 |
| `outputDir` / `outputMode` | 输出目录；`stdout` 或 `file` |
| `console.level` | 控制台消息最低级别 |
| `network.allowedOrigins` / `blockedOrigins` | 请求 origin 过滤 |
| `timeouts.action` / `navigation` | 默认 5000ms / 60000ms |
| `testIdAttribute` | 默认 `data-testid` |
| `codegen` | `typescript` 或 `none` |
| `allowUnrestrictedFileAccess` | 是否允许 workspace 外文件 upload |

团队可在仓库提交一份 config，agent 不用每次写 `--headed`。

---

## Windows：URL 里的 `&`

`cmd.exe` / PowerShell 会把 `&` 当命令分隔符，query 会被截断：

```batch
playwright-cli goto "https://example.com/?a=1^&b=2"
```

```powershell
playwright-cli --% goto "https://example.com/?a=1&b=2"
```

---

## 完整示例（来自上游 SKILL.md）

### TodoMVC 流程

```bash
playwright-cli open https://demo.playwright.dev/todomvc/ --headed
playwright-cli type "Buy groceries"
playwright-cli press Enter
playwright-cli type "Water flowers"
playwright-cli press Enter
playwright-cli snapshot
# 从 snapshot 找 checkbox ref
playwright-cli check e21
playwright-cli check e35
playwright-cli screenshot
playwright-cli close
```

### 表单

```bash
playwright-cli open https://example.com/form
playwright-cli snapshot
playwright-cli fill e1 "user@example.com"
playwright-cli fill e2 "password123"
playwright-cli click e3
playwright-cli snapshot
playwright-cli close
```

### 多标签 + trace

```bash
playwright-cli open https://example.com
playwright-cli tab-new https://example.com/other
playwright-cli tab-list
playwright-cli tab-select 0
playwright-cli tracing-start
playwright-cli click e4
playwright-cli tracing-stop
playwright-cli close
```

### UI 评审（`show --annotate`）

用户说「帮我看下这个页面设计」时，Skill 要求开 annotate 模式：用户在 live 页画框写备注，agent 收到标注截图、区域 snapshot 和文字反馈。

```bash
playwright-cli open https://example.com
playwright-cli show --annotate
```

---

## 和 MCP 版对比（默认用 CLI）

| | Playwright CLI（默认） | Playwright MCP（备选） |
|--|------------------------|-------------------------|
| 包 | `@playwright/cli` | `@playwright/mcp` |
| 安装 | **`cd ~ && playwright-cli install --skills`** → `~/.claude/skills/` | 见下节 |
| 模型侧 | Bash；输出里带 snapshot/ref | tools：`browser_navigate`、`browser_snapshot` 等 |
| Context 成本 | 按命令增量 | 常驻多 tool + 树 schema |
| 适用 | 改代码、E2E、快验 UI、脚本化 diff | 极少数：Agent 不能 Bash、又要长时间探页 |
| Skill | 自带 `playwright-cli` + references | 通常无等价 Skill |

**不要** CLI 和 MCP 同时常开。

---

<a id="playwright-mcp-fallback"></a>

## 备选：Playwright MCP（一般不必装）

**默认不要装。** 上游 [playwright-mcp](https://github.com/microsoft/playwright-mcp) 面向需要 **MCP tool 通道** 的 Host；Claude Code 有 Bash + 本章 Skill 时，CLI 更省 token。

### 何时才考虑 MCP

- 会话 Host **没有 Bash**（某些 IDE 插件场景）
- 你明确要 Cursor 内置 browser MCP 那套 tool 名，且 **不用** `playwright-cli`
- 已试过 CLI + Skill，仍无法满足（少见）

### 安装（与 CLI 互斥）

先确认未装 `playwright-cli` Skill，或 `/mcp` 里 disable 冲突项：

```bash
claude mcp add playwright npx @playwright/mcp@latest
# 或 CC Switch：Remote / stdio 按上游 README 填
```

模型侧是一组 `browser_*` tool（navigate、snapshot、click 等），不是 `playwright-cli` 命令。和本章 **ref/snapshot 工作流不同**——按 MCP 的 accessibility tree + tool schema 操作。

### 切回 CLI

```bash
claude mcp remove playwright   # 或 CC Switch 禁用
cd ~ && playwright-cli install --skills
cc-switch skills import-from-apps ~/.claude/skills/playwright-cli -a claude   # 若用 CC Switch
cc-switch skills sync
```

---

## 常见问题

| 现象 | 处理 |
|------|------|
| `command not found` | 确认全局 PATH；或 `npx @playwright/cli@latest …` |
| click 报 ref 不存在 | 页面变了，重新 `snapshot` |
| headless 看不清 | `--headed` 或 `playwright-cli show` |
| 和 Playwright MCP 抢浏览器 | 只留一种；见上表 |
| Skill 未触发 | `/skills` 看 `playwright-cli` Enabled；路径是否在 `.claude/skills/playwright-cli/` |
