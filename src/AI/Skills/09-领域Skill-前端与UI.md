---
order: 9
date: 2026-06-27
---

# 领域 Skill · 前端与 UI

> **章节类型：领域 Skill**（垂直场景：界面与体验）。与工作流方法论章（07/08）不同——这里讲的是「把 UI 做好」的能力包，不是整条开发流水线。  
> **五方案组合 playbook** → [02 本章导航](02-选型与组合方案.md#本章导航)；**只装一个前端 Skill** → [§ 9.6](#_9-6-standalone)。

前端设计是 Skills 生态中安装量最大的类别。五个安装量最高的 Skill 中有四个是前端相关的。本章覆盖最重要的四个。

## 9.1 frontend-design（Anthropic 官方）

### 项目概述

- **来源**：[anthropics/skills](https://github.com/anthropics/skills) 仓库
- **安装量**：skills.sh 上高安装量 Skill 之一（具体数字随时间变化）
- **定位**：教 Claude 做出**有意图的**设计决策，而不是落入 AI 生成的千篇一律

这个 Skill 不教 Claude 怎么写 HTML/CSS——它教的是审美判断。它的核心问题是：为什么 AI 生成的界面看起来都一样？同样的卡片布局、同样的蓝色按钮、同样的通用 Hero Section。社区管这叫做"AI slop"（AI 泥浆）。

frontend-design 是对"AI slop"的最直接回应。

### 设计思维流程

官方 Skill 完整源码要求 Claude 在写代码之前先做以下判断：

**目的（Purpose）**：这个界面解决什么问题？谁在用？

**调性（Tone）**：从一堆极端风格中选择一个，然后执行到极致：

> "Pick an extreme: brutally minimal, maximalist chaos, retro-futuristic, organic/natural, luxury/refined, playful/toy-like, editorial/magazine, brutalist/raw, art deco/geometric, soft/pastel, industrial/utilitarian..."

**约束（Constraints）**：技术需求（框架、性能、可访问性）。

**差异化（Differentiation）**：什么会让这个设计**难忘**？那个让人记住的一个点是什么？

> "CRITICAL: Choose a clear conceptual direction and execute it with precision. Bold maximalism and refined minimalism both work - the key is intentionality, not intensity."

### 前端美学指南（全部来自源码原文）

**Typography（字体）**：

选择独特、有趣的字体。永远不要用 Inter、Roboto、Arial、system fonts。搭配一个独特的展示字体和一个精致的正文字体。

**Color & Theme（颜色与主题）**：

忠于一个统一的美学方向。用 CSS 变量保持一致性。主导色 + 锐利强调色 > 胆怯的均匀分布的调色板。

**Motion（动效）**：

用动画来制造效果和微交互。优先选择纯 CSS 方案（用于 HTML），React 中优先使用 Motion 库。聚焦高影响力时刻：一个精心编排的页面加载效果（带交错的 `animation-delay`）比零散的微交互带来更多愉悦感。

**Spatial Composition（空间构图）**：

非预期的布局。不对称。重叠。对角线流动。打破网格的元素。慷慨的留白**或**受控的密度。

**Backgrounds & Visual Details（背景与视觉细节）**：

创造氛围和深度，而不是默认纯色。根据整体美学添加情境效果和纹理。创造性形式如渐变网格、噪点纹理、几何图案、分层透明、戏剧性阴影、装饰性边框、自定义光标和颗粒叠加。

### 永远禁止的元素（源码原文）

> "NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character."

### 关键指令

> "Interpret creatively and make unexpected choices that feel genuinely designed for the context. No design should be the same. Vary between light and dark themes, different fonts, different aesthetics. NEVER converge on common choices (Space Grotesk, for example) across generations."

> "Remember: Claude is capable of extraordinary creative work. Don't hold back, show what can truly be created when thinking outside the box and committing fully to a distinctive vision."

### 安装

> 多路径规则 → [05 安装手册 § 5.0](05-安装方式参考手册.md)

- **推荐（CC Switch 用户）**：`install owner/repo:目录名` + `sync`（**无** `--skill`；首次须 `repos add`）。
- **备选（终端 / 无 CC Switch）**：`npx skills add --skill …` 装到 Claude Code 全局。
- **不要两条都跑**。若已通过 Anthropic 官方 Plugin 包装了同名 Skill，不必再 npx 装一份。

```bash
# 推荐（CC Switch）
cc-switch skills install anthropics/skills:frontend-design && cc-switch skills sync
```

```bash
# 备选（npx）
npx skills add anthropics/skills --skill frontend-design -g -a claude-code -y
```

```
/skills
/doctor
```

## 9.2 UI/UX Pro Max — 全套设计基础设施

### 项目概述

- **仓库**：[nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill)
- **Stars**：36K+
- **协议**：MIT
- **定位**：不止是"设计数据库"——是一套完整的设计基础设施，包含 7 个子 Skill、Python 数据引擎、CLI 安装器、30+ 内嵌字体文件

UI/UX Pro Max 的实际规模远超大多数文章的描述。它不是给 Claude 一本设计书——而是给 Claude 装备了一个设计工作室。

### 7 个子 Skill 完整结构

实际仓库中的 `.claude/skills/` 目录包含 7 个独立的子 Skill：

**1. `ui-ux-pro-max` — 主设计引擎**

核心数据文件（`data/` 目录）：

| 数据文件 | 内容 |
|----------|------|
| `styles.csv` | 67 种 UI 风格定义 |
| `colors.csv` | 161 个色板 |
| `typography.csv` | 57 种字体配对 |
| `ux-guidelines.csv` | 99 条 UX 指南 |
| `charts.csv` | 25 种图表类型 |
| `landing.csv` | 24 种落地页布局模式 |
| `products.csv` | 161 个产品类别匹配规则 |
| `ui-reasoning.csv` | 161 条推理规则 |
| `stacks/*.csv` | 16 个框架的专属数据（React / Vue / Svelte / Next.js / Nuxt / Angular / Flutter / React Native / SwiftUI / Laravel / Astro / shadcn / Three.js / Jetpack Compose / HTML-Tailwind） |

Python 引擎（`scripts/` 目录）：
- `core.py` — 核心推理引擎
- `design_system.py` — Design System Generator（v2.0 旗舰功能）
- `search.py` — 跨维度搜索

**2. `design` — 综合设计生成**

子模块：
- `data/cip/` — 设计交付物（deliverables、industries、mockup-contexts、styles 四个 CSV）
- `data/icon/` — 图标设计数据
- `data/logo/` — Logo 设计（colors、industries、styles 三个 CSV + Python 搜索/生成引擎）
- `references/` — 15 个参考文档（cip-design、logo-color-psychology、slides-strategies 等）
- `scripts/cip/` — 设计交付物的 Python 生成引擎

**3. `design-system` — Design Token 系统**

- `data/` 目录含 8 个 CSV（幻灯片背景、图表、颜色逻辑、文案、布局逻辑、布局、策略、字体）
- `references/` — 7 个文档（component-specs、token-architecture、tailwind-integration 等）
- `scripts/` — 9 个脚本（token 生成器、验证器、HTML token 验证器、幻灯片搜索等）
- `templates/design-tokens-starter.json` — Token 起始模板

**4. `brand` — 品牌管理系统**

- `references/` — 11 个品牌指南文档（approval-checklist、color-palette-management、typography-specifications、voice-framework 等）
- `scripts/` — 4 个可执行脚本（含颜色提取器 cjs、品牌上下文注入器、品牌到 token 同步器）
- `templates/brand-guidelines-starter.md`

**5. `ui-styling` — UI 样式与组件**

- **内嵌 30+ 字体文件**（`canvas-fonts/` 目录）：Arsenal SC、Big Shoulders、Boldonse、Bricolage Grotesque、Crimson Pro、DM Mono、Erica One、Geist Mono、Gloock、IBM Plex Mono/Serif、Instrument Sans/Serif、Italiana、JetBrains Mono、Jura、Libre Baskerville、Lora、National Park、Nothing You Could Do、Outfit、Pixelify Sans、Poiret One、Red Hat Mono、Silkscreen、Smooch Sans、Tektur、Work Sans、Young Serif
- `references/` — 6 个文档（shadcn-accessibility、shadcn-components、shadcn-theming、tailwind-customization 等）
- `scripts/` — shadcn_add.py（shadcn 组件安装器）、tailwind_config_gen.py（Tailwind 配置生成器）

**6. `slides` — 幻灯片生成**

- `references/` — 5 个文档（copywriting-formulas、create、html-template、layout-patterns、slide-strategies）

**7. `banner-design` — 横幅设计**

- `references/banner-sizes-and-styles.md`

### CLI 安装器

UI/UX Pro Max 提供了一个独立的 npm CLI 工具（`ui-ux-pro-max-cli`），支持 18 个平台的模板化安装：

```
claude, cursor, codex, gemini, windsurf, copilot, continue, 
trae, qoder, roocode, kilocode, kiro, opencode, codebuddy, 
agent, augment, droid, warp
```

安装：`npm install -g ui-ux-pro-max-cli` 或 `npx ui-ux-pro-max-cli init`

### v2.0 核心能力：Design System Generator

这是 UI/UX Pro Max 的旗舰功能。你输入项目需求，它输出完整的设计系统——不再是手动从数据库中挑选，而是 AI 推理引擎自动分析后推荐。

**工作流程**：

```
用户需求："为我的美容水疗中心建一个落地页"
    ↓
并行搜索 5 个领域：
  · 产品类型匹配（161 个类别）
  · 风格推荐（67 种风格）
  · 色板选择（161 个调色板）
  · 落地页布局模式（24 种模式）
  · 字体配对（57 种组合）
    ↓
推理引擎合成：
  · 模式：Hero-Centric + Social Proof
  · 风格：Soft UI Evolution
  · 色板：Primary #E8B4B8 / Secondary #A8D5BA / CTA #D4AF37
  · 字体：Cormorant Garamond / Montserrat
  · 避免模式：霓虹色、锐利动画、暗色模式
    ↓
预交付检查清单：
  [ ] 没有 emoji 作为图标（用 SVG: Heroicons/Lucide）
  [ ] 所有可点击元素有 cursor-pointer
  [ ] Hover 状态平滑过渡（150-300ms）
  [ ] 浅色模式文字对比度 ≥ 4.5:1
  [ ] Focus 状态可见
  [ ] 尊重 prefers-reduced-motion
  [ ] 响应式断点：375px / 768px / 1024px / 1440px
```

### 安装

> 多路径怎么选 → [05 安装手册 § 5.0](05-安装方式参考手册.md)

- **推荐 A（`uipro init`）**：上游标准流程，写入 `.claude/skills/ui-ux-pro-max/` 及数据脚本、Python 引擎；Claude Code 本地开发首选。
- **备选 B（Claude Code Plugin）**：习惯所有能力走 `/plugin`、要 `/plugin update` 一键更新时用。
- **备选 C（npx / CC Switch 仅主 Skill）**：用 CC Switch 管 Skill，或不想装全局 `ui-ux-pro-max-cli` 时。
- **A / B / C 只选一条**；只装主 Skill `ui-ux-pro-max`，**不要**把 7 个子 Skill 全装进描述预算。

```bash
# A. uipro CLI（推荐）
npm install -g ui-ux-pro-max-cli
uipro init --ai claude
```

```
/skills
/doctor
```

```
# B. Plugin（备选）
/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill
/plugin install ui-ux-pro-max@ui-ux-pro-max-skill
/reload-plugins
/skills
```

```bash
# C.1 推荐（CC Switch）
cc-switch skills install nextlevelbuilder/ui-ux-pro-max-skill:ui-ux-pro-max && cc-switch skills sync
```

```bash
# C.2 备选（npx）
npx skills add nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max -g -a claude-code -y
```

### 与 frontend-design 的定位差异

| 维度 | frontend-design | UI/UX Pro Max |
|------|-----------------|---------------|
| 定位 | 审美品味指南 | 设计数据库 |
| 核心问题 | "如何不做得像 AI 生成的" | "库里有最适合你项目的设计参数" |
| 输出 | 设计理念指导 | 精确的色板代码、字体名称、布局模式 |
| 适合 | 创意方向和审美判断 | 快速系统化设计决策 |

两者可以组合使用——frontend-design 负责"想法和方向"，Pro Max 负责"精确参数"。

> **描述预算提醒**：UI/UX Pro Max 含 7 个子 Skill，Addy Osmani 含 24 个——不建议全部安装。按项目需求选 2-3 个即可，否则 description budget 会被打爆。用 `/skills` 禁用不需要的子 Skill。

## 9.3 Vercel Labs Agent Skills — 工程性能优化

### 项目概述

- **仓库**：[vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)
- **Stars**：26K+
- **定位**：Vercel 10 年 React/Next.js 工程经验的 Skills 化

Vercel 把自己的工程团队积累了 10 年的 React 和 Next.js 开发经验，整理成了一套给 AI 看的 Skills。这不是"最佳实践清单"，而是**优先分级的工程铁律**。

### 完整 Skill 清单

该仓库实际包含 9 个 Skill：

| Skill | 说明 |
|-------|------|
| `react-best-practices` | React/Next.js 性能铁律（skills.sh 高安装量） |
| `web-design-guidelines` | 100+ 条 Web 设计规则 |
| `composition-patterns` | 复合组件模式 |
| `react-native-skills` | React Native 最佳实践 |
| `react-view-transitions` | View Transition API |
| `deploy-to-vercel` | Vercel 部署 |
| `vercel-cli-with-tokens` | Vercel CLI 操作 |
| `vercel-optimize` | Vercel 优化 |
| `writing-guidelines` | 写作规范 |

> 安装量数字来自 skills.sh 快照，会实时变化。`next-best-practices` 在 [vercel-labs/next-skills](https://github.com/vercel-labs/next-skills) 独立仓库。

### react-best-practices

40+ 条规则，按优先级分 4 级。可在 skills.sh 查看当前安装排名。

**CRITICAL（关键——必须首先解决）**

1. 消除请求瀑布流（Waterfall）

```typescript
// ❌ 默认写法（AI 经常这样写）
async function handleRequest(userId, skip) {
  const userData = await fetchUserData(userId); // 即使 skip=true 也傻等
  if (skip) return { skipped: true };
  return process(userData);
}

// ✅ 按需等待
async function handleRequest(userId, skip) {
  if (skip) return { skipped: true };        // 不需要数据？直接返回
  const userData = await fetchUserData(userId); // 真的需要了才请求
  return process(userData);
}
```

核心原则：**尽早启动 Promise，尽可能晚地 await**。

2. Barrel File 问题

```typescript
// ❌ 为了用两个组件，加载了整个 MUI 库
import { Button, TextField } from '@mui/material';

// ✅ 直接引用源文件（减少构建时间）
import Button from '@mui/material/Button';
import TextField from '@mui/material/TextField';
```

**HIGH（高优先级——服务端性能）**

3. Server Component 序列化陷阱

```tsx
// ❌ 传了整个 user 对象（50 个字段），客户端只用 1 个
async function Page() {
  const user = await fetchUser();
  return <Profile user={user} />;
}

// ✅ 只传客户端真正需要的字段
async function Page() {
  const user = await fetchUser();
  return <Profile name={user.name} />;
}
```

**MEDIUM（中等——客户端优化）**

4. 别用 useEffect 监听一切——多个组件想监听同一个全局事件时用 `useSWRSubscription` 或全局 Map 去重。

**LOW（低优先级——微优化）**

JS 语法级别的微小优化。

### web-design-guidelines

100+ 条 Web 设计规则，覆盖 accessibility / performance / UX。

覆盖类别：
- Accessibility（aria-labels、语义 HTML、键盘处理）
- Focus States（可见 focus、focus-visible 模式）
- Forms（autocomplete、验证、错误处理）
- Animation（prefers-reduced-motion、compositor-friendly transforms）
- Typography（弯引号、合适的 em-dash、字体加载策略）

### composition-patterns

复合组件模式完整指南。

### 安装

> 多路径规则 → [05 安装手册 § 5.0](05-安装方式参考手册.md)。仓库共 **9 个** Skill，**按需装 2–3 个**，不要 `--all`。

- **推荐（React/Next 最小 trio）**：下面三条 `npx` 或 CC Switch 等价命令；CC Switch 用户每条 `install` 后统一 `sync`。
- **备选**：React Native 等其它栈 → 先 `--list` 再按需 `--skill`。
- **不要 `--all`**。

```bash
# 推荐 — npx（React/Next trio）
npx skills add vercel-labs/agent-skills --skill react-best-practices -g -a claude-code -y
npx skills add vercel-labs/agent-skills --skill web-design-guidelines -g -a claude-code -y
npx skills add vercel-labs/agent-skills --skill composition-patterns -g -a claude-code -y
```

```bash
# 推荐 — CC Switch（同上 trio）
cc-switch skills install vercel-labs/agent-skills:react-best-practices
cc-switch skills install vercel-labs/agent-skills:web-design-guidelines
cc-switch skills install vercel-labs/agent-skills:composition-patterns
cc-switch skills sync
```

```bash
# 备选 — 其它栈示例
npx skills add vercel-labs/agent-skills --list
npx skills add vercel-labs/agent-skills --skill react-native-skills -g -a claude-code -y
```

```
/skills
/doctor
```

## 9.4 addyosmani/agent-skills — 完整的 SDLC Skill 集

- **仓库**：[addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)
- **作者**：Addy Osmani（Google Chrome 工程总监）
- **Stars**：34K+
- **实际 Skill 数**：24 个（远不止 frontend-ui-engineering）

实际仓库中的完整 Skill 列表：

| Skill | 职能 |
|-------|------|
| `frontend-ui-engineering` | 组件架构、设计系统、状态管理、响应式、WCAG 2.1 AA |
| `performance-optimization` | 性能优化（Core Web Vitals、bundle、渲染） |
| `code-review-and-quality` | 代码审查和质量保障 |
| `test-driven-development` | TDD 工作流 |
| `debugging-and-error-recovery` | 调试和错误恢复 |
| `security-and-hardening` | 安全加固 |
| `browser-testing-with-devtools` | 浏览器 DevTools 测试 |
| `code-simplification` | 代码简化 |
| `api-and-interface-design` | API 和接口设计 |
| `context-engineering` | 上下文工程 |
| `incremental-implementation` | 增量实现 |
| `spec-driven-development` | Spec 驱动开发 |
| `source-driven-development` | 源码驱动开发 |
| `doubt-driven-development` | 怀疑驱动开发 |
| `planning-and-task-breakdown` | 规划和任务拆分 |
| `shipping-and-launch` | 发布上线 |
| `git-workflow-and-versioning` | Git 工作流和版本管理 |
| `ci-cd-and-automation` | CI/CD 自动化 |
| `observability-and-instrumentation` | 可观测性和插桩 |
| `documentation-and-adrs` | 文档和 ADR |
| `deprecation-and-migration` | 废弃和迁移 |
| `idea-refine` | 想法精炼 |
| `interview-me` | 面试式需求澄清 |
| `using-agent-skills` | Agent Skills 使用指南 |

### 安装

> 多路径规则 → [05 安装手册 § 5.0](05-安装方式参考手册.md)。仓库 **24 个** Skill，**不要** `--all`。

- **推荐（前端/UI 只装 1 个）**：`frontend-ui-engineering`；CC Switch 或 npx **二选一**。
- **备选**：按需追加 `performance-optimization`、`code-review-and-quality` 等 → 先 `--list`。

```bash
# 推荐 — CC Switch
cc-switch skills install addyosmani/agent-skills:frontend-ui-engineering && cc-switch skills sync
```

```bash
# 备选 — npx
npx skills add addyosmani/agent-skills --skill frontend-ui-engineering -g -a claude-code -y
```

```bash
# 备选 — 追加第二个 Skill 示例
npx skills add addyosmani/agent-skills --list
npx skills add addyosmani/agent-skills --skill performance-optimization -g -a claude-code -y
```

```
/skills
/doctor
```

---

<a id="_9-6-standalone"></a>

## 9.6 不经 02 方案单独使用

> 只装 **一个** 前端 Skill、不走 [02 五方案组合](02-选型与组合方案.md#本章导航) 时，用本节起手式。组合 playbook（Superpowers × frontend-design 等）仍在 02。

### frontend-design 单独用

**适合**：已有自己的需求/计划流程，只要 UI 审美层。

```
/skills && /doctor

> Build [页面类型] for [领域]。
> Follow frontend-design: pick a bold tone first, show color/type/layout plan before code.
> Avoid AI slop (Inter, purple gradients, generic hero).
```

| 步骤 | 做什么 |
|------|--------|
| 1 | Prompt **点名** frontend-design + 领域/调性 |
| 2 | 等 **设计计划**（hex、字体、布局 ASCII、signature）并确认 |
| 3 | 确认后再写组件；修 UI 时重复 2→3 |

**示例**

```
你: Redesign the settings page — industrial/utilitarian, dark, monospace accents.
     frontend-design: plan before code.

Claude: [调性 + token 计划 …]
你: 确认
Claude: [SettingsPage.tsx …]
```

深潜 → [§ 9.1](#91-frontend-designanthropic-官方)。与 Superpowers 组合 → [02 方案 A 工作流](02-选型与组合方案.md#方案-a-工作流)。

### ui-ux-pro-max 单独用

**适合**：要 **MASTER.md / 设计系统 CSV**，不要 gstack Sprint。

```
/skills && /doctor

> Build a [product type] dashboard.
> Use ui-ux-pro-max workflow: search styles/colors/typography, persist design-system/MASTER.md.
```

README 五步（search → reasoning → domain → stack → design system）见 [02 方案 C 示例 B/C](02-选型与组合方案.md#方案-c-工作流)。跨会话读 MASTER → [02 示例 D](02-选型与组合方案.md#方案-c-工作流)。

**不要**一次装 Pro Max 全部 7 子 Skill；只装主 `ui-ux-pro-max`（见 § 9.2 安装）。

### Vercel / Addy 单体

| Skill | 单独用时怎么说 |
|-------|----------------|
| `react-best-practices` | 「Review this React component for Vercel best practices」 |
| `web-design-guidelines` | 「Audit this page against web-design-guidelines」 |
| `frontend-ui-engineering` | 「Apply frontend-ui-engineering to this layout refactor」 |

安装 → 各节 § 9.3 / § 9.4。组合见 [02 方案 C](02-选型与组合方案.md#方案-c-工作流)。

---

> **本章资料来源**：[anthropics/skills - frontend-design](https://github.com/anthropics/skills/blob/main/skills/frontend-design/SKILL.md)、[nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill)、[vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)、[addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)
