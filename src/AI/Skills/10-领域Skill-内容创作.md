---
order: 10
date: 2026-06-27
---

# 领域 Skill · 内容创作

> **章节类型：领域 Skill**（垂直场景：文章、配图、发布）。与工作流方法论章（07/08）不同。  
> **五方案组合** → [02 本章导航](02-选型与组合方案.md#本章导航)；**只装 baoyu 单体** → [§ 10.6](#_10-6-standalone)。

## 10.1 项目概述

baoyu-skills 把内容创作流程——图片、视觉、文档转换、多平台发布——打包为 **20+ 个**可组合 Skill（上游 README 表述；Plugin 内具体数量随版本变）。

- **仓库**：[JimLiu/baoyu-skills](https://github.com/JimLiu/baoyu-skills)
- **作者**：Jim Liu（[JimLiu](https://github.com/JimLiu)）
- **Stars**：22K+（2026-06）
- **Skill 数**：20+（建议按需安装，勿 bulk-install 全部）
- **协议**：MIT-0（发布到 ClawHub 的 Skill 使用）
- **特色**：npm 包 + Claude Code Plugin Marketplace 双重分发

baoyu-skills 证明了一件事：Claude Code 不只是编程工具——它可以成为内容创作者的全套生产流水线。

## 10.2 三层架构

baoyu-skills 按职责分为三层：

### 第一层：内容生成层

这是用户直接用的 Skill，负责产出具体的视觉内容：

| Skill | 职能 | 核心参数 |
|-------|------|----------|
| `baoyu-xhs-images` | 小红书卡片生成 | 12 风格 × 6 布局 |
| `baoyu-infographic` | 信息图 | 22 布局 × 17 风格 |
| `baoyu-diagram` | SVG 图表 | 流程图、时序图、架构图 |
| `baoyu-comic` | 知识漫画 | 6 画风 × 7 色调 × 6 格布局 |
| `baoyu-slide-deck` | 幻灯片 | 17 种预设风格，输出 .pptx / .pdf |
| `baoyu-cover-image` | 封面图 | 5 维定制：类型 × 色板 × 渲染 × 文字 × 情绪 |
| `baoyu-article-illustrator` | 文章配图 | 自动分析文章内容 → 推荐风格 → 生成 |
| `baoyu-md` | Markdown 格式化 | 标准化排版 |

### 第二层：AI 后端引擎层

提供图片生成的底层能力，被第一层的 Skill 调用：

| 引擎/提供商 | 说明 |
|-------------|------|
| OpenAI GPT Image 2 | OpenAI 最新图片模型 |
| Google Gemini (Imagen) | Google 图片生成 |
| Azure OpenAI | Azure 部署的 OpenAI |
| OpenRouter | 聚合多家模型 |
| DashScope (阿里通义万相) | 阿里云图片生成 |
| Z.AI GLM-Image | 智谱图片模型 |
| MiniMax | image-01 / image-01-live |
| Jimeng（即梦） | 字节跳动图片生成 |
| Seedream（豆包） | 字节跳动另一图片模型 |
| Replicate | 开源模型平台 |

一个命令切换 Provider，系统自动检测你配置了哪些 API Key，按优先级选择可用 Provider。如果你什么图片生成 API Key 都没有，`baoyu-danger-gemini-web` 可以通过浏览器 Cookie 连接 Gemini Web 免费生成。

### 第三层：工具链层

内容创作的辅助工具：

| Skill | 职能 |
|-------|------|
| `baoyu-format-markdown` | Markdown 格式化与排版 |
| `baoyu-markdown-to-html` | Markdown 转 HTML |
| `baoyu-compress-image` | 图片压缩 |
| `baoyu-electron-extract` | Electron 应用内容提取 |
| `baoyu-wechat-summary` | 微信公众号文章摘要生成 |
| `baoyu-youtube-transcript` | 下载 YouTube 字幕，支持 yt-dlp 回退 |
| `baoyu-url-to-markdown` | 网页转 Markdown（Chrome CDP 渲染 + 清洗，内置 baoyu-fetch 抓取引擎） |
| `baoyu-translate` | 翻译，保留原文风格和修辞 |
| `baoyu-post-to-wechat` | 发布到微信公众号 |
| `baoyu-post-to-x` | 发布到 X（Twitter） |
| `baoyu-post-to-weibo` | 发布到微博 |

发布 Skill 采用的是**半自动模式**：脚本把内容填入浏览器，你检查后手动点击发布。这是有意为之的安全机制——如果你需要全自动发布，需要自己对接 API 层。

## 10.3 逐个核心 Skill 详细教学

### 10.3.1 baoyu-xhs-images — 小红书卡片

**触发**：`/baoyu-xhs-images <文章路径或内容>`

**核心参数**：Style × Layout 二维组合系统。

**12 种风格（Style）预览**：

| 风格 | 视觉特征 |
|------|----------|
| `cute`（默认） | 可爱、圆润、明亮 |
| `fresh` | 清新、透亮、氧气感 |
| `warm` | 温暖、柔和、治愈 |
| `bold` | 强烈对比、视觉冲击 |
| `minimal` | 极简、大量留白 |
| `retro` | 怀旧、做旧质感 |
| `pop` | 波普艺术风格 |
| `notion` | Notion 风格的简洁排版 |
| `chalkboard` | 黑板风格 |
| `study-notes` | 学习笔记风格 |
| `screen-print` | 丝网印刷风格 |
| `sketch-notes` | 手绘笔记风格 |

**6 种布局（Layout）**：

| 布局 | 信息密度 | 适合场景 |
|------|----------|----------|
| `sparse` | 1-2 个要点/卡 | 封面、引言 |
| `balanced` | 3-4 个要点/卡 | 常规内容 |
| `dense` | 5-8 个要点/卡 | 知识卡片、速查表 |
| `list` | 4-7 条 | 清单、排行 |
| `comparison` | 2 个对比面 | 前后对比 |
| `timeline` | 时间线样式 | 步骤流程 |

**可选色板覆盖**：

| 色板 | 特征 |
|------|------|
| `macaron` | 马卡龙柔和色 |
| `warm` | 暖色调 |
| `neon` | 霓虹荧光色 |

**使用示例**：

```bash
# 自动选择风格和布局
/baoyu-xhs-images posts/ai-future/article.md

# 指定风格
/baoyu-xhs-images posts/ai-future/article.md --style notion

# 组合风格和布局
/baoyu-xhs-images posts/ai-future/article.md --style notion --layout list

# 覆盖色板
/baoyu-xhs-images posts/ai-future/article.md --style notion --palette macaron

# 直接输入内容
/baoyu-xhs-images 今日星座运势

# 非交互模式（跳过所有确认，用于定时任务）
/baoyu-xhs-images posts/ai-future/article.md --yes
```

### 10.3.2 baoyu-infographic — 信息图

**触发**：`/baoyu-infographic <内容>`

信息图是 baoyu-skills 中最复杂的 Skill，有两维度组合系统：信息结构（布局）× 视觉美学（风格）。

**22 种信息布局（Layout）**：

| 布局类型 | 适用内容 |
|----------|----------|
| `linear-progression` | 时间线、步骤流程 |
| `pyramid` | 层级关系、优先级 |
| `funnel` | 漏斗转化、筛选流程 |
| `circular-flow` | 循环流程、生态系统 |
| `iceberg` | 表面/深层对比 |
| `venn-diagram` | 交集关系 |
| `tree-branching` | 树形分类、决策树 |
| `fishbone` | 因果关系分析（鱼骨图）|
| `comparison-matrix` | 多维度对比 |
| `bento-grid` | 功能展示 |
| `dashboard` | 数据仪表盘 |
| `hub-spoke` | 中心辐射关系 |
| `bridge` | 两方连接/过渡 |
| `comic-strip` | 叙事性流程 |
| `story-mountain` | 故事起承转合 |
| `isometric-map` | 等距场景 |
| `jigsaw` | 拼图关系 |
| `periodic-table` | 分类目录 |
| `structural-breakdown` | 结构拆解 |
| `winding-roadmap` | 曲折路线图 |
| `binary-comparison` | 二元对立对比 |
| `hierarchical-layers` | 分层架构 |

**17 种视觉风格（Style）**（完整列表见 `references/styles/` 目录）：

| 风格 | 特征 |
|------|------|
| `hand-drawn-illustration` | 手绘插画 |
| `claymation` | 黏土动画质感 |
| `corporate-memphis` | 企业孟菲斯风格 |
| `cyberpunk-neon` | 赛博朋克霓虹 |
| `aged-academia` | 古老学术 |
| `bold-graphic` | 大胆图形 |
| `chalkboard` | 黑板风格 |
| `craft-handmade` | 手工质感 |
| `ikea-manual` | IKEA 说明书风格 |

（完整 17 种风格定义在 `references/styles/` 目录下各自的 `.md` 文件中）

### 10.3.3 baoyu-diagram — SVG 图表

**与其他生成类 Skill 的根本区别**：不依赖图片生成模型，Claude 直接写 SVG 代码。零 API 成本，完全可控。默认使用暗色主题配色方案。

**支持类型**：Architecture（架构图）、Flowchart（流程图）、Sequence（时序图）、Structural（类图/ER 图）、Mind Map（思维导图）、Timeline（时间线）、Illustrative（概念图）、State Machine（状态机）、Data Flow（数据流图）。

### 10.3.4 baoyu-comic — 知识漫画

**三维组合系统**：

| 维度 | 选项 |
|------|------|
| 画风（Art Style） | 6 种（chalk / ink-brush / ligne-claire / manga / minimalist / realistic） |
| 色调（Tone） | 7 种 |
| 格布局（Panel Layout） | 6 种 |

特殊预设：`ohmsha`（欧姆社学习漫画风格）、`wuxia`（武侠风格）、`shoujo-manga`（少女漫画特殊规则）。

### 10.3.5 baoyu-slide-deck — 幻灯片

从文章自动生成完整幻灯片。17 种预设风格，输出 `.pptx` 和 `.pdf`。

### 10.3.6 baoyu-image-gen — 图片生成引擎

**触发方式**：所有需要 AI 图片生成的 Skill 均调用此引擎（v2.5.0 起 `baoyu-imagine` 已合并至此，目录名仍为 `baoyu-image-gen`）。

**Provider 自动选择**：检测已配置的 API Key → 按优先级选择 → 无 Key 时 fallback 到 Gemini Web（浏览器 Cookie）。

```bash
/baoyu-image-gen --prompt "tech poster" --image poster.png --provider dashscope --ar 16:9
/baoyu-image-gen --batchfile batch.json --jobs 4
```

## 10.4 安装

> 多路径规则 → [05 安装手册 § 5.0](05-安装方式参考手册.md)

### 推荐（npx / CC Switch，按需装）

- **适合**：只要几个 `baoyu-*`、用 CC Switch、**不要**整包 20+ Skill 占满描述预算。
- **推荐（CC Switch 用户）**：`install owner/repo:<目录名>` + `sync`（**无** `--skill`）。
- **备选（终端 / 无 CC Switch）**：`npx skills add`；整仓或 `--skill` 单个。
- **不要与下方 Plugin 双装**。

```bash
# 推荐 — CC Switch（示例：只装小红书卡片）
cc-switch skills install JimLiu/baoyu-skills:baoyu-xhs-images
cc-switch skills sync
```

```bash
# 备选 — npx 只装一个
npx skills add JimLiu/baoyu-skills --skill baoyu-xhs-images -g -a claude-code -y

# 备选 — npx 整仓（装后在 /skills 里关掉不用的）
npx skills add JimLiu/baoyu-skills -g -a claude-code -y
```

**收尾**

```
/skills
/doctor
/baoyu-xhs-images --help   # 以已装 Skill 名为准
```

### 备选（整包 Plugin，仅 Claude Code）

- **适合**：要 `/plugin update` 一键更新整包；接受 20+ Skill 进 listing（须在 `/skills` 关掉不用的）。
- **不要与上方 npx / CC Switch 双装**。

```
/plugin marketplace add JimLiu/baoyu-skills
/plugin install baoyu-skills@baoyu-skills
/reload-plugins
/skills
/doctor
```

---

<a id="_10-6-standalone"></a>

## 10.6 不经 02 方案单独使用

> 只装 **一两个** baoyu Skill、不走 [02 五方案](02-选型与组合方案.md#本章导航) 时，用本节。内容创作 **没有** Superpowers 式强制链，靠 **Skill 名 / slash** 触发。

### 通用起手

```
/skills && /doctor
/<baoyu-skill-name> --help    # 以 listing 里 Enabled 的名为准
```

**原则**（上游 README）：按需装 1–3 个 Skill；在 `/skills` **关掉**不用的 baoyu 条目，避免 20+ 描述占满预算。

### 示例：只装 baoyu-xhs-images

**适合**：小红书卡片，不涉及 gstack / grill-me。

```bash
cc-switch skills install JimLiu/baoyu-skills:baoyu-xhs-images && cc-switch skills sync
```

```
/skills && /doctor

你: /baoyu-xhs-images
     主题：Claude Code Skills 入门
     风格：知识卡片，3 张图

Claude: [按 Skill 参数选 style × layout → 调 baoyu-image-gen 引擎 → 输出图片路径]

你: 第二张标题字再大一点

Claude: [在同 Skill 内迭代，不必重装]
```

| 步骤 | 做什么 |
|------|--------|
| 1 | 确认 API Key / Provider（见 § 10.2 引擎层；无 Key 时上游支持 Gemini Web fallback） |
| 2 | `/baoyu-xhs-images` 或自然语言点名 Skill + 主题/风格/张数 |
| 3 | 迭代 prompt；发布用 `baoyu-post-to-*` **另装**对应 Skill |

### 示例：文章配图 baoyu-article-illustrator

```
你: 为这篇 Markdown 文章生成 3 张配图，editorial 风格
     （匹配 baoyu-article-illustrator description）

Claude: [分析文章结构 → 推荐风格 → 生成插图文件]
```

### 与 02 方案的关系

| 你的情况 | 读哪里 |
|----------|--------|
| 只写内容、已有 coding 流程 | **本节** |
| 内容 + 产品 Sprint + UI | [02 方案 C](02-选型与组合方案.md#方案-c-工作流)（gstack，非 baoyu 专章） |
| Token 敏感 + 多步研究写作 | [02 方案 E](02-选型与组合方案.md#方案-e-工作流) + 本节 baoyu |

安装细节 → [§ 10.4](#104-安装)。生态表 → [11 生态索引](11-生态索引.md)。

## 10.5 更新

| 安装方式 | 更新 |
|----------|------|
| **Plugin** | `/plugin update baoyu-skills` 或 `/plugin` → Marketplaces → Update |
| **npx** | `npx skills update` 或重新 `npx skills add JimLiu/baoyu-skills ... -y` |
| **CC Switch** | Skills 面板的更新检测 / Update All（v3.13+） |

---

> **本章资料来源**：[JimLiu/baoyu-skills](https://github.com/JimLiu/baoyu-skills) 仓库 README、CHANGELOG 及各 Skill 目录结构
