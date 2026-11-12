---
order: 1
date: 2026-08-24
---
# pnpm

## 1. `pnpm` 是什么

### 1.1 定义

`pnpm` 是 Node.js 生态中的包管理器。其职责并不局限于“下载依赖”，而是围绕一个项目的依赖声明、版本解析、安装结果、锁定文件、工作区组织和日常命令入口，提供一套完整的管理模型。

在工程语义上，`pnpm` 与 `npm`、Yarn 一样，都属于包管理器。三者处理的是同一类问题：

- 读取项目依赖声明
- 解析版本范围并生成可安装结果
- 从 registry 获取包内容
- 在本地生成可运行的依赖目录
- 维护 lockfile
- 提供安装、更新、删除、执行脚本、发布等命令入口

因此，`pnpm` 不是脱离 Node.js 包管理体系的另一套东西，而是这一体系中的一种具体实现。

### 1.2 解决的问题

在实际工程中，包管理器需要同时处理三个层面的问题。

第一层是**依赖获取**。项目声明了哪些依赖、允许哪些版本范围、从哪个 registry 获取、安装时是否允许执行构建脚本，这些都属于包管理器的处理范围。

第二层是**依赖结果的可复现性**。同一份仓库在开发机、CI、Docker、离线环境中，是否能够得到一致的安装结果，决定了构建、测试、发布是否稳定。

第三层是**依赖图治理**。当仓库进入 Monorepo、基础依赖复用、版本统一、安全审查、私服接入等场景后，包管理器已经不只是一个“安装命令”，而是依赖治理的核心基础设施。

`pnpm` 的价值，主要体现在以下几个方向：

- 以更严格的依赖模型暴露声明问题
- 以更稳定的 lockfile 和目录结构支撑可复现安装
- 以原生 workspace 能力支撑 Monorepo
- 以集中化配置支撑版本统一、依赖修补、安装安全与工程治理

这些能力共同构成了 `pnpm` 的工程定位。其重点不是“命令写法不同”，而是“依赖管理模型不同”。

### 1.3 与 `npm`、Yarn 的关系

`pnpm` 与 `npm`、Yarn 的关系，不应理解为“谁替代谁”，而应理解为“面对同一问题域的不同实现”。

相同点主要有两类。

第一类是对象相同。三者都围绕 Node.js 项目的 `package.json`、依赖类型、registry、版本范围、lockfile、脚本命令等对象工作。

第二类是目标相同。三者都试图解决项目依赖如何声明、如何安装、如何锁定、如何更新、如何发布的问题。

差异不在“是否管理依赖”，而在“如何管理依赖”。差异主要落在以下方面：

- 依赖图如何解析
- 安装结果如何组织到磁盘
- 对未声明依赖、错误 peer、历史兼容行为的容忍度
- 对 Monorepo、版本统一、供应链安全的内建支持程度

这也是为什么迁移到 `pnpm` 后，经常会出现一种现象：旧项目在 `npm` 或 Yarn Classic 下能够工作，但切换到 `pnpm` 后暴露出依赖缺失、peer 冲突、工具链兼容性问题。这里暴露的并不一定是 `pnpm` 的缺陷，而更常见的是项目原本就存在依赖声明不完整、安装结果依赖偶然布局、工具链假设过强等问题。

### 1.4 `pnpm` 不是什么

为了避免后续章节理解偏移，有几类常见误解需要提前排除。

`pnpm` 不是新的 JavaScript 运行时。代码最终仍然运行在 Node.js、浏览器、Deno、Bun 等运行时上，`pnpm` 的职责是管理依赖和项目环境，而不是替代运行时本身。

`pnpm` 不是构建工具。Vite、Webpack、Rollup、esbuild、tsdown 负责打包、转译、产物生成；`pnpm` 负责把这些工具及其依赖正确安装进项目，并提供脚本执行入口。

`pnpm` 也不是完整的发布编排系统。虽然它支持 workspace 与发布相关流程，但多包版本发布、变更集管理、发版编排通常还需要配合 Changesets、Rush 等工具。

### 1.5 为什么需要单独学习 `pnpm`

在简单项目里，包管理器容易被当成一种习惯性工具：执行安装、执行脚本、升级依赖即可。进入中大型工程后，这种理解会迅速失效。

以下场景都会直接暴露对 `pnpm` 理解深度的不足：

- Monorepo 中多个包之间的本地依赖关系
- `peerDependencies` 导致的多实例和类型分裂
- lockfile 在 CI 中的可复现性约束
- 私服、离线环境、Docker 构建中的依赖获取链路
- 依赖版本统一、补丁修复、安全策略与供应链治理

因此，学习 `pnpm` 的重点不应停留在命令记忆，而应建立一张稳定的知识地图：

- 它管理哪些对象
- 它的安装与解析模型如何工作
- 它的配置分别属于哪个文件
- 它在真实工程里如何落地

本篇文档按照这一顺序展开。后续章节将分别处理基础对象、安装模型、workspace、peer 解析、治理配置、工程案例、离线环境与 CI/CD 等问题。

## 2. Node 包管理共同基础

这一章处理的不是 `pnpm` 独有能力，而是 Node.js 包管理的共有基础。无论使用 `npm`、`pnpm` 还是 Yarn，只要项目依赖 JavaScript 包生态，这些对象都必须先理解清楚。

### 2.1 包管理器实际管理的对象

Node.js 包管理器并不只管理“下载依赖”这一件事。更完整的对象划分至少包括以下几类：

- **依赖声明**
  - 项目需要哪些包、接受哪些版本范围、哪些依赖属于运行时、哪些属于开发时
- **解析结果**
  - 这一次安装最终确定了哪些精确版本、这些版本之间如何组成完整依赖图
- **包源与获取链路**
  - 包从哪个 registry 获取、是否经过私服代理、是否允许离线复用缓存
- **本地安装结果**
  - 解析完成后，依赖如何落到本地目录并被运行时和构建工具访问
- **脚本执行入口**
  - 项目如何调用构建、测试、检查、发布等命令

这几类对象里，最容易被混淆的是“声明”和“结果”。`package.json` 负责表达项目的需求，lockfile 负责记录本次解析后的结果，二者并不是同一种文件。

### 2.2 `package.json` 是声明文件，不是安装结果

`package.json` 是 Node.js 项目的清单文件。它描述的是项目希望外界如何理解这个包，而不是“这次安装已经变成了什么样”。

以依赖声明为例：

```json
{
  "dependencies": {
    "vue": "^3.5.0"
  }
}
```

这段配置表达了两层含义：

- 当前项目依赖 `vue`
- 接受满足 `^3.5.0` 的版本范围

但它并没有表达“当前机器已经安装了 `vue@3.5.40`”这一事实。原因在于 `package.json` 只声明需求，不记录解析后的精确结果。

这一点非常关键。很多“仓库没改，为什么安装出来不一样”的问题，根源都在于把 `package.json` 错当成了安装快照。实际工程里，`package.json` 更接近依赖需求说明书，而不是依赖结果清单。

除了依赖字段，`package.json` 还承载了包名、版本、脚本、入口、导出、运行环境约束等大量元数据。后续章节会分别处理其中与 `pnpm` 关系最紧密的部分。本章只保留最基础的认知：**`package.json` 是声明，不是结果。**

### 2.3 依赖类型的边界

Node.js 包管理最常见的依赖类型有四类。

#### `dependencies`

`dependencies` 表示运行时依赖。应用启动、服务运行、构建后产物执行时，需要直接用到这些包。

典型例子包括：

- 服务端项目的 Web 框架
- 前端项目的运行时框架
- 业务代码直接 `import` 的库

#### `devDependencies`

`devDependencies` 表示开发时依赖。它们通常参与开发、构建、测试、格式化、类型检查，但不是部署后产物在运行时直接依赖的业务库。

典型例子包括：

- TypeScript
- Vitest
- ESLint
- Vite

#### `peerDependencies`

`peerDependencies` 表示宿主依赖约束。其语义不是“当前包自己再安装一份”，而是“当前包要求使用者提供一份满足条件的依赖”。

这类声明常见于：

- 插件与宿主框架
- 组件库与上层运行时
- 对同一实例有共享要求的生态包

`peerDependencies` 是最容易引发误判的一类依赖。它和普通依赖在解析和安装上的差异非常大，后文会单独展开。

#### `optionalDependencies`

`optionalDependencies` 表示可选依赖。它们不是“完全不重要”，而是“当前包可以在缺失该依赖的情况下继续安装或降级运行”。

常见场景包括：

- 某些平台相关的原生模块
- 某些增强能力或加速能力
- 某些只在特定环境下才会启用的依赖

依赖类型的边界不应靠“感觉”判断，而应由包的职责决定。一个库如果在运行时必须存在，就不应放到 `devDependencies`。一个包如果要求与宿主共享同一份运行时实例，就不应写成普通 `dependencies`。

### 2.4 semver 范围与版本选择

Node.js 包生态默认使用 semver（Semantic Versioning，语义化版本）表达版本范围。一个版本号通常由 `major.minor.patch` 三段组成。

- `major`
  - 表示不兼容变更
- `minor`
  - 表示向后兼容的新功能
- `patch`
  - 表示向后兼容的问题修复

工程里最常见的声明方式有以下几类：

- `1.2.3`
  - 精确版本，只接受这一版
- `^1.2.3`
  - 接受同主版本下的更新版本
- `~1.2.3`
  - 接受同次版本下的更新版本
- `>=1.2.3 <2.0.0`
  - 显式区间范围
- `latest`
  - 跟随 dist-tag，而不是显式版本区间

版本范围的核心意义在于：**声明“可以接受什么”，而不是“这次一定安装什么”。**

因此，版本范围总是与 lockfile 联动理解。只有版本范围而没有锁定结果，安装结果就可能随时间、registry 状态、依赖树变化而发生变化。

### 2.5 registry 的角色

registry 是包获取的来源。包管理器在解析出依赖结果后，需要从 registry 获取包元数据、包内容以及对应的版本发布信息。

最常见的 registry 类型包括：

- npm 公共仓库
- 企业内部私服或代理仓库
- GitHub Packages
- GitLab Package Registry
- 基于 Verdaccio、Nexus、Artifactory 等产品搭建的 npm 仓库

registry 不只是一个下载地址，它还直接影响以下行为：

- 可获取哪些包和哪些版本
- 下载是否经过企业代理或镜像
- 认证信息如何配置
- 安装速度、稳定性与供应链边界如何控制

后文在私服、离线环境与 CI/CD 章节中会专门处理 `.npmrc`、镜像、认证与缓存策略。本章只需要建立一个基本认识：**依赖解析得到的是“要什么”，registry 决定的是“从哪里拿”。**

### 2.6 lockfile 是解析结果快照

lockfile 经常被误解为“安装时自动生成的附属文件”或“删了也能再装回来，所以不重要”。这类理解都不准确。

更准确的划分方式如下：

- `package.json`
  - 声明项目需要什么
- lockfile
  - 记录当前依赖图最终解析成了什么

lockfile 的核心价值在于，它把“版本范围”收敛成“精确结果”，并记录整张依赖图的解析状态。只有这样，同一份仓库才能在不同机器、不同时间、不同构建环境中尽可能得到一致的安装结果。

不同包管理器会生成不同名字的 lockfile。例如：

- `npm` 使用 `package-lock.json`
- Yarn 使用 `yarn.lock`
- `pnpm` 使用 `pnpm-lock.yaml`

虽然文件名不同，但它们处理的是同一类问题：**把依赖声明转化为可复现的解析结果。**

是否提交 lockfile，不是个人偏好问题，而是工程可复现性的基础决策。对于应用仓库、Monorepo、CI 驱动的交付链路，lockfile 通常都应纳入版本控制。

## 3. `pnpm` 和传统安装模型的差异

这一章讨论的不是命令差异，而是安装模型差异。真正决定工程体验的，通常不是 `install`、`add`、`update` 这些命令名，而是以下几个问题：

- 依赖图如何映射到磁盘
- 一个包在运行时到底能看到哪些依赖
- 未声明依赖为什么有时能工作、有时突然失效
- 为什么同一份代码在不同包管理器下会暴露出不同问题

### 3.1 差异不在命令层，而在安装模型

从 `package.json` 出发，所有包管理器都要先做一件事：把“依赖声明”解析成“可安装结果”。真正分歧发生在下一步，也就是：

> 解析结果如何落到本地目录，以及运行时最终能沿着哪些路径找到依赖。

这一步决定的不是“安装快一点还是慢一点”这么简单，而是整个依赖边界是否稳定、是否可预测。

传统平铺安装模型的核心取向是：尽量把依赖放到更上层，减少重复目录，提升兼容性，让更多包能在上层路径找到依赖。

`pnpm` 默认模型的核心取向是：尽量保留依赖边界，让包只消费自己声明过的依赖，减少“因为目录刚好摆成这样所以能工作”的偶然性。

因此，`pnpm` 与传统平铺模型的主要差异，不是“多了一个 store”，也不是“命令换了个名字”，而是：

- 对依赖边界的表达更严格
- 对安装结果的结构化程度更高
- 对历史兼容行为的默认容忍度更低

### 3.2 平铺安装为什么会制造隐式依赖

先看一个最常见的例子。

项目结构如下：

```json
{
  "name": "app",
  "dependencies": {
    "a": "1.0.0"
  }
}
```

包 `a` 自己又依赖 `lodash`：

```json
{
  "name": "a",
  "version": "1.0.0",
  "dependencies": {
    "lodash": "^4.17.21"
  }
}
```

如果应用代码里直接这样写：

```ts
import _ from 'lodash';
```

那么从依赖声明角度看，`app` 是有问题的。原因很简单：`app` 自己并没有声明 `lodash`。

问题在于，平铺安装模型下，这段代码有时不会立刻失败。目录可能长成这样：

```text
node_modules/
├─ a/
└─ lodash/
```

`lodash` 出现在顶层 `node_modules` 后，`app` 里的源码就有机会直接解析到它。于是工程中就会出现一种危险状态：

- 代码在当前机器上可以运行
- 依赖声明实际上不完整
- 一旦依赖树变化，行为就会失效

这种现象的本质，不是代码正确，而是目录布局替代码掩盖了错误。

### 3.3 `pnpm` 默认模型如何收紧依赖边界

`pnpm` 默认不会把整张依赖图简单压平成“谁都能从顶层顺手拿到”。它更强调的是：

> 包应该通过自己声明的依赖关系获得依赖，而不是通过顶层目录里的偶然结果获得依赖。

还是上面的例子。`app` 只声明了 `a`，没有声明 `lodash`。在 `pnpm` 的默认安装模型下，`app` 对 `lodash` 的访问更容易暴露为错误，因为目录布局不再优先照顾“未声明但碰巧能拿到”的场景。

结果就是：

- 依赖声明不完整的问题会更早暴露
- 项目更难依赖偶然的平铺结果继续运行
- 依赖边界与源码关系更一致

这也是很多团队第一次切到 `pnpm` 时最强烈的体感来源：不是“代码突然坏了”，而是原本被目录结构隐藏的问题被提前暴露了。

### 3.4 一个最小案例：未声明依赖为何在 `pnpm` 下更容易失败

假设仓库只有两个直接依赖：

```json
{
  "dependencies": {
    "vite": "^7.0.0",
    "eslint": "^9.0.0"
  }
}
```

某段脚本却直接 `import esbuild`，而项目并没有声明 `esbuild`。

在传统平铺模型里，只要 `vite` 或某个上游依赖把 `esbuild` 带进来，顶层目录就可能出现这一项：

```text
node_modules/
├─ eslint/
├─ esbuild/
└─ vite/
```

脚本因此能够工作，但前提是依赖树刚好收敛成这种结构。

一旦出现以下任一变化，行为就可能改变：

- `vite` 改了对子依赖的组织方式
- `esbuild` 不再被提升到顶层
- 另一个依赖引入了不同版本
- lockfile 刷新后，整体依赖图发生重排

`pnpm` 的默认模型对这种写法更不宽容，因此问题通常在安装、构建或运行期更早出现。这个“更早失败”不是副作用，而是严格依赖模型的结果。

### 3.5 从目录结果看差异

为了避免只停留在抽象描述，这里直接给出一个最小的目录预览。

仍然以一个直接依赖 `vite` 的项目为例，平铺模型常见的直观结果是：

```text
node_modules/
├─ vite/
├─ esbuild/
├─ picocolors/
├─ postcss/
└─ rollup/
```

从目录表面看，很多包都直接出现在顶层。这会形成一种强烈错觉：这些包似乎都“属于当前项目自己”。

`pnpm` 的默认结果不会这样表达整张依赖图。顶层更接近入口层，真实包实例会集中落到 `.pnpm` 下。下面的目录只用于说明结构，不表达某次真实安装的精确版本结果：

```text
node_modules/
├─ vite -> .pnpm/vite@<resolved_version>/node_modules/vite
└─ .pnpm/
   ├─ vite@<resolved_version>/node_modules/vite
   ├─ esbuild@<resolved_version>/node_modules/esbuild
   ├─ postcss@<resolved_version>/node_modules/postcss
   └─ rollup@<resolved_version>/node_modules/rollup
```

这里先只强调两点：

- 顶层 `node_modules` 不再承担“把所有依赖平铺展示出来”的职责
- 真实包实例目录被显式组织起来，入口层与实例层被区分

第 7 章会把 `.pnpm`、virtual store、symlink、hard link / clone / copy、Node.js 解析路径完整拆开。本章只需要先看到结论：**目录结构的表达方式不同，直接决定了依赖边界是否容易失真。**

### 3.6 为什么 `pnpm` 会更早暴露声明缺陷

`pnpm` 被认为“更严格”，通常不是因为它额外发明了一套新的语法约束，而是因为它不愿意默认替错误声明兜底。

最常见的暴露点有以下几类：

#### 未声明直接依赖

源码直接 `import` 某个包，但当前包自己的 `package.json` 没有声明它。平铺模型可能暂时掩盖这个问题，`pnpm` 默认模型更容易把问题暴露出来。

#### 错误的依赖类型

某个运行时依赖被错误地写进 `devDependencies`，或者一个本应使用 `peerDependencies` 的包被错误地写成普通 `dependencies`。这类问题在简单项目里也许不明显，但在多环境安装、构建拆分、Monorepo 共享时会迅速变成真实故障。

#### 对顶层目录结构的错误假设

部分旧工具、旧脚本或历史遗留代码会默认认为某些包一定存在于项目根 `node_modules`。只要安装模型不再保证这一点，问题就会暴露。

#### 对 peer 解析的错误理解

`peerDependencies` 本来就不是“当前包自己带一份”的模型。只要项目长期依赖平铺目录掩盖 peer 冲突，迁移到更严格的布局后，多实例、类型分裂、宿主不匹配等问题就会集中出现。

### 3.7 这类差异为什么在 Monorepo 里更重要

在单包仓库里，依赖边界问题有时还能靠人工习惯勉强维持。一旦进入 Monorepo，这种侥幸空间会迅速缩小。

原因有三点。

第一，包的数量变多后，隐式依赖会指数级扩散。一个包偷用未声明依赖，短期只影响自己；多个包都这么做，最终会让整张依赖图失去可预测性。

第二，共享依赖增多后，peer 问题不再是边角案例，而会变成结构性问题。组件库、插件、运行时框架、工具链插件都可能要求严格的宿主关系。

第三，CI、Docker、离线环境、不同操作系统的安装链路会放大目录偶然性。只依赖“本机当前能跑”几乎没有意义，依赖模型必须在不同环境下保持稳定。

这也是为什么 `pnpm` 的优势在大型工程里会更明显。真正重要的不是“它能不能装依赖”，而是：

- 依赖图是否更容易治理
- 问题是否更容易被定位到声明层
- 安装结果是否更适合被 CI 和团队协作复用

### 3.8 严格并不等于拒绝兼容

`pnpm` 默认模型强调严格依赖边界，但这并不意味着它完全拒绝兼容历史生态。

工程中确实存在两类现实问题：

- 一些第三方包的依赖声明本身就不完整
- 一些工具链默认假设依赖能从更上层目录被解析到

因此，`pnpm` 也提供了兼容与调节手段，用于在严格性和兼容性之间做工程取舍。这些能力包括布局兼容、依赖图修补、peer 解析调节等，但它们不应提前替代默认模型的理解。

顺序必须保持清楚：

1. 先理解默认依赖模型为什么更严格
2. 再理解哪些场景必须为兼容性做让步
3. 最后再决定是否启用兼容策略

相关配置会在第 10 章和第 11 章分别展开。

## 4. 安装、启用与版本管理

这一章处理的是包管理器版本基线，而不是单纯的安装命令。真正需要回答的问题有四个：

- 机器上如何获得 `pnpm` 命令
- 项目要求使用哪一版 `pnpm`
- 版本要求是固定值还是范围
- Node 与 `pnpm` 的兼容边界如何一起声明

如果这四个问题没有被拆开，工程里很容易出现以下结果：

- 开发机能安装，CI 却失败
- 团队成员使用的 `pnpm` 主版本不一致
- lockfile 变化与包管理器升级纠缠在一起
- 某些问题看起来像依赖错误，根因却是 Node / `pnpm` 基线漂移

### 4.1 先区分“本机安装”与“项目版本声明”

这是最容易混淆的一层。

**本机安装 `pnpm`** 解决的是命令可用性问题，即当前机器上是否存在可执行的 `pnpm`。

**项目版本声明** 解决的是治理问题，即进入这个仓库后，应该用哪一版 `pnpm` 和 Node 工作。

两者的职责完全不同：

- 本机安装决定“能不能执行”
- 项目声明决定“应该执行哪一版”

一个开发者通过某种方式把 `pnpm` 装到了机器上，并不代表仓库已经建立版本基线。团队治理真正依赖的，是仓库内的显式声明，而不是“大家自己装得差不多就行”。

### 4.2 获取 `pnpm` 命令只是入口，不是版本治理

机器获得 `pnpm` 的方式可以有多种，例如安装脚本、系统包管理器、独立发行方式，或者由已有工具链间接安装。无论入口如何变化，它们解决的都只是“当前环境如何拿到 `pnpm` 命令”。

工程上需要关心的是另一件事：

> 本机安装方式是否会让团队成员、CI、文档、脚本入口指向同一条版本基线。

如果仓库没有显式声明版本策略，那么以下差异都会放大问题：

- 某台机器全局安装的是 `pnpm 10`
- 某台机器已经切到 `pnpm 11`
- CI 镜像内预装的是另一版
- 某些人通过 `corepack` 调起包管理器，某些人直接执行全局命令

因此，获取命令这一步只能视为环境准备，不能替代项目版本治理。

### 4.3 `corepack` 的角色

`corepack` 是 Node.js 生态中的包管理器分发入口，用于根据项目声明调起对应的包管理器版本。它本身不是新的包管理器，也不负责依赖安装结果的解析。

在 `pnpm` 场景里，`corepack` 的价值主要体现在两点：

- 能根据项目里的包管理器声明下载并启用对应版本
- 能减少“每个人都手工管理全局包管理器版本”带来的偏差

从工程角度看，`corepack` 最重要的意义不是命令本身，而是它让仓库内的版本声明真正具备执行力。没有这层能力，`package.json` 里的版本信息更容易沦为静态备注。

### 4.4 `packageManager`：最基础的固定版本声明

`packageManager` 是写在 `package.json` 中的字段，用于声明项目希望使用的包管理器及其明确版本。

典型写法如下：

```json
{
  "packageManager": "pnpm@11.16.0"
}
```

这个字段回答的是：

> 当前仓库要求使用哪一个明确版本的包管理器。

它的特点非常清楚：

- 归属文件是 `package.json`
- 表达方式是单一固定版本，而不是范围
- 最适合建立一条明确、稳定、易理解的团队基线

在当前文档采用的 `pnpm 11` 主线下，还需要补一条行为边界：

- 当 `pnpm` 通过传统 `packageManager` 字段声明时，它本身的解析信息默认不会写入 `pnpm-lock.yaml`
- 这一点与 `devEngines.packageManager` 不同，后者会把解析结果记录到 lockfile 的相关区域

因此，`packageManager` 更适合表达“仓库要求使用哪一个明确版本”，而不是把包管理器版本解析过程本身纳入 lockfile 治理。

这种写法的工程价值很高，因为它减少了以下歧义：

- 到底应该用哪个主版本
- 小版本是否允许自由漂移
- CI 与本地是否应该严格一致

如果一个团队的诉求是“整仓统一到一条稳定版本线”，那么 `packageManager` 往往已经足够。

### 4.5 `devEngines.packageManager`：范围治理与解析复用

`devEngines.packageManager` 同样写在 `package.json` 中，但它处理的不是“固定单一版本”，而是“允许的版本范围以及不满足时的处理方式”。

典型写法如下：

```json
{
  "devEngines": {
    "packageManager": {
      "name": "pnpm",
      "version": ">=11.0.0 <12.0.0",
      "onFail": "download"
    }
  }
}
```

这一组字段比 `packageManager` 多出了三层能力。

第一，支持**版本范围**。  
项目不必把自己锁死在某一个精确版本，而可以允许一个受控区间。

第二，支持**失败处理策略**。  
`onFail` 用来表达当前环境不满足要求时应如何处理，而不是只留下一个静态版本字符串。

第三，支持**解析结果复用**。  
`pnpm` 会把它解析出来的包管理器依赖结果记录到 lockfile 相关区域，而不是每次都临时决定。

这使得 `devEngines.packageManager` 更接近“受控范围治理”，而 `packageManager` 更接近“固定版本锁定”。

### 4.6 `packageManager` 与 `devEngines.packageManager` 的取舍

这两个字段不是简单的新旧替代关系，而是两种不同的治理策略。

#### 选择 `packageManager` 的场景

适合以下目标：

- 团队希望锁定一条明确版本线
- 配置需要尽量简单直接
- 不希望把包管理器版本解析策略引入 lockfile 治理

这是大多数仓库都能接受的默认方案，也是成本最低的方案。

#### 选择 `devEngines.packageManager` 的场景

适合以下目标：

- 团队接受某个主版本区间，而不是单一精确值
- 希望不满足条件时有明确的处理动作
- 希望把包管理器版本解析结果纳入更严格的工程治理

这类方案更适合对工具链基线治理要求更高的仓库，但同时也会增加理解和维护成本。

如果压缩成一句话，两者的差别可以表达为：

- `packageManager`
  - 固定一个明确版本
- `devEngines.packageManager`
  - 声明一段允许范围，并控制解析与失败行为

### 4.7 `engines.node` 与 `engines.pnpm` 的职责

`engines` 同样属于 `package.json` 字段，但它的职责不同于前面两个字段。

例如当前 `daisy-ui` 项目的根 `package.json` 中写的是：

```json
{
  "engines": {
    "node": "^22.18.0 || ^24.12.0",
    "pnpm": ">=11.0.0"
  },
  "packageManager": "pnpm@11.16.0"
}
```

这里需要区分三件事：

- `packageManager`
  - 项目希望使用的明确包管理器版本
- `engines.pnpm`
  - 项目对 `pnpm` 的兼容范围声明
- `engines.node`
  - 项目对 Node 运行时的兼容范围声明

换句话说，`engines` 更接近兼容边界，而不是版本获取来源。

但这里有一个经常被忽略的行为边界，必须单独说清：

- 对本地开发环境而言，如果当前使用的 `pnpm` 版本不满足 `engines.pnpm`，`pnpm` 会直接报错失败
- 当当前包被别人作为依赖安装时，`engines` 默认更接近告警语义；若配置了 `engineStrict`，则可以提升为更严格的约束

这也是为什么它不能替代 `packageManager`：

- 只写 `engines.pnpm`，不能表达“仓库推荐的精确版本”
- 只写 `packageManager`，又不足以完整表达兼容区间

在团队治理里，这三者通常要一起看，而不是单独看某一个字段。

### 4.8 为什么 Node 基线必须和 `pnpm` 一起维护

包管理器版本策略不能脱离 Node 版本策略单独设计。原因非常直接：`pnpm` 本身运行在 Node 之上，不同主版本对 Node 的支持范围并不一致。

这意味着一次看似普通的 `pnpm` 升级，可能同时带来以下影响：

- 本地开发机 Node 最低版本变化
- CI 镜像基线变化
- 某些脚本运行时行为变化
- lockfile 与安装行为差异放大

因此，工程上不应把这件事拆成两个互不相干的问题：

- “Node 版本以后再说”
- “先把 `pnpm` 升上去”

更合理的处理方式是把它们视为同一套工具链基线的一部分，一起维护，一起评估，一起落到仓库配置和 CI 环境中。

### 4.9 `pnpm 11` 语境下的工程建议

本篇文档以 `pnpm 11` 为事实主线，因此版本治理建议也围绕这一主线展开。

对于大多数应用仓库和 Monorepo，建议优先遵循以下顺序：

1. 在 `package.json` 中显式声明 `packageManager`
2. 同时声明 `engines.node` 与 `engines.pnpm`
3. 保证 CI、本地文档、镜像基线与仓库声明一致
4. 将包管理器升级与 lockfile 大规模变动分开处理

只有当团队确实需要“范围治理 + 自动处理失败 + 更强的版本解析控制”时，再考虑把 `devEngines.packageManager` 引入主方案。

### 4.10 一个真实案例：`daisy-ui` 当前采用的策略

当前 `daisy-ui` 根 `package.json` 采用的是较稳健、也较易维护的组合：

```json
{
  "engines": {
    "node": "^22.18.0 || ^24.12.0",
    "pnpm": ">=11.0.0"
  },
  "packageManager": "pnpm@11.16.0"
}
```

这套组合的含义可以直接拆开：

- 通过 `packageManager` 锁定当前仓库推荐的 `pnpm` 精确版本
- 通过 `engines.node` 声明支持的 Node 基线
- 通过 `engines.pnpm` 声明最低兼容的 `pnpm` 范围

这种策略的优点在于：

- 团队成员看到配置即可知道推荐版本
- CI 更容易围绕精确版本统一
- 兼容边界和推荐版本没有混在一个字段里
- 文档、脚本、环境镜像更容易围绕同一条基线对齐

它没有使用 `devEngines.packageManager`，说明当前仓库的治理目标更偏向“固定稳定版本线”，而不是“允许版本区间内自由解析”。

### 4.11 本章结论

安装、启用与版本管理这件事，不应理解为“把命令跑通即可”。真正重要的是：

- 命令如何被获取
- 版本如何被声明
- 兼容边界如何被表达
- 本地、CI、文档、镜像是否指向同一条工具链基线

`package.json` 中的 `packageManager`、`devEngines.packageManager`、`engines.node`、`engines.pnpm` 分别承担不同职责。只有把它们的边界分清，后续关于 lockfile、CI、依赖治理和升级策略的讨论才不会混乱。

## 5. `package.json` 与依赖声明模型

这一章处理的是依赖建模，而不是安装结果。需要回答的核心问题只有一个：

> 当前包与某个依赖之间，到底是什么关系。

`package.json` 中这些字段看上去都在“写依赖”，但它们表达的不是同一种语义：

- `dependencies`
  - 当前包运行时直接需要什么
- `devDependencies`
  - 当前包在开发、构建、测试、类型检查阶段需要什么
- `peerDependencies`
  - 当前包要求宿主提供什么依赖环境
- `optionalDependencies`
  - 当前包在缺失某个依赖时，是否允许降级安装
- `peerDependenciesMeta`
  - 当前包如何补充 peer 依赖的附加语义
- `dependenciesMeta`
  - 当前包如何补充已声明依赖的附加安装语义

真正决定字段归属的，不是“哪个字段看起来顺手”，而是这个依赖在当前包里的职责。

### 5.1 `dependencies`：运行时直接依赖

`dependencies` 写在 `package.json` 中，用于声明当前包在运行时直接需要的依赖。

典型写法如下：

```json
{
  "dependencies": {
    "vue": "^3.5.0",
    "axios": "^1.12.0"
  }
}
```

这类依赖的共同特征是：

- 业务代码在运行时会直接使用它
- 没有它，程序核心路径无法正常工作
- 它不是只在构建阶段参与，而是运行阶段真正存在于依赖关系中

对应用项目来说，这个判断通常比较直接：

- 前端应用运行时依赖框架本体
- 服务端项目运行时依赖 Web 框架、数据库客户端、日志库
- 业务逻辑运行时直接导入的工具库属于运行时依赖

对库项目来说，判断会更谨慎。库作者需要区分：

- 这是库自己运行时必须携带的依赖
- 还是应该由宿主项目提供的依赖环境

这也是 `dependencies` 与 `peerDependencies` 最容易混淆的地方。

### 5.2 `devDependencies`：工程阶段依赖

`devDependencies` 同样写在 `package.json` 中，但它声明的是当前包在开发、构建、测试、检查阶段需要的依赖，而不是最终运行时代码直接消费的依赖。

典型例子包括：

- TypeScript
- ESLint
- Vitest
- Vite
- Rollup
- Webpack

典型写法如下：

```json
{
  "devDependencies": {
    "typescript": "^5.9.0",
    "vite": "^7.0.0",
    "vitest": "^3.0.0"
  }
}
```

这里最容易出现两个误判。

第一种误判是把“经常使用”误当成“运行时需要”。  
TypeScript、Vite、Vitest 可能每天都在用，但这不意味着它们属于业务运行时代码本身。

第二种误判是把“构建产物离不开它”误当成“部署后也离不开它”。  
如果一个包只参与源码编译、类型检查、测试执行，而最终产物运行时不直接加载它，它通常属于 `devDependencies`。

因此，`devDependencies` 的关键词不是“不重要”，而是“工程阶段”。

### 5.3 `peerDependencies`：宿主兼容契约

`peerDependencies` 也是写在 `package.json` 中，但它表达的不是“当前包自己要安装一份什么”，而是：

> 当前包要求宿主环境提供一份满足条件的依赖。

官方文档对 `peerDependencies` 的描述已经把这个边界定得很清楚：它的值通常是 semver 范围，也可以是 `workspace:` 或 `catalog:` 形式的说明；自 `pnpm v11.14.0` 起，还支持带 scheme 的写法，例如 `npm:` alias、`file:`、git、URL 等形式，但语义仍然是“描述宿主应提供什么依赖约束”。

最常见的写法如下：

```json
{
  "peerDependencies": {
    "vue": "^3.5.0"
  }
}
```

这段声明表达的不是“当前包自己带一份 `vue`”，而是：

> 使用当前包的宿主项目，必须提供一份满足 `^3.5.0` 的 `vue`。

这类依赖常见于：

- Vue / React 组件库
- ESLint 插件
- Vite / Rollup / Webpack 插件
- 运行在某个宿主框架上下文中的生态包

为什么这些场景适合 `peerDependencies`，而不是普通 `dependencies`？

因为它们往往要求和宿主共享同一份运行时实例或同一份工具链上下文。如果这类依赖被错误地写成普通 `dependencies`，就可能引入多份宿主实例，进而导致：

- 运行时上下文分裂
- 插件挂载对象不一致
- 类型系统出现多份实例不兼容
- 单例对象、Store、Context、Router 等语义失真

`peerDependencies` 的本质不是“少装一个包”，而是“声明兼容契约”。

### 5.4 `optionalDependencies`：允许降级的依赖

`optionalDependencies` 写在 `package.json` 中，用于声明那些“最好存在，但缺失时允许安装继续进行”的依赖。

典型写法如下：

```json
{
  "optionalDependencies": {
    "fsevents": "^2.3.3"
  }
}
```

这个字段真正表达的是安装容错语义，而不是“这个依赖不重要”。

它适合的场景通常包括：

- 平台相关的原生能力
- 有高性能实现与降级实现的双路线依赖
- 某些增强能力或附加插件

但它有一个非常重要的前提：

> 当前包必须真的能够在缺失该依赖时退化运行。

如果一个依赖一旦缺失，核心功能就完全不可用，那么把它写进 `optionalDependencies` 只是在掩盖建模错误，而不是正确表达依赖关系。

### 5.5 `peerDependenciesMeta`：给 peer 契约补充附加语义

`peerDependenciesMeta` 也写在 `package.json` 中，用于给已经声明在 `peerDependencies` 里的依赖补充额外信息。

官方文档明确指出，这个字段列出的是与 `peerDependencies` 相关的附加信息；其中最重要、最常见的子属性是：

- `peerDependenciesMeta.<dep>.optional`

典型写法如下：

```json
{
  "peerDependencies": {
    "foo": "1"
  },
  "peerDependenciesMeta": {
    "foo": {
      "optional": true
    }
  }
}
```

这段配置表达的是：

- `foo` 仍然属于 peer 依赖
- 但宿主缺失它时，不再按普通 peer 缺失方式处理

这里的关键点不是“把 peer 变成普通依赖”，而是“给 peer 契约增加可选语义”。

因此，`peerDependenciesMeta` 的职责不是新增一类依赖，而是修饰已经存在的 peer 契约。

### 5.6 `dependenciesMeta`：给已声明依赖补充安装语义

`dependenciesMeta` 同样写在 `package.json` 中。官方文档对它的定义非常明确：

> 这是用于补充已经声明在 `dependencies`、`optionalDependencies`、`devDependencies` 中依赖的附加元信息。

这句话有两个边界必须先记住。

第一，它不是新的依赖类型。  
第二，它修饰的是已经声明过的依赖，而不是凭空新增依赖。

最小结构如下：

```json
{
  "dependencies": {
    "button": "workspace:1.0.0"
  },
  "dependenciesMeta": {
    "button": {
      "injected": true
    }
  }
}
```

这里的层级关系是：

- `dependencies.button`
  - 先声明当前包依赖 `button`
- `dependenciesMeta.button`
  - 再补充这个依赖的安装元语义

因此，`dependenciesMeta` 的问题域不是“依赖属于哪一类”，而是“这个依赖在安装时有没有额外语义”。

### 5.7 `dependenciesMeta.<dep>.injected` 的归属与边界

`dependenciesMeta` 中最容易绕进去的字段是 `dependenciesMeta.<dep>.injected`。它同样写在消费者包自己的 `package.json` 中，而不是写在 `pnpm-workspace.yaml` 或被依赖包自己的 `package.json` 中。

官方文档对它的核心语义定义得很具体：

- 当 `injected: true` 且目标依赖是本地 workspace package 时，`pnpm` 会在 virtual store（`node_modules/.pnpm`）中为它创建 hard linked copy
- 如果未设置或为 `false`，则该依赖默认通过 `node_modules` 中指向 workspace 源目录的 symlink 被使用

这里先只建立三个结论，不在本章提前展开完整场景：

1. `injected` 只在本地 workspace dependency 语境下讨论才有意义  
2. 它属于 `dependenciesMeta.<dep>` 这一层，而不是全局安装模式  
3. 它改变的是本地 workspace 依赖进入消费者依赖树的方式

为什么这一节不在这里展开所有案例？  
因为它和 `peerDependencies` 的多上下文解析、workspace 本地包安装方式、virtual store 中的实例落点强相关。完整问题会在第 8 章和第 9 章中拆开处理。

本章只保留最重要的边界：`dependenciesMeta.<dep>.injected` 是 `package.json` 中对某个已声明 workspace 依赖的附加安装语义。

### 5.8 本章结论

`package.json` 中的依赖字段，本质上是在描述当前包与外部包之间的关系模型。

需要始终区分三层问题：

- 这个依赖是不是当前包运行时直接需要
- 这个依赖是不是宿主环境应提供的契约
- 这个依赖是否还需要额外的元语义修饰

压缩成一组判断规则如下：

- 运行时直接需要：优先判断 `dependencies`
- 只服务于开发、构建、测试：优先判断 `devDependencies`
- 宿主必须提供：优先判断 `peerDependencies`
- 缺失时允许降级：再判断 `optionalDependencies`
- peer 是否附加可选语义：看 `peerDependenciesMeta`
- 已声明依赖是否附加安装语义：看 `dependenciesMeta`

## 6. `pnpm-lock.yaml` 与可复现安装

这一章只处理一个问题：

> `pnpm-lock.yaml` 到底在固定什么，为什么它能决定安装是否可复现。

如果不先把这个问题讲清楚，后面关于 CI、离线环境、依赖漂移、版本统一、供应链安全的讨论都会失去支点。因为这些问题最终都绕不开同一件事：

- `package.json` 只声明意图
- `pnpm-lock.yaml` 固定解析结果

### 6.1 `pnpm-lock.yaml` 的职责

`pnpm-lock.yaml` 是当前仓库依赖图的解析结果文件。它记录的不是“想要什么”，而是“这次最终解析成了什么”。

这一定义必须和 `package.json` 分开理解：

- `package.json`
  - 表达依赖声明、版本范围、脚本、入口和元信息
- `pnpm-lock.yaml`
  - 记录当前依赖图被解析后的精确结果

例如：

```json
{
  "dependencies": {
    "vue": "^3.5.0"
  }
}
```

这段 `package.json` 只表达两件事：

- 当前包依赖 `vue`
- 可接受满足 `^3.5.0` 的版本

它并没有表达：

- 本次实际解析到了哪个精确版本
- 这个版本又依赖了哪些下游包
- 这些下游包在当前仓库里如何组合成最终依赖图

这些信息由 `pnpm-lock.yaml` 负责固定。

### 6.2 它不是附属文件，而是依赖图快照

把 lockfile 理解成“安装时顺手生成的附属文件”是不准确的。更准确的说法是：

> lockfile 是依赖图快照。

这里的“快照”不是指单个版本号清单，而是当前仓库在当前声明、当前配置、当前解析规则下，得到的一整张依赖图结果。

因此，`pnpm-lock.yaml` 至少承担三类职责：

- 固定直接依赖和传递依赖的精确版本
- 固定 peer 上下文参与后的最终包实例组合
- 固定安装所需的校验信息和部分解析元数据

这也是为什么 lockfile 变化往往会带来大量 diff。它不是一个“增一条依赖就只改一行”的文件，而是在描述整张解析结果图。

### 6.3 当前项目的 lockfile 长什么样

当前 `daisy-ui` 项目的 [`pnpm-lock.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-lock.yaml) 开头结构如下：

```yaml
lockfileVersion: '9.0'

settings:
  autoInstallPeers: true
  excludeLinksFromLockfile: false

catalogs:
  default:
    vue:
      specifier: ^3.5.40
      version: 3.5.41

importers:
  .:
    devDependencies:
      vite:
        specifier: catalog:
        version: 7.1.5

packages:
  vite@7.1.5:
    resolution: ...
```

这里至少有五层信息值得先建立概念。

#### `lockfileVersion`

`lockfileVersion` 表示 lockfile 自身采用的格式版本，而不是项目依赖版本，也不是 `pnpm` CLI 版本本身。

这个字段的重要性在于兼容性判断。不同主版本 `pnpm` 对 lockfile 格式的读写兼容边界并不完全相同，因此不能把它理解为可忽略的头部元数据。

#### `settings`

`settings` 记录会影响 lockfile 语义的部分安装设置。当前项目里可以看到：

- `autoInstallPeers: true`
- `excludeLinksFromLockfile: false`

这说明 lockfile 并不只保存“最终版本号”，还会带上与解析结果相关的关键环境信息。

#### `catalogs`

当前项目使用了 `catalog`。因此 lockfile 会记录 catalog 中每个依赖入口的：

- 原始声明 `specifier`
- 实际锁定结果 `version`

这意味着 lockfile 不是只面向普通 `dependencies`，它也要把版本治理层的最终结果固定下来。

#### `importers`

在 workspace / Monorepo 场景下，`importers` 是理解 lockfile 的关键入口。它表示当前仓库中每个 importer，也就是每个安装入口包，各自声明了什么依赖。

当前项目里 `importers` 下会列出：

- 根包 `.`
- 各个 `apps/*`
- 各个 `packages/*`
- `internal/*`
- `scripts/*`

这说明 Monorepo 下的 lockfile 不是“某一个包的锁文件”，而是**整个仓库所有 importer 的统一解析快照**。

#### `packages`

`packages` 区域描述的是最终被解析出来的具体包实例。这里会记录更完整的包级信息，例如：

- 精确版本
- resolution 信息
- integrity 等校验信息
- 该包自身的依赖边界
- 与 peer 上下文有关的实例化结果

真正决定安装落盘的，不是单独某个 importer 项，而是 `importers` 与 `packages` 共同构成的完整解析结果。

### 6.4 lockfile 具体记录什么

从工程视角看，`pnpm-lock.yaml` 至少会记录以下几类信息。

**每个 importer 的依赖入口**

也就是每个 workspace 包或根包，声明了哪些：

- `dependencies`
- `devDependencies`
- `optionalDependencies`

并且每一项通常都会同时出现：

- `specifier`
  - 原始声明范围
- `version`
  - 当前锁定结果

这让 lockfile 同时保留了“原始输入”和“当前输出”的对应关系。

**整张传递依赖图的解析结果**

这部分信息不是 `package.json` 能表达的，因为传递依赖并不是当前项目手写出来的，而是包管理器根据每个包的声明继续递归解析得到的。

**peer 参与后的实际实例**

`pnpm` 的 lockfile 不是普通的“包名 -> 版本”平面表。对于存在 peer 上下文差异的包，同一个包版本可能会在不同上下文下形成不同实例结果。lockfile 必须把这种实例化结果记录下来，否则安装无法被稳定复现。

**完整性校验与来源相关元数据**

lockfile 中会记录 tarball integrity 等校验信息。自 `pnpm 11.4.0` 起，若下载到的 tarball 与 lockfile 中记录的 integrity 不一致，`pnpm install` 默认会直接以 `ERR_PNPM_TARBALL_INTEGRITY` 失败，而不是静默重写 lockfile。

这使得 lockfile 的意义不再只是“固定版本”，而是进一步参与：

- 安装结果校验
- registry / proxy 异常识别
- 供应链篡改风险防护

### 6.5 `package.json` 与 `pnpm-lock.yaml` 的关系

两者的关系可以压缩成一条稳定流程：

1. `package.json` 声明依赖意图
2. `pnpm` 根据版本范围、workspace 结构、peer 关系和相关配置做解析
3. `pnpm-lock.yaml` 固定这次解析结果
4. 后续安装优先复用这个固定结果

因此，`pnpm-lock.yaml` 不是 `package.json` 的备份，也不是它的副本。它是“声明”经过解析后得到的“结果”。

这也解释了两个常见现象。

第一，`package.json` 不变，不代表 lockfile 永远不会变。  
如果解析规则、workspace 结构、catalog、overrides、peer 上下文或包管理器版本发生变化，lockfile 仍可能需要更新。

第二，lockfile 变化，也不必然意味着 `package.json` 手工改过。  
某些变动来自重新解析、依赖去重、配置调整、merge 冲突修复或版本基线变化。

### 6.6 `pnpm install` 如何使用 lockfile

`pnpm install` 的默认目标不是“每次都重新解析一整张依赖图”，而是优先复用现有 lockfile 描述的结果。

可以把安装过程收敛为下面这条判断链：

1. 读取 `package.json`
2. 读取 `pnpm-lock.yaml`
3. 判断当前声明、配置和 lockfile 是否仍然一致
4. 若一致，则按 lockfile 复现安装结果
5. 若不一致，则重新解析并更新 lockfile

这就是 lockfile 成为可复现安装基础的原因。只要：

- lockfile 已提交
- 当前环境与它兼容
- 安装不需要改写它

那么不同开发机、CI 节点、Docker 构建环境就更容易得到同一份依赖图结果。

### 6.7 lockfile 什么时候会变化

lockfile 变化本身不是异常，关键要看变化来源是否可解释。

最常见的变化来源包括：

- `pnpm add`
- `pnpm remove`
- `pnpm update`
- `pnpm dedupe`
- 直接修改 `package.json`
- 修改 `catalog`、`overrides`、`packageExtensions` 等治理配置
- workspace 包增删或路径调整
- peer 上下文变化导致实例重新计算

除此之外，普通的 `pnpm install` 也可能更新 lockfile。原因不是“install 会乱改文件”，而是：

> 当当前声明、配置和 lockfile 之间不再一致时，`pnpm install` 需要重新解析，并把新结果写回 `pnpm-lock.yaml`。

因此，看到 lockfile diff 时，首先应判断：

- 这次变化是否由依赖声明变更引起
- 是否由治理配置变更引起
- 是否由 `pnpm` 主版本变化引起
- 是否由 merge 冲突修复不完整引起

只有在变化来源解释不通时，lockfile 才是异常信号。

### 6.8 `--frozen-lockfile` 到底在冻结什么

`--frozen-lockfile` 的准确含义不是“使用 lockfile”，因为普通安装本来就会使用 lockfile。它冻结的是：

> 安装过程中不允许通过重新解析来改写 lockfile。

因此，下面两种情况的结果不同：

- 当前声明与 lockfile 一致
  - 可以按 lockfile 正常安装
- 当前声明与 lockfile 不一致，或安装过程需要更新 lockfile
  - 直接失败

官方 `pnpm install` 文档对这个行为写得很明确：`--frozen-lockfile` 下，`pnpm-lock.yaml` 不会被更新；若 lockfile 缺失、过期或需要变更，则安装失败。参考：`pnpm install` 官方文档 `--frozen-lockfile`、`--lockfile-only`、`--no-lockfile` 说明，见 `https://pnpm.io/cli/install`。

### 6.9 为什么 CI 经常配 `--frozen-lockfile`

CI 的职责通常不是“替仓库重新算一版依赖图”，而是验证：

> 仓库当前已经提交的状态，能否被稳定复现。

因此，CI 与 `--frozen-lockfile` 的组合非常自然。它在语义上做的是同一件事：

- 仓库提交了某份 lockfile
- CI 必须按这份 lockfile 验证
- 如果这份结果已经和声明不一致，就应该失败，而不是在 CI 里顺手改文件

在 `pnpm 11/12` 官方文档中，还需要补一条容易遗漏的行为边界：

- 当 `pnpm` 检测到当前运行在 CI 中时，会默认切换到 frozen-lockfile 模式
- 自 `pnpm 11` 起，如果 lockfile 是由更新的 `pnpm` 主版本写出的，CI 安装也会直接失败，而不是静默改写

参考：`pnpm install` 与 `Continuous Integration` 官方文档，见 `https://pnpm.io/cli/install`、`https://pnpm.io/continuous-integration`。

这条规则的工程意义非常直接：

- CI 不是重新生成依赖结果的地方
- CI 是验证仓库提交结果是否自洽的地方

### 6.10 为什么 lockfile 不等于“结果一定合理”

lockfile 的存在只能证明：

> 当前仓库已经固定了一份解析结果。

它不能证明这份结果一定合理、最优、健康。

仍然可能出现以下情况：

- peer 上下文导致多实例分裂
- 治理配置本身就不合理
- 旧版 `pnpm` 生成了团队不想保留的结果
- merge 冲突修复不完整
- 去重前后的结果混杂

因此，正确理解 lockfile 的方式不是“只要有它就万事大吉”，而是：

- 没有它，很难保证可复现
- 有了它，也仍然需要评审这份结果是否合理

### 6.11 为什么 lockfile 变更值得评审

lockfile diff 往往很长，但它不是“无脑生成文件”，不能因为行数多就直接忽略。至少应关注以下几类信息：

- 新增了哪些包
- 某些包是否发生了异常升级或降级
- 是否出现了与当前改动不相称的大范围重排
- integrity、来源、peer 相关结果是否异常

尤其在 Monorepo 中，一个小范围配置调整可能连带改写大量 importer 和 packages 区域。如果不审 lockfile，只审 `package.json`，就等于只看输入、不看最终输出。

### 6.12 本章结论

`pnpm-lock.yaml` 的职责不是陪衬 `package.json`，而是把依赖声明解析成一个可复用、可校验、可评审的结果快照。

这一章可以收敛为四个结论：

1. `package.json` 声明依赖意图，`pnpm-lock.yaml` 固定解析结果
2. lockfile 记录的不只是版本号，还包括 importer、包实例、校验信息和部分解析设置
3. `pnpm install` 默认优先复用 lockfile，`--frozen-lockfile` 则禁止安装时改写它
4. CI 使用 frozen lockfile，本质上是在验证仓库提交结果的可复现性

## 7. 安装结果如何落到磁盘

这一章只讨论磁盘布局，不讨论 workspace 定义、peer 分裂排错和 hoist 策略取舍。核心问题只有三个：

- `pnpm install` 之后，磁盘上到底出现了哪几层对象
- 根 `node_modules` 里为什么大多是链接，但程序仍然能正常解析依赖
- `.pnpm` 里那些很长的目录名到底在表达什么

### 7.1 安装结果至少要分三层看

理解 `pnpm` 的磁盘布局，不能只盯着项目根 `node_modules`。至少要把下面三层拆开：

- 全局 store
  - 保存包内容对象
- 项目级 virtual store，也就是 `node_modules/.pnpm`
  - 保存当前项目实际生成出的包实例目录
- 项目根 `node_modules`
  - 作为当前项目的直接依赖入口层

如果把这三层混成一层，就会反复出现这些误判：

- 把 store 误以为是“全局 node_modules”
- 把 `.pnpm` 误以为是缓存目录
- 把根 `node_modules` 误以为包含了所有真实包目录

### 7.2 第一层：store 负责保存内容对象

store 是 `pnpm` 的全局内容存储区。当前机器上执行 `pnpm store path` 得到的路径是：

```text
/Users/apple/Library/pnpm/store/v11
```

这个路径与具体机器、用户目录、`pnpm` 主版本相关，不应把它当成固定常量。真正需要理解的是职责，而不是路径字符串本身。

store 的职责可以压缩成一句话：

> 保存包内容对象，并尽可能在多个项目之间复用这些内容。

当前机器上的 store 目录下可以看到典型结构：

```text
/Users/apple/Library/pnpm/store/v11
├── files/
│   ├── 00/
│   ├── 01/
│   ├── 02/
│   └── ...
├── index.db
├── links/
├── projects/
└── .tmp/
```

这里最关键的是 `files/`。它不是按包名建目录，而是按内容哈希前缀分桶。例如当前机器上能看到：

```text
files/
├── 00/
├── 01/
├── 02/
├── 03/
└── ...
```

这说明 store 的核心组织方式不是：

```text
files/vue/
files/vite/
files/postcss/
```

而是：

> 先以内容为单位寻址，再按哈希前缀分桶保存。

因此，store 更接近内容对象库，而不是给开发者直接阅读的“包树”。

### 7.3 第二层：`node_modules/.pnpm` 才是当前项目的包实例层

项目根的 `node_modules/.pnpm` 是 virtual store。它不是全局缓存，而是当前项目依赖图在本地磁盘上的实例化结果。

当前 `daisy-ui` 的 [`node_modules/.pnpm`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/.pnpm) 中可以看到大量实例目录，例如：

```text
@ant-design+icons-vue@7.0.1_vue@3.5.41_typescript@6.0.3_
@antdv-next+cssinjs@1.0.6_vue@3.5.41_typescript@6.0.3_
@apideck+better-ajv-errors@0.3.7_ajv@8.20.0
vue@3.5.41_typescript@6.0.3
vite@8.2.2_@types+node@25.9.5_esbuild@0.28.2_jiti@2.7.0_less@4.9.0_supports-color@10.2._1d88c90ccb5404d5cb742f52e701d83b
```

这些目录名表达的不是“显示起来难看”，而是当前项目真实的包实例身份。至少包含两层信息：

- 包本身的名称与版本
- 参与该实例解析的 peer 上下文或关键依赖上下文

因此，`.pnpm` 中保存的不是抽象包名，而是**当前项目真正要落盘和参与解析的实例**。

### 7.4 第三层：根 `node_modules` 是入口层，不是完整包树

当前 `daisy-ui` 根 [`node_modules`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules) 中可以直接看到大量符号链接入口，例如：

```text
cross-env -> .pnpm/cross-env@10.1.0/node_modules/cross-env
eslint -> .pnpm/eslint@10.9.0_jiti@2.7.0_supports-color@10.2.2/node_modules/eslint
postcss -> .pnpm/postcss@8.5.26/node_modules/postcss
vite -> .pnpm/vite@8.2.2_@types+node@25.9.5_esbuild@0.28.2_jiti@2.7.0_less@4.9.0_supports-color@10.2._1d88c90ccb5404d5cb742f52e701d83b/node_modules/vite
vue -> .pnpm/vue@3.5.41_typescript@6.0.3/node_modules/vue
```

这说明根 `node_modules` 的职责不是“保存所有真实文件”，而是：

- 暴露当前项目的直接依赖入口
- 把入口指向 `.pnpm` 中的真实包实例目录

因此，把根 `node_modules` 看成“真实依赖树”会很容易误判。对 `pnpm` 来说，它更接近入口索引层。

### 7.5 为什么包实例目录要长成 `.../node_modules/<pkg>`

`.pnpm` 中的包实例路径不是直接写成：

```text
node_modules/.pnpm/vue@3.5.41/
```

而是继续套一层：

```text
node_modules/.pnpm/vue@3.5.41_typescript@6.0.3/node_modules/vue
```

官方关于 symlinked `node_modules` structure 的说明给出了两个直接原因：

1. 允许包导入自己  
   包本身需要能够从自己的运行位置解析到自己，例如访问自己的 `package.json`
2. 避免构造循环 symlink  
   依赖链接关系需要建立在统一的 `node_modules` 语义层级里

这也是为什么：

- `.pnpm/<instance>/node_modules/<pkg>` 是真实包实例目录
- 它不是一个再指向别处的符号链接目录

当前 `daisy-ui` 中的 [`vue` 实例目录](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/.pnpm/vue@3.5.41_typescript@6.0.3/node_modules/vue) 就是一个真实目录，而不是 symlink。

### 7.6 包内容从 store 进入实例目录的方式

这里最容易说错，所以边界先定清楚。

`pnpm` 的包图链接关系大量依赖 symlink，但**store 到实例目录的内容导入方式**并不只有一种。它由 `packageImportMethod` 控制。

官方 settings 文档对 `packageImportMethod` 的默认值定义为 `auto`。其核心含义是：

- 先尝试 clone
- 若 clone 不可用，再尝试 hard link
- 若仍不可用，再退回 copy

因此，下面三件事不能混为一谈：

- store 到实例目录的内容导入方式
  - clone / hard link / copy
- 包实例之间构造依赖图的方式
  - symlink
- 根 `node_modules` 暴露直接依赖入口的方式
  - symlink

这也是为什么不能把 `.pnpm` 一概说成“里面全是硬链接”。更准确的说法是：

> 当前项目的包实例目录由 store 导入内容，但具体导入方式受文件系统能力和 `packageImportMethod` 影响；在这些实例之间，`pnpm` 再用 symlink 构造依赖图。

### 7.7 一个真实例子：`vue` 的入口层与实例层

当前 `daisy-ui` 中，根入口 [`node_modules/vue`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/vue) 的 `realpath` 是：

```text
/Users/apple/code/WebProjects/daisy-ui/node_modules/.pnpm/vue@3.5.41_typescript@6.0.3/node_modules/vue
```

这说明：

- 项目源码里 `import 'vue'`
  - 命中的是根 `node_modules` 入口
- 入口本身只是 symlink
  - 它最终会落到 `.pnpm` 中的实例目录

当前 `vue` 实例所在的同级 `node_modules` 目录下还能看到：

```text
@vue/
typescript -> ../../typescript@6.0.3/node_modules/typescript
vue/
```

这反映出一个关键事实：

- 当前包实例所在目录并不是孤立目录
- 它的依赖入口与包自身位于同一个实例级 `node_modules` 层中

### 7.8 一个更复杂的例子：`vite` 为什么目录名这么长

当前项目里，根入口 [`node_modules/vite`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/vite) 指向的是：

```text
.pnpm/vite@8.2.2_@types+node@25.9.5_esbuild@0.28.2_jiti@2.7.0_less@4.9.0_supports-color@10.2._1d88c90ccb5404d5cb742f52e701d83b/node_modules/vite
```

这个目录名里，至少可以直接读出几层信息：

- 包名：`vite`
- 包版本：`8.2.2`
- 参与该实例解析的一组上下文依赖：
  - `@types/node`
  - `esbuild`
  - `jiti`
  - `less`
  - `supports-color`
- 末尾还有一段哈希样的后缀

这里不应把这串目录名理解成“只是内部实现噪音”。它表达的是：

> 当前项目里这个 `vite` 实例，并不是脱离上下文独立存在的；它带着一组在当前解析结果中参与实例身份判定的依赖条件。

第 9 章会专门展开 peer 与实例分裂，这一章先只保留结论：`.pnpm` 中的目录名不是为了好看，而是为了标识实例身份。

### 7.9 包实例之间如何连成依赖图

只看根 `node_modules` 容易误以为依赖图已经结束。实际上，真正的依赖图是在实例层继续通过 symlink 连接起来的。

当前项目中某个 `vite` 实例的 `node_modules` 下可以看到这样的结构：

```text
esbuild -> ../../esbuild@0.28.2/node_modules/esbuild
fsevents -> ../../fsevents@2.3.3/node_modules/fsevents
jiti -> ../../jiti@2.7.0/node_modules/jiti
less -> ../../less@4.9.0_supports-color@10.2.2/node_modules/less
postcss -> ../../postcss@8.5.26/node_modules/postcss
vite/
yaml -> ../../yaml@2.9.0/node_modules/yaml
```

这说明：

- 当前 `vite` 实例自己的目录是 `vite/`
- 它依赖的其他包不是复制进来一份
- 而是通过同级 `node_modules` 下的 symlink 指向其他实例目录

因此，`pnpm` 的依赖图不是“把所有内容平铺到一个目录里”，而是：

1. 先生成包实例目录
2. 再在实例级 `node_modules` 中建立依赖链接
3. 最后只把项目的直接依赖暴露到根 `node_modules`

### 7.10 Node.js 为什么能沿这套结构正常解析

这是很多人第一次看到 `.pnpm` 布局时最不放心的地方：根入口明明是 symlink，为什么运行时还能正常找到子依赖？

关键在于 Node.js 默认解析时会基于 `realpath` 工作。也就是说：

- 入口 `node_modules/vite`
  - 先通过 symlink 落到真实路径
- 真实路径变成：
  - `.pnpm/vite@.../node_modules/vite`

从这里开始，Node 在解析 `vite` 内部依赖时，会从它真实所在的位置向上查找 `node_modules`。而这个真实路径的父级正好就是：

```text
.pnpm/vite@.../node_modules
```

这个目录里已经存在：

- `vite/`
- `postcss -> ../../postcss@.../node_modules/postcss`
- `esbuild -> ../../esbuild@.../node_modules/esbuild`
- 其他依赖入口

因此，`vite` 内部去解析 `postcss`、`esbuild`、`yaml` 时，Node 能在真实路径的上层 `node_modules` 里找到这些依赖。

这也是 `pnpm` 这套结构成立的关键。它不是依赖 Node 对“奇怪目录”做特殊支持，而是：

- 利用真实路径回到实例目录
- 再利用标准的 `node_modules` 向上查找规则完成解析

### 7.11 `.pnpm/node_modules` 这个隐藏层是什么

当前 `daisy-ui` 的 [`node_modules/.pnpm/node_modules`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/.pnpm/node_modules) 目录是存在的，里面可以看到大量包入口。

这一层和上面讲的包实例层不是同一个概念。它通常与 hoist 相关，用于承载被提升出来的入口。

这一章只先建立两个边界：

- 它可能存在，而且在默认配置下很常见
- 它不等于 `.pnpm` 的包实例层

至于它为什么存在、默认 hoist 行为如何影响生态兼容、哪些布局配置会改变这一层结构，这些问题统一放到第 11 章展开。

### 7.12 本章结论

`pnpm` 的磁盘布局至少要分成三层理解：

- store
  - 保存内容对象
- `node_modules/.pnpm`
  - 保存当前项目真正的包实例目录
- 根 `node_modules`
  - 暴露当前项目的直接依赖入口

这套结构能成立，依赖的是两件事：

- 内容导入与目录构造分层处理  
  store 负责内容来源，实例层负责项目内的包身份
- Node.js 默认基于 `realpath` 和标准 `node_modules` 查找规则完成解析  
  symlink 只是入口和图连接手段，不会破坏模块解析

因此，`.pnpm/<instance>/node_modules/<pkg>` 才是理解 `pnpm` 安装结果的核心观察点。只看根 `node_modules`，一定会低估它的真实结构。

## 8. workspace 与 Monorepo 模型

这一章讨论的不是“Monorepo 很流行”这种泛概念，而是 `pnpm` 里 workspace 到底如何成立，以及它如何改变本地包解析。

核心问题只有两个：

- 哪些目录会被 `pnpm` 认定为同一个 workspace
- 当一个包依赖另一个本地包时，`pnpm` 到底按什么规则解析

### 8.1 workspace 在 `pnpm` 里不是口号，而是一个明确对象

在 `pnpm` 语境下，workspace 不是泛指“一个仓库里有很多包”，而是一个由 `pnpm-workspace.yaml` 显式定义出来的对象。

官方文档对这件事的定义很直接：

- workspace 根由 `pnpm-workspace.yaml` 决定
- 该文件负责定义哪些目录被包含进 workspace
- 如果 `packages` 字段省略，则只有根包被包含
- 根包始终属于 workspace，即使使用了自定义通配规则

这意味着两个边界必须先分开。

第一，仓库里有很多 `package.json`，不等于这些包自动属于同一个 workspace。  
第二，目录结构看起来像 Monorepo，也不等于 `pnpm` 已经把它识别为 workspace。

真正让多个包被统一纳入 `pnpm` 管理边界的，是根目录下的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml)。

### 8.2 `pnpm-workspace.yaml` 的职责

`pnpm-workspace.yaml` 首先定义的是 workspace 根和包收集范围。它不是 lockfile，不负责记录解析结果；也不是 `package.json`，不负责单个包自己的依赖声明。

它在 workspace 语境里的核心职责可以压缩成三点：

- 确定当前仓库的 workspace 根
- 通过 `packages` 字段定义哪些路径属于 workspace
- 承载作用于整个 workspace 的部分集中配置

本章只展开前两点。后续像 `catalog`、`overrides`、布局策略、安全策略这些集中配置，分别在第 10、11、14 章展开。

### 8.3 当前项目如何定义 workspace 边界

当前 `daisy-ui` 的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 开头是：

```yaml
packages:
  - internal/*
  - internal/lint-configs/*
  - packages/*
  - packages/@core/base/*
  - packages/@core/ui-kit/*
  - packages/@core/forward/*
  - packages/@core/*
  - packages/effects/*
  - packages/business/*
  - apps/*
  - scripts/*
  - docs
  - playground
```

这段配置的含义不是“列几个常见目录看看”，而是在定义：

> 哪些目录下的包会被收进当前 workspace，成为同一张依赖图、同一个 lockfile、同一套 workspace 本地解析规则的一部分。

从当前项目可以直接读出几类成员：

- `apps/*`
  - 各个应用入口
- `packages/*` 及其更细的子层级
  - 各类共享包、基础包、效果包、业务包
- `internal/*`
  - 仓库内部工具包、配置包
- `scripts/*`
  - 脚本类包
- `docs`
  - 文档站点本身也是 workspace 成员
- `playground`
  - 演示或实验入口

这里一个很重要的观察是：`docs` 和 `playground` 用的是字面路径，而不是通配符。说明 `packages` 不只是“批量搜包”，也可以显式纳入单个目录。

### 8.4 `packages` 字段到底在表达什么

`packages` 字段表达的是**路径匹配规则**，不是包名列表。

这一点必须说清，因为很多人第一次看 workspace 都会产生误解，以为这里列的是：

```yaml
packages:
  - '@vben/utils'
  - '@vben/types'
  - '@vben/icons'
```

这不是 `pnpm-workspace.yaml` 的工作方式。它真正做的是告诉 `pnpm`：

- 去哪些路径下寻找 package
- 哪些目录应纳入 workspace
- 哪些目录不属于当前 workspace

官方文档给出的能力边界包括：

- 支持普通路径
- 支持 `*`、`**` 这类通配
- 支持以 `!` 开头的排除规则

例如官方示例里就有：

```yaml
packages:
  - 'packages/*'
  - 'components/**'
  - '!**/test/**'
```

因此，`packages` 解决的是“边界收集”，而不是“依赖声明”。

### 8.5 根包永远属于 workspace

这是一个很容易被忽略，但对理解整仓行为非常关键的规则。

官方文档明确指出：

> 根包始终会被包含进 workspace，即使 `packages` 配置使用了自定义通配规则。

这条规则意味着：

- 根目录的 `package.json` 不需要出现在 `packages` 列表里
- 根包照样会成为 workspace 的 importer
- 根包的依赖、脚本、工具链入口会进入整仓依赖图

这和当前项目的 lockfile 结构是对得上的。前面第 6 章已经看到，当前 [`pnpm-lock.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-lock.yaml) 的 `importers` 里存在根项 `.`，这正是根包属于 workspace 的结果。

### 8.6 workspace 解决的不是“多目录”，而是“本地包统一解析”

把多个目录纳入 workspace，本身不是最终目的。真正重要的是：这些目录中的包一旦被统一纳入 workspace，就可以参与同一套本地包解析规则。

这会带来几件具体事情：

- 同一个 lockfile 管理整仓依赖图
- 本地包之间可以通过 workspace 机制建立依赖关系
- 安装时，`pnpm` 知道哪些包是“仓库内部成员”，哪些包必须走远程 registry
- 命令执行、过滤、递归安装、发布前替换等行为都可以基于 workspace 统一工作

因此，workspace 不是目录组织美学，而是本地依赖解析与工程治理的基础对象。

### 8.7 `workspace:` 协议为什么重要

到这里才真正进入本章的第二个重点：workspace 协议。

官方文档给出了一个非常关键的对照：

- `linkWorkspacePackages` 的官方默认值是 `false`
- 如果 `linkWorkspacePackages` 为 `true`，并且某个本地包版本满足声明范围，`pnpm` 可以直接把 workspace 包链接进来
- 但如果声明范围不匹配，或者解析条件不成立，`pnpm` 仍可能转而从 registry 安装同名包

这会引入一种不确定性：

> 看起来像在依赖本地包，实际却可能在某些情况下退回到远程 registry 结果。

`workspace:` 协议就是用来消除这种不确定性的。它表达的是：

> 这个依赖只能解析到本地 workspace 包，不能退回到 registry 上的同名包。

官方文档对它的行为写得很明确：

- 如果写成 `"foo": "workspace:2.0.0"`
- 而当前 workspace 中不存在满足该条件的 `foo`
- 安装会直接失败

这就是它的核心价值：**把“优先本地”变成“必须本地”。**

### 8.8 `workspace:` 不是语法糖，而是解析约束

很多人会把：

```json
{
  "dependencies": {
    "@vben/utils": "workspace:*"
  }
}
```

理解成一种“写法更短的本地引用”。这个理解不够准确。

更准确的说法是：

> `workspace:` 是对解析来源的显式约束。

它真正改变的是失败条件和回退策略：

- 普通 semver 范围
  - 在一定条件下可能匹配到本地包，也可能退回到 registry
- `workspace:` 协议
  - 只能使用本地 workspace 包，不满足就报错

所以它不是“本地包别名语法”，而是“本地包强约束协议”。

### 8.9 当前项目为什么大量使用 `workspace:*`

当前 `daisy-ui` 中，大量包都明确写了 `workspace:*`。例如：

- [apps/web-antd/package.json](file:///Users/apple/code/WebProjects/daisy-ui/apps/web-antd/package.json)
- [apps/web-ele/package.json](file:///Users/apple/code/WebProjects/daisy-ui/apps/web-ele/package.json)
- [apps/web-naive/package.json](file:///Users/apple/code/WebProjects/daisy-ui/apps/web-naive/package.json)
- [docs/package.json](file:///Users/apple/code/WebProjects/daisy-ui/docs/package.json)
- 根 [package.json](file:///Users/apple/code/WebProjects/daisy-ui/package.json)

这类写法的工程含义非常明确：

- 应用层明确依赖整仓内部的共享包
- 根包明确依赖整仓内部的配置包和工具包
- 安装时不接受悄悄退回到 registry 的同名包

从这个项目的写法也能看出一条明确倾向：

> 依赖解析不依赖“也许会匹配到本地包”的隐式行为，而是直接用 `workspace:*` 把本地来源写成显式契约。

例如当前项目根包里就有：

```json
{
  "devDependencies": {
    "@vben/commitlint-config": "workspace:*",
    "@vben/eslint-config": "workspace:*",
    "@vben/tailwind-config": "workspace:*",
    "@vben/vite-config": "workspace:*"
  }
}
```

这不是为了“写起来方便”，而是在表达：

- 这些配置包必须来自当前仓库内部
- 如果当前 workspace 没有这些包，安装就应该失败

对于 Monorepo 来说，这种写法通常比裸 semver 范围更稳。

### 8.10 `workspace:*`、`workspace:^`、`workspace:~`、`workspace:<range>` 的含义

workspace 协议并不是只有 `workspace:*` 一种写法。官方文档列出了几种常见形式：

- `workspace:*`
- `workspace:^`
- `workspace:~`
- `workspace:^1.5.0`
- `workspace:2.0.0`

这些写法都先共享一个前提：

> 解析目标必须是本地 workspace 包。

差别在于发布前替换时如何生成最终对外版本声明。

例如官方文档给出的转换规则是：

```json
{
  "dependencies": {
    "foo": "workspace:*",
    "bar": "workspace:~",
    "qar": "workspace:^",
    "zoo": "workspace:^1.5.0"
  }
}
```

发布时会被替换成：

```json
{
  "dependencies": {
    "foo": "1.5.0",
    "bar": "~1.5.0",
    "qar": "^1.5.0",
    "zoo": "^1.5.0"
  }
}
```

文档还明确说明：

- 裸 `workspace:` 等同于 `workspace:*`

因此，`workspace:*` 不是“永远写死星号”，而是“发布时替换为当前目标包的精确版本”。

### 8.11 alias 形式：`workspace:foo@*`

workspace 协议不仅能引用同名包，也支持 alias。

官方文档给出的写法是：

```json
{
  "dependencies": {
    "bar": "workspace:foo@*"
  }
}
```

这里表达的是：

- 当前项目里实际依赖的是 workspace 包 `foo`
- 但在当前包的依赖名位置上，暴露出来的名字是 `bar`

发布前，官方文档说明它会被替换成普通 alias 依赖：

```json
{
  "dependencies": {
    "bar": "npm:foo@1.0.0"
  }
}
```

这个能力的意义不是“写法花哨”，而是：

- 允许本地 workspace 包以别名方式被消费
- 同时保留发布后对外可识别的标准 npm alias 语义

### 8.12 相对路径形式：`workspace:../foo`

官方文档还支持通过相对路径引用 workspace 包。

在如下结构中：

```text
packages/
├── foo/
└── bar/
```

`bar` 可以这样声明：

```json
{
  "dependencies": {
    "foo": "workspace:../foo"
  }
}
```

这个语法的核心价值是：

- 目标仍然必须是 workspace 成员
- 引用方式改成了相对路径，而不是包名

发布前，这类声明同样会被替换成普通包管理器可理解的版本声明，而不会把 `workspace:` 原样发布出去。

### 8.13 publish / pack 阶段为什么必须做替换

`workspace:` 协议本质上是仓库内部协议。外部消费者拿到发布包后，并不处在当前仓库的 workspace 边界里。

因此，发布时如果不把：

```json
"@vben/utils": "workspace:*"
```

转换成普通版本声明：

```json
"@vben/utils": "1.2.3"
```

那么外部安装者根本无法理解这份依赖意图。

这就是为什么官方文档明确说明：

- `pnpm pack`
- `pnpm publish`
- 其他 publish 命令

都会在打包 / 发布时动态替换 `workspace:` 依赖声明。

从工程语义上看，这一步不是“帮忙改格式”，而是在做协议边界转换：

- 仓库内
  - 用 `workspace:` 表达“必须来自本地 workspace”
- 仓库外
  - 用普通 semver / alias 声明表达“发布后外部应如何安装”

### 8.14 workspace 包的解析边界与失败语义

这一章最后还需要把一个容易忽略的失败语义单独立起来。

如果一个包写的是：

```json
{
  "dependencies": {
    "@vben/utils": "workspace:*"
  }
}
```

那它表达的不是“优先本地，没有就算了”，而是：

- 本地必须存在这个 workspace 包
- 并且它必须满足当前声明
- 否则安装直接失败

这条失败语义非常重要。因为它使得 workspace 依赖从“最佳努力”变成“显式契约”。

这也是 Monorepo 中推荐大量使用 `workspace:` 的根本原因之一：问题会尽早暴露在安装阶段，而不是等到运行阶段才发现引用了错误来源。

### 8.15 本章结论

workspace 在 `pnpm` 中不是抽象概念，而是由 `pnpm-workspace.yaml` 显式定义的本地包管理边界。

这一章可以收敛为五个结论：

1. `pnpm-workspace.yaml` 决定 workspace 根和成员收集范围，`packages` 表达的是路径匹配规则，不是包名列表
2. 根包始终属于 workspace，因此整仓 lockfile 和命令行为都可以把根包作为 importer
3. workspace 真正的价值不在“目录多”，而在“本地包被纳入统一解析边界”
4. `workspace:` 协议不是语法糖，而是“只能解析到本地 workspace 包”的强约束
5. `workspace:*`、alias、相对路径等写法在发布前都会被替换成普通依赖声明，以保证发布包能被外部消费者正确安装

## 9. peerDependencies、分裂与 `injected`

这一章不再重复第 5 章“peerDependencies 是什么”的定义，而是只讨论一个更难的问题：

> peer 依赖在安装时到底怎么解析，为什么同一个版本的包会在同一个项目里出现多份实例。

如果这个问题不讲透，下面这些现象都会显得像“莫名其妙”：

- 明明包版本一样，`.pnpm` 里却出现多份实例
- 同一个库在不同消费者里表现不一致
- 类型系统出现多份实例不兼容
- Monorepo 本地包在不同应用里需要不同 peer 宿主

### 9.1 peer 解析不是“包自己带一份”，而是“向更高层取宿主”

第 5 章已经定义过 `peerDependencies` 是宿主兼容契约。本章只补它的解析规则。

官方关于 `How peers are resolved` 的说明给出的核心结论是：

> peer 依赖是从依赖图中更高的位置解析的，因为它需要与父级共享同一份宿主依赖。

这句话可以直接拆成两层含义：

- 一个声明了 peer 的包，不能把 peer 当成自己私有的一份普通依赖
- 它实际能拿到哪一份 peer，取决于它被放进了哪一个父级上下文

因此，peer 问题本质上不是“有没有装这个包”，而是：

> 这份包实例所在的上下文，最终把哪一份宿主依赖提供给了它。

### 9.2 为什么 peer 会让“同版本唯一实例”失效

前面第 7 章已经讲过，`pnpm` 的一个重要特征是：同一个项目里，一个包版本通常对应一组稳定依赖集合。

但官方文档也明确指出，这条规则有一个重要例外：

> 带 peer dependencies 的包，可能在同一个项目里对应多组不同的依赖集合。

原因很直接。

假设：

- `foo@1.0.0` 声明了两个 peer：
  - `bar@^1`
  - `baz@^1`

现在项目里存在两个不同父级：

- `foo-parent-1`
  - 提供 `bar@1.0.0`
  - 提供 `baz@1.0.0`
- `foo-parent-2`
  - 提供 `bar@1.0.0`
  - 提供 `baz@1.1.0`

那么虽然被安装的都是 `foo@1.0.0`，但它实际拿到的 peer 上下文并不相同：

- 一份 `foo@1.0.0` 工作在 `bar@1.0.0 + baz@1.0.0` 上下文里
- 另一份 `foo@1.0.0` 工作在 `bar@1.0.0 + baz@1.1.0` 上下文里

这时如果还强行只保留一份实例，就会破坏 peer 解析语义。于是 `pnpm` 必须为不同上下文生成不同实例。

### 9.3 `.pnpm` 里的“变体目录”就是 peer 上下文编码结果

官方 `How peers are resolved` 文档直接给出了这种结构：

```text
node_modules/.pnpm/
├── foo@1.0.0_bar@1.0.0+baz@1.0.0/
└── foo@1.0.0_bar@1.0.0+baz@1.1.0/
```

这两个目录并不是“重复安装失控”，而是在表达：

- 同一版本 `foo@1.0.0`
- 在两组不同 peer 解析结果下
- 形成了两份不同包实例

因此，看到 `.pnpm` 里同名同版但后缀不同的目录时，第一反应不应该是“pnpm 装重了”，而应该是：

> 当前项目里，这个包在不同 peer 上下文下被实例化了。

### 9.4 不是只有“自己声明了 peer”的包才会分裂

这一点非常关键，也是很多人第一次排查 peer 问题时最容易漏掉的地方。

官方文档专门强调了一种更隐蔽的情况：

> 即使某个包自己没有声明 peer，只要它依赖的下游包有 peer，并且这些 peer 在更高层解析，那么这个上游包也可能跟着出现多份实例。

官方给出的最小模型是：

- `a@1.0.0`
  - 依赖 `b@1.0.0`
- `b@1.0.0`
  - peer 依赖 `c@^1`

如果项目里存在两种不同的 `c` 上下文，例如：

- 一处解析到 `c@1.0.0`
- 另一处解析到 `c@1.1.0`

那么不仅 `b@1.0.0` 会分裂，`a@1.0.0` 也会跟着分裂，因为它依赖的是“带着不同 peer 结果的 b 实例”。

这说明 peer 分裂不是只影响 peer 包自己，而是可能沿着依赖链向上传播。

### 9.5 当前项目里的真实现象：同一个 `vite` 版本已经出现了多实例

当前 `daisy-ui` 的 [`node_modules/.pnpm`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/.pnpm) 中，就已经能直接看到同版本多实例的样本。

例如 `vite@8.2.2` 当前至少有两份实例：

```text
vite@8.2.2_@types+node@25.9.5_esbuild@0.28.2_jiti@2.7.0_less@4.9.0_supports-color@10.2._1d88c90ccb5404d5cb742f52e701d83b
vite@8.2.2_@types+node@26.2.0_esbuild@0.28.2_jiti@2.7.0_less@4.9.0_supports-color@10.2._6b4737a66a76ec097e450a8734e0c4ec
```

这里最值得注意的是：

- 包版本相同，都是 `vite@8.2.2`
- 但实例后缀不同
- 差异至少体现在 `@types/node` 上下文不同：
  - 一份是 `25.9.5`
  - 一份是 `26.2.0`

这正是实例分裂的典型信号：**包本身版本没变，但它参与解析的依赖上下文不同。**

需要注意的是，这个例子用于说明“同版多实例”现象本身。是否由直接 peer 声明触发，要结合该包及其依赖链继续分析；但从 `pnpm` 的实例目录命名上，已经可以确认当前项目存在多上下文实例化结果。

### 9.6 分裂本身不是 bug，关键在于是否超出预期

peer 分裂不应被简单理解成错误。更准确的判断方式是：

- **合理分裂**
  - 不同消费者确实需要不同宿主上下文
  - 多份实例是依赖模型的真实结果
- **异常分裂**
  - 团队本来希望共享同一份宿主
  - 却因为依赖声明、版本收敛、治理策略失控，导致出现了多份实例

也就是说，问题不在“有没有多份”，而在：

- 这些多份实例是不是预期内的
- 是否影响运行时共享语义
- 是否影响类型系统、单例、上下文、插件挂载点

### 9.7 什么时候分裂会真正变成工程问题

下面几类场景里，peer 分裂通常会从“可接受现象”变成“需要治理的问题”。

**运行时需要共享单例**

例如：

- React / Vue 运行时
- Router
- 状态管理器
- 上下文容器
- 插件宿主对象

如果这些对象被拆成多份实例，就容易出现：

- 插件挂不上去
- context 读不到
- store 不共享
- `instanceof`、符号比较、全局状态判断失真

**类型系统要求唯一来源**

TypeScript 场景下，多份同名库实例很容易带来：

- 相同名字但不同来源的类型不兼容
- 泛型约束无法统一
- 声明文件中的符号无法相互赋值

**构建工具链依赖宿主一致性**

插件类生态尤其明显：

- ESLint 插件与 ESLint 主体
- Vite / Rollup / Webpack 插件与宿主版本
- Vue / React 生态组件与宿主运行时

这类场景里，peer 关系本来就是为了保证“挂在宿主上工作”，一旦宿主不一致，问题就会很快暴露。

### 9.8 常见症状

peer 分裂在工程里通常不会直接报“peer split”这种好懂错误，而是通过症状暴露。

常见症状包括：

- 同一个包在 `.pnpm` 下出现多份同版不同后缀实例
- `pnpm why <pkg> -r` 显示多个不同来源链路
- 运行时出现上下文不共享、单例失效
- 类型报错提示“不是同一个类型”或“来自不同声明源”
- 某些插件在一个子项目里正常，在另一个子项目里失效
- lockfile 中同一包出现多组带上下文后缀的解析结果

### 9.9 排查顺序

peer 问题如果上来就看代码，通常效率很低。更稳的排查顺序是：

**先看 `.pnpm` 目录里有没有多实例**

例如直接查：

```shell
ls node_modules/.pnpm | rg '^vite@8\.2\.2'
```

如果同一版本出现多条结果，就已经说明存在多上下文实例化。

**再看 lockfile 里的 importer 与 packages 关系**

前面第 6 章已经说明，lockfile 不只是版本表。这里要查的是：

- 哪些 importer 引入了相关包
- 这些 importer 所在的依赖图上下文是否一致

**再用 `pnpm why` 看反向依赖链**

例如：

```shell
pnpm why vite -r
```

它能帮助定位：

- 谁把这个包引进来
- 是哪些项目或包在消费它

**最后再判断是“应该合并”还是“应该隔离”**

这是最关键的一步。不是所有多实例都该消灭，有些场景本来就需要隔离。

### 9.10 `injected` 解决的到底是什么问题

前面讲的是“外部依赖在不同 peer 上下文中会分裂”。  
`injected` 要解决的是另一个更具体的问题：

> 在 Monorepo 里，一个本地 workspace 包被多个消费者依赖时，如何让它在不同消费者中获得不同的 peer 宿主上下文。

这是 `dependenciesMeta.<dep>.injected` 存在的根本原因。

如果不启用 `injected`，官方文档说明默认行为是：

- 对于本地 workspace package
- 消费者的 `node_modules/<dep>` 会是一个 symlink
- 直接指向 workspace 中该包的源码目录

这种模式的优点是快，而且源码修改会立即对消费者可见。  
但它有一个硬边界：

> symlink 到同一份源码目录时，无法让同一个本地包在不同消费者里分别获得不同的 peer 宿主解析结果。

### 9.11 官方最小案例：`button` 同时被 `card` 和 `form` 使用

官方 `package.json` 文档给出的案例非常典型。

假设：

- `button` 是 workspace 本地包
- `button` 的 `peerDependencies` 里要求 `react`
- `card` 依赖 `button`，并使用 `react@16`
- `form` 依赖 `button`，并使用 `react@17`

如果 `button` 只是以普通 symlink 方式暴露给两个消费者，那么它始终指向同一份 workspace 源目录。这样一来：

- `card` 视角下的 `button`
- `form` 视角下的 `button`

无法分别稳定拿到不同版本的 `react` 宿主上下文。

这正是 `injected` 出场的地方。

### 9.12 `dependenciesMeta.<dep>.injected` 的配置位置与行为

`injected` 的位置前面第 5 章已经定过，这里只补它的行为。

它写在**消费者包自己的 `package.json`** 中，例如：

```json
{
  "name": "card",
  "dependencies": {
    "button": "workspace:1.0.0",
    "react": "16"
  },
  "dependenciesMeta": {
    "button": {
      "injected": true
    }
  }
}
```

另一个消费者 `form` 则是：

```json
{
  "name": "form",
  "dependencies": {
    "button": "workspace:1.0.0",
    "react": "17"
  },
  "dependenciesMeta": {
    "button": {
      "injected": true
    }
  }
}
```

官方文档对它的行为定义很明确：

- 当 `injected: true`
- 且目标依赖是本地 workspace package

该依赖会在 virtual store（`node_modules/.pnpm`）中以 hard linked copy 的形式安装出来，而不是仅仅用一个指向 workspace 源目录的 symlink 暴露给消费者。

结果就是：

- `card` 里的 `button`
  - 可以在 `card` 的上下文里解析到 `react@16`
- `form` 里的 `button`
  - 可以在 `form` 的上下文里解析到 `react@17`

这时同一个本地 workspace 包，就能在不同消费者里形成不同上下文实例。

### 9.13 `injected` 不是 link 模式的升级版，而是隔离手段

这一点必须单独立住。

`injected` 不应被理解成“比 symlink 更高级”。更准确的理解是：

- 默认 symlink 模式
  - 更快
  - 修改即时可见
  - 适合所有消费者共享同一份本地包落点
- `injected` 模式
  - 为消费者生成进入 virtual store 的隔离实例
  - 允许不同消费者拥有不同宿主上下文
  - 代价是同步更新更复杂

因此，`injected` 不是 workspace 默认模式的替代品，而是：

> 当多个消费者需要对同一本地包形成不同 peer 上下文时，使用的隔离安装手段。

### 9.14 `injected` 的代价

官方文档对这一点讲得很直接：因为 injected dependency 会产生 workspace 源目录的副本，这些副本在源码变化后需要被更新，否则消费者看到的不会是最新状态。

这意味着启用 `injected` 后，要额外面对一个工程问题：

- 被注入包重建之后
- 消费者看到的实例如何同步更新

官方文档给出的最小建议包括：

- 简单场景下重新执行一次 `pnpm install`
- 或者用诸如 `"prepare": "pnpm run build"` 这类生命周期脚本保证该包先重建

官方文档还提到了专门同步 injected 依赖的第三方工具，但这一章只保留最重要的结论：

> `injected` 解决的是上下文隔离问题，但会把“源码变更如何同步到消费者实例”变成新的工程成本。

### 9.15 当前项目里为什么没有直接使用 `injected`

当前 `daisy-ui` 项目源码中，没有直接声明 `dependenciesMeta.<dep>.injected`。

这并不奇怪。它通常说明以下两种情况之一：

- 当前项目里，本地 workspace 包之间暂时没有出现“同一本地包必须在不同消费者里拿到不同 peer 宿主”的刚性需求
- 或者这类问题通过版本收敛、包职责拆分、消费者约束等方式规避掉了

也就是说，`injected` 不是 Monorepo 必配项。只有当：

- 本地共享包存在 peer 宿主要求
- 且多个消费者必须给它不同宿主上下文

它才会真正成为必要工具。

### 9.16 本章结论

peer 问题的核心，不在于“某个包有没有被安装”，而在于“它最终处在哪个宿主上下文里”。

这一章可以收敛为六个结论：

1. peer 依赖从更高层依赖图中解析，因此同一个包版本可能在不同宿主上下文中形成不同实例
2. 同版多实例并不自动等于错误，关键在于这些差异是否超出工程预期
3. 即使某个包自己没有声明 peer，只要它依赖链上的下游包有 peer，它也可能跟着分裂
4. `.pnpm` 中同名同版不同后缀目录，是识别 peer 分裂最直接的信号之一
5. `dependenciesMeta.<dep>.injected` 用于让本地 workspace 包在不同消费者中形成隔离实例，以便分别解析不同 peer 宿主
6. `injected` 解决的是上下文隔离，不是默认 link 模式的普遍替代；它会引入额外的同步维护成本

## 10. 依赖治理配置

这一章不讨论“依赖如何被解析”，而讨论“团队如何主动治理依赖图”。

前面第 8、9 章讲的是：

- workspace 如何把本地包纳入统一边界
- peer 如何让同版包形成不同实例

本章换一个视角，专门回答：

> 当依赖图已经存在时，团队能在哪些层面主动施加治理。

`pnpm` 在这件事上提供的能力不少，但最容易混淆的是下面四类：

- `catalog` / `catalogs`
  - 管理版本入口
- `overrides`
  - 改写最终解析结果
- `packageExtensions`
  - 补包清单元数据
- `patchedDependencies`
  - 直接补上游包源码

如果不把这四层拆开，后面几乎所有治理配置都会被误用。

### 10.1 先分清治理层级

这四类能力处理的不是同一层问题。

**第一层：版本入口治理**

解决的问题是：

- 同一个依赖在很多包里反复声明
- 升级时需要改很多 `package.json`
- 多人改同一依赖时容易产生大量 merge conflict

这一层主要由 `catalog` / `catalogs` 处理。

**第二层：最终解析结果治理**

解决的问题是：

- 依赖图里某个包必须统一成指定版本
- 上游依赖链把错误版本带进来
- 需要把某个包替换成 fork 或别的来源
- 某个依赖可以整体移除

这一层主要由 `overrides` 处理。

**第三层：包清单元数据治理**

解决的问题是：

- 某个上游包的 `package.json` 声明不完整
- 少写了 `peerDependencies`
- 少写了 `optionalDependencies`
- `peerDependenciesMeta` 有缺失

这一层主要由 `packageExtensions` 处理。

**第四层：源码级治理**

解决的问题是：

- 包的运行时代码本身有问题
- 构建脚本有 bug
- 需要临时改一段源码才能继续推进

这一层主要由 `patchedDependencies` 处理。

四者的边界一定要记住：

- `catalog` 管的是“入口怎么写”
- `overrides` 管的是“最终装成什么”
- `packageExtensions` 管的是“这个包在声明层面应该长什么样”
- `patchedDependencies` 管的是“这个包的源码要怎么改”

### 10.2 `catalog`：把版本范围提到统一入口

官方文档对 catalogs 的定义很明确：

> Catalogs 是 workspace feature，用来把依赖版本范围定义成可复用常量。

最简单的形式就是 `pnpm-workspace.yaml` 里的默认 catalog：

```yaml
catalog:
  react: ^18.3.1
  redux: ^5.0.1
```

之后各个 `package.json` 中可以直接写：

```json
{
  "dependencies": {
    "react": "catalog:",
    "redux": "catalog:"
  }
}
```

这等价于直接写版本范围，但语义不同。它表达的是：

- 版本范围的维护权上移到 workspace 级配置
- 各个包只声明“我要用这个统一版本入口”

因此，`catalog` 解决的不是安装机制，而是版本入口重复和版本漂移问题。

### 10.3 为什么 `catalog` 比在每个 `package.json` 里手写版本更稳

官方文档给出的几个优势非常直接：

- 维护单一版本更容易
- 升级时只改一处
- `package.json` 冲突更少

把这几条翻译成工程语义，就是：

- 同一个 workspace 里，公共依赖的版本决策集中到 `pnpm-workspace.yaml`
- 业务包不再各自维护一份版本范围
- 升级动作更接近“改版本表”，而不是“全仓批量替换”

这也是为什么 `catalog` 非常适合：

- UI 框架运行时
- 构建工具
- 类型包
- 大量 workspace 共享的基础依赖

### 10.4 当前项目里的 `catalog` 用法

当前 `daisy-ui` 的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 使用了一个非常大的默认 `catalog`，里面集中管理了：

- `vue`
- `vite`
- `pinia`
- `@types/node`
- `eslint`
- `stylelint`
- `@vitejs/plugin-vue`
- `tailwindcss`
- 以及大量其他基础依赖

而各个 workspace 包里则直接写 `catalog:`。例如：

- [apps/web-antd/package.json](file:///Users/apple/code/WebProjects/daisy-ui/apps/web-antd/package.json) 中的 `vue`、`pinia`、`dayjs`、`ant-design-vue`
- [packages/stores/package.json](file:///Users/apple/code/WebProjects/daisy-ui/packages/stores/package.json) 中的 `pinia`、`secure-ls`、`vue-router`
- 根 [package.json](file:///Users/apple/code/WebProjects/daisy-ui/package.json) 中的 `@types/node`、`cross-env`、`@vitejs/plugin-vue`

这套写法的结果是：

- 包内依赖声明更短
- 版本决策点更集中
- 升级公共依赖时，不需要扫全仓 `package.json`

### 10.5 `catalog` 能写在哪些位置

官方文档明确说明，`catalog:` 协议可以用在：

- `package.json`
  - `dependencies`
  - `devDependencies`
  - `peerDependencies`
  - `optionalDependencies`
- `pnpm-workspace.yaml`
  - `overrides`

这一点很重要。因为它说明 `catalog` 不只是写在普通依赖声明里，还可以继续参与更高一层的治理配置。

当前项目里就有一个典型例子：

```yaml
overrides:
  vue: 'catalog:'
  pinia: 'catalog:'
```

这意味着：

- 最终解析要统一走 override
- override 又不手写版本号
- 而是继续指向 catalog 中定义的统一版本入口

这是一种很稳的组合方式。

### 10.6 `catalogs`：给版本入口分组

`catalog` 是默认 catalog。  
`catalogs` 则允许在同一个 workspace 里定义多个命名版本表。

官方文档给出的典型形式是：

```yaml
catalogs:
  react17:
    react: ^17.0.2
    react-dom: ^17.0.2
  react18:
    react: ^18.2.0
    react-dom: ^18.2.0
```

然后在 `package.json` 中使用：

```json
{
  "dependencies": {
    "react": "catalog:react18"
  }
}
```

这类能力适合的不是“所有包都完全统一版本”的场景，而是：

- 大仓分批迁移主版本
- 不同产品线需要不同宿主版本
- 兼容过渡期需要并存多个版本入口表

当前 `daisy-ui` 没有启用 `catalogs`，只使用了默认 `catalog`。这说明当前项目更偏向：

- 大部分共享依赖集中收敛到单一版本入口
- 暂时没有通过命名 catalog 并行维护多组大版本基线

### 10.7 `catalog:` 不会原样发布出去

官方文档明确说明：

- `pnpm publish`
- `pnpm pack`

执行时都会移除 `catalog:` 协议，把它替换成普通版本范围。

例如：

```json
{
  "dependencies": {
    "react": "catalog:react18"
  }
}
```

发布后会变成类似：

```json
{
  "dependencies": {
    "react": "^18.3.1"
  }
}
```

这和第 8 章讲过的 `workspace:` 替换是同一类边界转换：

- 仓库内
  - 使用内部协议维护治理能力
- 仓库外
  - 替换成普通包管理器可理解的标准声明

因此，`catalog:` 也是 workspace 内部协议，而不是对外发布格式。

### 10.8 `catalogMode`：`pnpm add` 如何对待 catalog

官方 catalogs 文档还定义了 `catalogMode`，用来控制执行 `pnpm add` 时，依赖是如何进入默认 catalog 的。

它的取值包括：

- `manual`
  - 默认值
  - 不自动把依赖写入 catalog
- `strict`
  - 只允许使用 catalog 中的版本
  - 超出 catalog 范围会报错
- `prefer`
  - 优先使用 catalog 版本
  - 若没有兼容版本，再退回直接依赖

这个配置影响的是“新增依赖时的治理入口”，而不是安装结果本身。

当前项目没有显式设置 `catalogMode`，因此按官方默认值 `manual` 处理。

### 10.9 `catalogPrune`：清理没有再被引用的 catalog 条目

官方文档说明：

- `catalogPrune` 自 `v11.22.0` 起可用
- 默认值是 `false`
- 打开后，安装时会移除未被使用的 catalog 条目

它解决的问题不是解析正确性，而是治理整洁度。

长期维护的大仓里，catalog 很容易出现这种情况：

- 某些依赖已经不再被任何包引用
- 但它们还残留在 `catalog` 里
- 随着时间推移，版本表越来越长，真实活跃项越来越难识别

`catalogPrune` 就是在安装阶段自动做这件事：

- 检查哪些 catalog 条目已经没有引用
- 把这些未使用项从 catalog 中清掉

当前项目没有启用 `catalogPrune`。这意味着 catalog 的清理更可能依赖人工维护，而不是交给安装过程自动收口。

### 10.10 `overrides`：改写最终依赖图

如果说 `catalog` 管的是“版本入口”，那 `overrides` 管的就是“最终解析结果”。

官方文档对 `overrides` 的定义是：

> instruct pnpm to override any dependency in the dependency graph, including peer dependencies

这句话很关键，它说明 `overrides` 的作用范围不是“只改直接依赖”，而是：

- 整张依赖图
- 包括 peerDependencies 在内

它适合的场景包括：

- 强制所有包统一到同一个版本
- 给上游依赖链回灌一个修复版本
- 把某个依赖替换成 fork
- 删掉实际上不需要的依赖

### 10.11 `overrides` 的几个核心写法

官方文档给出的几个模式非常值得直接记住。

**直接把某个包统一到指定版本**

```yaml
overrides:
  foo: ^1.0.0
```

**替换成另一个包来源**

```yaml
overrides:
  quux: npm:@myorg/quux@^1.0.0
```

**只覆盖特定版本范围**

```yaml
overrides:
  "bar@^2.1.0": "3.0.0"
```

**只覆盖某个父包下面的子依赖**

```yaml
overrides:
  "qar@1>zoo": "2"
```

这类 `parent>child` 语法非常重要。它意味着 `overrides` 不只是全局大锤，还可以精确指到某条依赖边。

### 10.12 `overrides` 只能在根部配置，并且还能引用直接依赖规范

官方文档明确说明：

> `overrides` 只能设置在项目根部。

这条规则非常重要，因为它决定了 `overrides` 的治理层级是整个 workspace / 项目根，而不是某个子包各自定义一套局部 override 规则。

官方文档还提供了一种很实用的写法：用 `$` 引用直接依赖的 spec。

例如根 `package.json` 里已经声明：

```json
{
  "dependencies": {
    "foo": "^1.0.0"
  }
}
```

那么在 `pnpm-workspace.yaml` 里可以写：

```yaml
overrides:
  bar: "$foo"
```

它表达的含义是：

- `bar` 的 override 结果
- 直接复用当前直接依赖 `foo` 的 spec

这种写法的意义在于复用版本决策来源，避免在多个治理位置重复写死同一份规范。

### 10.13 `overrides` 也可以删除依赖

官方文档明确支持用 `-` 删除某条依赖边：

```yaml
overrides:
  "foo@1.0.0>bar": "-"
```

这适合的场景通常是：

- 某个依赖实际上不被使用
- 它体积很大
- 或者它只是一个可选依赖分支

文档特别提到，这个能力在 `optionalDependencies` 上尤其有用。

因此，`overrides` 不只是“改版本”，也能做“从依赖图里移走某一条边”。

### 10.14 `overrides` 对 peer 也生效

这是 `overrides` 最容易被低估的地方。

官方文档明确说明，`overrides` 会作用于 `peerDependencies`。但行为取决于 override 值的类型：

- semver range / `workspace:` / `catalog:`
  - 仍然保持 peer 语义
- `link:` / `file:` 这类非 range specifier
  - 会把 peer 改写进 `dependencies`
- `-`
  - 直接移除该 peer

这条规则非常关键，因为它说明：

- `overrides` 不只是“普通依赖改版本”
- 它可能直接改变 peer 的最终表现层

### 10.15 当前项目里的 `overrides`

当前 `daisy-ui` 的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 中，`overrides` 配置是：

```yaml
overrides:
  '@ast-grep/napi': 'catalog:'
  '@ctrl/tinycolor': 'catalog:'
  clsx: 'catalog:'
  pinia: 'catalog:'
  vue: 'catalog:'
```

这组配置反映出当前项目的治理思路非常清楚：

- 对某些关键依赖，不只是在声明层统一入口
- 还要在最终解析层明确统一结果
- 而统一结果仍然回指到 catalog 中维护的公共版本表

换句话说，这个项目不是只靠“大家自觉都写 `catalog:`”来收敛，而是：

- `catalog` 负责统一入口
- `overrides` 负责兜底最终解析结果

### 10.16 `packageExtensions`：补上游包的清单声明

`packageExtensions` 不是为了改版本，也不是为了改源码。它的职责更窄，也更重要：

> 当某个上游包的 `package.json` 写错了或写漏了，用 `packageExtensions` 把清单元数据补齐。

官方文档给出的典型场景是：

- `react-redux` 本来应该声明 `react-dom` 为 `peerDependencies`
- 但它没有写

这时可以在 `pnpm-workspace.yaml` 中补：

```yaml
packageExtensions:
  react-redux:
    peerDependencies:
      react-dom: "*"
```

它处理的不是运行时代码 bug，而是包元数据声明缺陷。

### 10.17 `packageExtensions` 能补哪些字段

官方文档明确列出，`packageExtensions` 可扩展的字段包括：

- `dependencies`
- `optionalDependencies`
- `peerDependencies`
- `peerDependenciesMeta`

也就是说，它能做的事情主要是：

- 给包补普通依赖
- 补可选依赖
- 补 peer 依赖
- 给 peer 再补 optional 等元信息

但它不能替代源码补丁，也不是通用重写器。

### 10.18 `packageExtensions` 的适用边界

`packageExtensions` 适合处理的是：

- 清单缺项
- peer 声明缺失
- optional 标记缺失
- 依赖元数据不完整

它不适合处理的是：

- 运行时代码 bug
- 需要修改构建脚本
- 需要替换实际依赖实现

因此，遇到上游包问题时，先问清楚：

- 是 `package.json` 声明错了
- 还是包源码本身错了

前者优先考虑 `packageExtensions`，后者才考虑 patch。

当前 `daisy-ui` 没有启用 `packageExtensions`。这意味着当前项目暂时没有通过这种方式修补上游包声明缺陷。

### 10.19 `patchedDependencies`：直接补上游包源码

当问题已经不在清单层，而在源码层时，才轮到 `patchedDependencies`。

官方 `pnpm patch` 文档给出的标准流程是：

1. `pnpm patch <pkg>@<version>`
2. 在临时目录修改包内容
3. 执行 `pnpm patch-commit <path>`
4. 生成 patch 文件，并把记录写入顶层清单中的 `patchedDependencies`

例如：

```yaml
patchedDependencies:
  express@4.18.1: patches/express@4.18.1.patch
```

这里记录的是：

- 哪个包
- 哪个版本或范围
- 应用哪一个 patch 文件

### 10.20 `patchedDependencies` 不是拿来改依赖关系的

这一点官方文档专门强调过：

> 如果想改某个包的依赖关系，不要用 patch 去修改它的 `package.json`，应优先使用 `overrides` 或 package hook。

这条边界非常重要。因为 patch 很强，但正因为强，才更容易被滥用。

正确的分工应该是：

- 依赖图收敛、替换、删除
  - `overrides`
- 清单元数据补齐
  - `packageExtensions`
- 源码行为修补
  - `patchedDependencies`

### 10.21 `patchedDependencies` 的匹配粒度

官方文档说明，`patchedDependencies` 的 key 可以是：

- 精确版本
- 版本范围
- 只有包名

优先级顺序是：

1. 精确版本
2. 版本范围
3. 只有包名

因此，patch 不是只能精确绑死到单一版本，也可以为一段版本范围提供统一 patch。但如果版本范围重叠过多，治理复杂度也会迅速上升。

### 10.22 当前项目里没有启用 `patchedDependencies`

当前 `daisy-ui` 仓库中没有看到：

- `patchedDependencies` 配置
- `patches/` 目录

这说明当前项目至少在当前状态下，没有把“直接 patch 上游包源码”作为主要治理手段。

这通常意味着两种情况之一：

- 当前上游依赖问题主要通过版本收敛和配置治理解决
- 或者项目刻意避免引入源码补丁维护成本

### 10.23 四类治理能力如何选

这一章最后需要把选择原则说死，否则配置项虽然都认识了，实际还是容易乱用。

如果问题是：

- **很多包都在重复写同一个版本范围**
  - 优先用 `catalog` / `catalogs`
- **最终安装结果必须统一、替换或删掉某条依赖**
  - 优先用 `overrides`
- **上游包 `package.json` 声明不完整**
  - 优先用 `packageExtensions`
- **上游包源码本身有问题**
  - 优先用 `patchedDependencies`

这个顺序本质上是在尽量选择“离问题最近、改动面最小”的治理手段。

### 10.24 本章结论

依赖治理配置的核心不是“配置项很多”，而是每一类配置都在修改不同层面的事实。

这一章可以收敛为六个结论：

1. `catalog` / `catalogs` 解决的是版本入口集中治理，不直接改写最终依赖图
2. `catalogPrune` 解决的是版本表长期维护时的陈旧条目清理问题
3. `overrides` 作用于最终依赖图，能统一版本、替换来源、删除依赖，并且对 peer 也生效
4. `packageExtensions` 用于补上游包的清单元数据，不用于修改源码
5. `patchedDependencies` 用于修补上游包源码，不应用来替代依赖关系治理
6. 当前 `daisy-ui` 的治理重点放在 `catalog` + `overrides`，而不是 `packageExtensions` 或源码 patch

## 11. 布局兼容与解析策略

这一章处理的是默认模型之外的几个关键开关。

前面几章已经分别讲过：

- 第 7 章：`pnpm` 默认的磁盘布局如何成立
- 第 8 章：workspace 如何定义本地包边界
- 第 9 章：peer 如何导致同版多实例

但真实工程里，默认模型不是总能直接落地。生态里仍然存在大量：

- 对 phantom dependency 有依赖的旧工具
- 不善于处理 symlink 的运行环境
- peer 声明不干净的包
- 本地 `node_modules` 容易漂移的开发流

所以这一章要回答的是：

> 当默认严格模型与工程兼容性冲突时，`pnpm` 提供了哪些布局与解析策略开关，这些开关到底各自在改哪一层。

### 11.1 先把这一章的对象分成两类

这一章里的配置项虽然都写在 `pnpm-workspace.yaml`，但它们不是同一类东西。

**第一类：布局兼容开关**

它们改变的是：

- 依赖入口会暴露到哪里
- `node_modules` 的可见性边界
- 是否继续保留 `pnpm` 默认的 virtual store + symlink 结构

这一类主要包括：

- `hoist`
- `publicHoistPattern`
- `nodeLinker`

**第二类：peer 与运行前策略开关**

它们改变的是：

- peer 缺失时是否自动补装
- peer 有问题时是报错还是继续
- 不同项目之间是否尽量共享 peer 解析结果
- 运行脚本前是否检查 `node_modules` 是否过期

这一类主要包括：

- `strictPeerDependencies`
- `autoInstallPeers`
- `dedupePeerDependents`
- `verifyDepsBeforeRun`

如果不先把这两类拆开，就很容易把“目录布局问题”和“peer 解析策略问题”混成一章糊在一起。

### 11.2 `hoist`：把依赖提升到隐藏提升层

官方 settings 文档对 `hoist` 的定义非常直接：

- 默认值：`true`
- 含义：当它为 `true` 时，依赖会被 hoist 到 `node_modules/.pnpm/node_modules`

这句话有两个关键点。

第一，`hoist` 不是把依赖直接提升到项目根 `node_modules`。  
第二，它提升到的是一个**隐藏提升层**：

```text
node_modules/.pnpm/node_modules
```

这正是当前项目里实际存在的目录：

- [`node_modules/.pnpm/node_modules`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules/.pnpm/node_modules)

这层目录的作用，是在不彻底放弃 `pnpm` 默认隔离模型的前提下，给生态里那些“会偷拿未声明依赖”的包留一个缓冲层。

### 11.3 为什么 `hoist` 默认就是 `true`

很多人第一次接触 `pnpm` 时会有一个误解：既然它强调依赖边界，那默认应该完全不 hoist。

官方文档没有这么设计，原因很现实。

如果完全不 hoist，那么生态里那些隐式依赖较多、解析写得不规范的包，会更频繁地直接报错。为了降低切换成本，`pnpm` 默认选择的是一种 **semistrict** 布局：

- `node_modules` 外的应用代码
  - 不能随便捡到 phantom dependency
- `node_modules` 内部的一些包
  - 仍然可能通过隐藏提升层拿到未声明依赖

官方对 `shamefullyHoist` 的说明里明确把这种默认布局称为 semistrict `node_modules`。

所以，默认 `hoist: true` 的目的不是“更宽松更先进”，而是：

> 在隔离和兼容之间取一个工程上更能落地的折中点。

### 11.4 `hoist` 改的是隐藏层，不是根暴露层

这一点必须单独立住，因为它正是很多人第一次看到这些配置时最容易混的地方。

- `hoist`
  - 目标位置是 `node_modules/.pnpm/node_modules`
- `publicHoistPattern`
  - 目标位置是项目根 `node_modules`

两者都叫 hoist，但语义层级不同：

- 前者是隐藏提升
- 后者是公开提升

把这两者混在一起，就会误以为“只要开启 hoist，根 `node_modules` 就会重新变成 npm/Yarn Classic 那样的平铺目录”。这并不准确。

### 11.5 `publicHoistPattern`：只把指定包公开提升到根入口层

官方文档对 `publicHoistPattern` 的定义是：

- 默认值：`[]`
- 它会把匹配模式的依赖 hoist 到**根 modules directory**

这里的“根 modules directory”就是项目根 `node_modules`。  
这和 `hoist` 的隐藏提升层是两回事。

官方还特别强调了风险边界：

> 提升到根模块目录后，应用代码就可能访问到 phantom dependencies，即使这些代码本来不应该按正确解析策略拿到这些依赖。

因此，`publicHoistPattern` 不应被理解成“优化安装结构”，而应理解成：

> 为了兼容某些解析写得不规范的工具，只把指定包公开暴露到根入口层。

### 11.6 当前项目为什么配置了 `publicHoistPattern`

当前 `daisy-ui` 的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 中配置了：

```yaml
publicHoistPattern:
  - lefthook
  - eslint
  - oxfmt
  - oxlint
  - stylelint
  - '*postcss*'
  - '@commitlint/*'
  - czg
```

这组模式非常说明问题。它们几乎都不是业务运行时依赖，而是：

- lint 工具
- format 工具
- commit / hook 工具
- postcss 相关工具链

也就是说，当前项目把 `publicHoistPattern` 用在了一个非常典型的位置：

> 让工具链相关包更容易从项目根脚本环境、hook 环境或插件装配环境中被访问到。

这和把业务运行时库大面积公开提升到根目录，是两种完全不同的治理思路。

### 11.7 当前项目里可以直接看到公开提升的结果

当前 `daisy-ui` 根 [`node_modules`](file:///Users/apple/code/WebProjects/daisy-ui/node_modules) 中，可以直接看到这些入口：

```text
czg -> .pnpm/czg@1.14.0/node_modules/czg
eslint -> .pnpm/eslint@10.9.0_jiti@2.7.0_supports-color@10.2.2/node_modules/eslint
lefthook -> .pnpm/lefthook@2.1.10/node_modules/lefthook
postcss -> .pnpm/postcss@8.5.26/node_modules/postcss
postcss-html -> .pnpm/postcss-html@1.8.1/node_modules/postcss-html
postcss-scss -> .pnpm/postcss-scss@4.0.9_postcss@8.5.26/node_modules/postcss-scss
stylelint -> .pnpm/stylelint@17.14.1_supports-color@10.2.2_typescript@6.0.3/node_modules/stylelint
```

这批包和当前 `publicHoistPattern` 的匹配关系是对得上的。

需要注意的是，根入口里看到某个包，不自动等于“它一定只是因为 public hoist 才在这里”。有些包本来也可能是根直接依赖。真正稳定的判断方式是：

- 看配置是否命中该模式
- 看它是否属于工具链包
- 再结合根清单和实际解析链路判断

但至少从当前项目的配置和根入口现象可以确认：

- 这个仓库确实在用 `publicHoistPattern` 为工具链暴露根级入口

### 11.8 `shamefullyHoist` 不应作为默认思路

官方文档明确写了：

> `shamefullyHoist: true` 等价于把 `publicHoistPattern` 设成 `*`

这意味着什么很清楚：

- 所有依赖都被公开提升到根 `node_modules`
- phantom dependency 的可见性会大幅扩大

从兼容性上看，它确实省心；从依赖边界上看，它几乎是在主动放弃 `pnpm` 默认模型的优势。

因此，工程里更稳的顺序通常应该是：

1. 先用默认布局
2. 真有工具兼容问题时，优先用小范围 `publicHoistPattern`
3. 只有当生态约束极强、根本无法细粒度治理时，才考虑 `shamefullyHoist`

当前项目也明显没有走“全部公开提升”这条路，而是只公开提升一小批工具链相关依赖。

### 11.9 `nodeLinker`：决定整个安装模型的大方向

如果说 `hoist` / `publicHoistPattern` 还只是对默认模型做局部调节，那么 `nodeLinker` 改的就是更上层的总开关。

官方文档定义如下：

- 默认值：`isolated`
- 可选值：
  - `isolated`
  - `hoisted`
  - `pnp`

这三个值代表的是三种不同的安装布局方向，而不是同一模型上的小修小补。

### 11.10 `nodeLinker=isolated`：默认的 `pnpm` 风格

`isolated` 就是前面第 7 章整章讲的那套默认布局：

- virtual store 在 `node_modules/.pnpm`
- 根入口大量使用 symlink
- 包实例通过实例层 `node_modules` 连成依赖图

当前 `daisy-ui` 没有显式配置 `nodeLinker`，因此按官方默认值 `isolated` 工作。

这意味着当前项目的真实策略其实是：

- 安装总模型仍然是 `pnpm` 默认隔离模型
- 只是在这个模型上额外叠加了：
  - 隐藏 hoist
  - 部分 public hoist
  - 若干 peer 与运行前策略开关

所以当前项目不是“改成 npm 风格了”，而是：

> 仍然站在 `isolated` 模型上，只对兼容性做局部调参。

### 11.11 `nodeLinker=hoisted`：回到平铺 `node_modules`

官方文档对 `hoisted` 的描述非常直接：

- 会创建一个没有 symlink 的平铺 `node_modules`
- 效果接近 npm 或 Yarn Classic

官方给出的两个典型使用理由也很明确：

1. 工具链不能很好处理 symlink  
   例如 React Native 项目
2. 部署目标环境不支持 symlink  
   例如某些 serverless 平台

这说明 `nodeLinker=hoisted` 不是“更简单就更推荐”，而是：

> 当目标运行环境或工具链对 symlink 支持很差时，主动切换到另一套兼容布局。

它解决的是“环境不吃 symlink”问题，而不是普通的工具链 phantom dependency 问题。

### 11.12 `nodeLinker=pnp`：走另一套解析机制

`pnp` 不是“比 hoisted 更强的 hoist”，而是彻底换一套链接与解析模式。

这一份文档主线以当前项目和 `pnpm 11` 的常见 Node.js 工程用法为主，因此这里只保留边界，不展开实现细节：

- `pnp` 不再以当前章节前面讲的 `node_modules/.pnpm` 结构作为主模型
- 它适合的不是普通前端仓库默认选型

当前项目没有使用 `pnp`，因此本书也不把它当成主线方案。

### 11.13 `strictPeerDependencies`：peer 不满足时是否直接失败

官方定义非常清楚：

- 默认值：`false`
- 当开启时，如果依赖树中存在缺失或无效的 peer dependency，命令会直接失败

也就是说，这个开关控制的不是“如何解析 peer”，而是：

> 当 peer 关系不自洽时，系统是把它当成告警，还是当成硬错误。

当前项目显式配置了：

```yaml
strictPeerDependencies: false
```

这和官方默认值一致。它表达的是：

- 当前仓库不把 peer 问题一律升级成安装失败
- 更倾向于在兼容性和可推进性之间保留缓冲空间

这并不代表 peer 不重要，而是说明当前项目没有选择最严格的失败策略。

### 11.14 `autoInstallPeers`：缺失的非 optional peer 是否自动补装

官方对 `autoInstallPeers` 的定义是：

- 默认值：`true`
- 当为 `true` 时，缺失的非 optional peer dependency 会被自动安装

官方还补了两个关键边界：

- 自动挑选的版本是满足 peer range 的最高版本
- 如果不同包对同一个 peer 给出了冲突版本要求，`pnpm` 不会自动装任何一个版本，而是打印 warning

因此，`autoInstallPeers` 不是“帮忙把所有 peer 问题都解决掉”，它只处理：

- 缺失
- 且没有冲突

当前项目显式配置：

```yaml
autoInstallPeers: true
```

这与官方默认值一致，说明当前仓库接受“对无冲突的缺失 peer 自动补装”这条策略。

### 11.15 `dedupePeerDependents`：尽量让不同项目共享同一份 peer 结果

这是第 9 章之后非常值得接上的一个开关。

官方定义如下：

- 默认值：`true`
- 当为 `true` 时，带 peer dependencies 的包在 peer 解析完成后会尽量被去重

官方给出的典型例子是 `webpack`：

- 一个项目有 `esbuild`
- 另一个项目没有 `esbuild`
- 于是 `webpack` 可能本来会形成两份实例：
  - 一份带 `esbuild`
  - 一份不带

而开启 `dedupePeerDependents` 后，如果这些差异不构成真正冲突，`pnpm` 会尽量让两个项目共享同一份实例。

这条策略的核心不是“强行全局只留一份”，而是：

> 在不违反 peer 冲突语义的前提下，尽量减少看起来出乎直觉的重复实例。

当前项目显式配置：

```yaml
dedupePeerDependents: true
```

这同样与官方默认值一致。

### 11.16 `strictPeerDependencies`、`autoInstallPeers`、`dedupePeerDependents` 的组合语义

这三个开关最容易被分开记，结果真正分析项目时还是糊。

可以把它们压缩成下面这组关系：

- `autoInstallPeers`
  - 处理“缺失但无冲突时，要不要自动补”
- `strictPeerDependencies`
  - 处理“peer 不满足时，是 warning 还是 hard fail”
- `dedupePeerDependents`
  - 处理“解析完成后，不冲突的 peer 结果要不要尽量收敛”

当前项目选择的是：

```yaml
strictPeerDependencies: false
autoInstallPeers: true
dedupePeerDependents: true
```

把这三条合起来看，当前仓库的偏好就很明确：

- 对常见缺失 peer，尽量自动补齐
- 对 peer 问题，不默认升级为硬失败
- 对可共享的 peer 结果，尽量收敛实例

这是一种明显偏工程可用性的组合，而不是“最大化暴露问题”的最严格组合。

### 11.17 `verifyDepsBeforeRun`：执行脚本前要不要先检查 `node_modules`

这个配置在 `pnpm 11` 下非常关键，因为它直接改变本地开发时“跑脚本前是否自动补安装”的行为。

官方 build settings 文档定义如下：

- 默认值：`install`
- 生效范围：`pnpm run` 和 `pnpm exec`

支持的取值包括：

- `install`
  - 如果 `node_modules` 不是最新状态，自动执行 install
- `warn`
  - 只打印警告
- `prompt`
  - 询问是否执行 install
- `error`
  - 直接报错
- `false`
  - 不做依赖状态检查

它处理的不是 peer，也不是 hoist，而是：

> 在执行脚本之前，当前 `node_modules` 是否已经和声明状态保持一致。

### 11.18 当前项目为什么显式把 `verifyDepsBeforeRun` 关掉

当前 `daisy-ui` 显式配置了：

```yaml
verifyDepsBeforeRun: false
```

这和 `pnpm 11` 的默认值 `install` 不同。

这条差异的工程含义非常直接：

- `pnpm run`
- `pnpm exec`

执行前，不会因为 `node_modules` 过期而自动触发 install，也不会先做这层校验。

这样做的好处通常是：

- 本地脚本启动路径更直接
- 避免命令前自动安装带来的额外等待和副作用

代价也同样明确：

- 如果本地依赖状态已经漂移
- 开发者更可能在运行期才发现问题

这类取舍在大仓里很常见，本质上是把“依赖一致性检查”从命令执行前移交给别的流程，例如：

- 显式安装步骤
- CI 校验
- 团队约定

### 11.19 当前项目的整体策略画像

把这一章涉及的真实配置合起来，当前 `daisy-ui` 的安装与兼容策略可以概括成：

```yaml
publicHoistPattern:
  - lefthook
  - eslint
  - oxfmt
  - oxlint
  - stylelint
  - '*postcss*'
  - '@commitlint/*'
  - czg

strictPeerDependencies: false
autoInstallPeers: true
dedupePeerDependents: true
verifyDepsBeforeRun: false
```

再结合未显式设置、因此走默认值的项：

- `hoist: true`
- `nodeLinker: isolated`

当前项目的整体画像就很清楚了：

- 安装总模型仍然是 `pnpm` 默认隔离布局
- 对生态兼容问题，使用小范围 public hoist 暴露工具链入口
- 对 peer 问题，尽量自动补、尽量收敛，但不过早硬失败
- 对脚本执行前依赖校验，选择关闭自动检查

这套配置的重点不是“更严格”，而是：

> 在保留 `pnpm` 默认模型主体的前提下，把兼容成本和开发摩擦压到一个可接受区间。

### 11.20 本章结论

布局兼容与解析策略这一组配置，真正难的不是记住名字，而是分清它们各自在改哪一层事实。

这一章可以收敛为七个结论：

1. `hoist` 默认开启，但它提升到的是隐藏层 `node_modules/.pnpm/node_modules`，不是直接平铺到根 `node_modules`
2. `publicHoistPattern` 只应作为小范围兼容工具使用，用来把特定工具链依赖公开暴露到根入口层
3. `shamefullyHoist` 等价于把所有依赖都公开提升到根目录，通常不应作为默认方案
4. `nodeLinker` 决定总安装模型；当前项目仍然站在默认的 `isolated` 模型上
5. `strictPeerDependencies`、`autoInstallPeers`、`dedupePeerDependents` 分别控制 peer 错误是否硬失败、缺失 peer 是否自动补装、解析结果是否尽量收敛
6. `verifyDepsBeforeRun` 控制 `pnpm run` / `pnpm exec` 前是否检查依赖状态；`pnpm 11` 默认是 `install`，而当前项目显式关闭了它
7. 当前 `daisy-ui` 的策略不是放弃 `pnpm` 默认模型，而是在默认模型上叠加有限兼容开关

## 12. 常用命令与日常操作

这一章不按命令名罗列，而按日常任务组织。

原因很简单：工程里真正出现的需求通常不是“想用哪个子命令”，而是：

- 第一次把仓库装起来
- 给某个包增加依赖
- 从某个包移除依赖
- 升级一批依赖
- 查一个包为什么会出现在仓库里
- 看当前到底装了什么
- 尽量收敛 lockfile
- 预热依赖缓存
- 审批安装脚本

因此，本章只回答三个问题：

1. 这类任务首选什么命令
2. 命令会改哪些文件
3. 在 workspace 里使用时要注意哪些边界

### 12.1 把整个仓库装起来：`pnpm install`

`pnpm install` 是整个工作区最基础的入口。

官方定义很直接：

- 用于安装项目全部依赖
- 在 workspace 中，默认会为所有项目安装依赖

这意味着在 Monorepo 根执行：

```bash
pnpm install
```

默认不是只装当前根包，而是：

- 读取 workspace 边界
- 读取 lockfile
- 为整个 workspace 解析并安装依赖

当前 `daisy-ui` 根 [`package.json`](file:///Users/apple/code/WebProjects/daisy-ui/package.json) 里也把这件事固定成了脚本：

```json
{
  "scripts": {
    "bootstrap": "pnpm install"
  }
}
```

因此，对这类仓库来说，“第一次装仓库”与“运行 bootstrap”本质上是同一件事。

### 12.2 `pnpm install` 会改什么

`pnpm install` 的影响面主要在三层：

1. 安装结果目录  
   - `node_modules`
   - `node_modules/.pnpm`
2. lockfile  
   - `pnpm-lock.yaml`
3. 与构建脚本审批相关的治理文件  
   - 例如 `pnpm-workspace.yaml` 中的 `allowBuilds`

但它不是每次都会同时改这三层。

最常见的情况是：

- lockfile 已经和声明一致
- 当前 `node_modules` 缺失或过期

这时它主要更新的是安装结果目录。

而在 CI 环境中，官方文档明确说明：

- 如果 lockfile 已存在
- 但当前安装会要求更新它
- 安装会直接失败

这正是第 6 章讲 `--frozen-lockfile` 语义时的延伸：  
日常本地开发可以让 install 推进解析结果，CI 则通常只接受已提交的解析结果。

### 12.3 常见的 install 变体

官方文档给出了四个最重要的变体：

```bash
pnpm i --offline
pnpm i --frozen-lockfile
pnpm i --lockfile-only
pnpm i --dry-run
```

它们分别对应：

- `--offline`
  - 只从本地 store 安装，不访问 registry
- `--frozen-lockfile`
  - 不允许更新 `pnpm-lock.yaml`
- `--lockfile-only`
  - 只更新 lockfile，不落安装结果
- `--dry-run`
  - 预览变更，不真正写入

这四个选项里：

- `--offline`
  - 会在第 14 章展开
- `--frozen-lockfile`
  - 第 6 章已经讲过核心语义

这里先记住它们是 install 的四个高频变体即可。

### 12.4 给某个包增加依赖：`pnpm add`

`pnpm add` 的作用是：

- 安装指定包
- 并把依赖声明写回当前项目清单

默认行为是写入 `dependencies`：

```bash
pnpm add sax
```

而下面几种写法分别对应不同声明位置：

```bash
pnpm add -D sax
pnpm add -O sax
pnpm add --save-peer react
```

对应关系是：

- `-D`
  - 写入 `devDependencies`
- `-O`
  - 写入 `optionalDependencies`
- `--save-peer`
  - 写入 `peerDependencies`，并作为 dev dependency 安装

因此，`pnpm add` 并不是只做“下载依赖”这一步，它同时也是“改清单声明”的入口。

### 12.5 在 workspace 根部执行 `pnpm add` 的边界

这是 Monorepo 中最容易误操作的点之一。

官方文档明确说明：

- 在 workspace 根包新增依赖时
- 默认会失败
- 需要显式加 `--ignore-workspace-root-check` 或 `-w`

例如：

```bash
pnpm add debug -w
```

这条限制背后的含义是：

> 根包不是默认的“随便塞所有依赖”的公共垃圾桶。

这和当前 `daisy-ui` 的项目结构是吻合的。它把大量业务依赖放在具体应用包和共享包里，而根包更多承担：

- workspace 级脚本入口
- 构建工具链依赖
- 全局治理配置的承载点

所以在这类仓库中新增依赖时，第一件事应该先判断：

- 这是某个具体包的依赖
- 还是整个 workspace 根工具链的依赖

### 12.6 `pnpm add` 在当前项目里的几个典型场景

以当前 `daisy-ui` 为例，至少有三类常见 add 场景：

**给根工具链增加依赖**

例如：

- `eslint`
- `lefthook`
- `typescript`
- `vite`

这类依赖本来就由根包统一承载，因此 add 的落点通常是根包。

**给具体应用包增加运行时依赖**

例如给某个 `apps/*` 包增加 UI 库或业务运行时依赖。  
这类依赖应该落到对应包的 `package.json`，而不是根包。

**给 workspace 包建立本地依赖关系**

这类场景里，更重要的往往不是 `add` 这个动作本身，而是写清：

- `workspace:*`
- `workspace:^`
- `workspace:~`

也就是说，真正关键的是本地依赖契约，而不是“命令能不能执行成功”。

### 12.7 `--save-catalog`：把新依赖直接写入 catalog

`pnpm add` 在 `v10.12.1` 起支持：

```bash
pnpm add <pkg> --save-catalog
pnpm add <pkg> --save-catalog-name <catalog_name>
```

它的意义不是“加一个依赖时顺手记个别名”，而是：

- 把新依赖的版本决策提升到 catalog
- 当前包的清单不直接写死版本号

这和第 10 章的 catalog 治理策略是联动的。

如果仓库已经明确采用 catalog 管理公共依赖，那么新增依赖时直接走 catalog 入口，会比“先写版本号，后面再人工抽到 catalog”更稳。

### 12.8 移除依赖：`pnpm remove`

`pnpm remove` 会同时做两件事：

- 从 `node_modules` 中移除对应包
- 从当前项目的 `package.json` 中移除对应声明

最常见的形式是：

```bash
pnpm remove react
```

而下面几种写法分别限定移除范围：

```bash
pnpm remove -D eslint
pnpm remove -O fsevents
pnpm remove -P axios
```

这意味着 remove 也不是只对安装结果生效，它同样是“声明层 + 安装层”的双写操作。

### 12.9 在 workspace 里批量移除：`pnpm remove -r`

官方文档说明：

- 在 workspace 中使用 `--recursive`
- 会从每个 workspace 包中移除指定依赖

例如：

```bash
pnpm remove -r lodash
```

这类命令威力很大，但也更需要确认边界。因为它不是“当前包删掉”，而是“整仓扫描所有 workspace 包后统一删除”。

因此，它更适合：

- 全仓统一淘汰某个依赖
- 做治理级收敛

不适合在未确认影响范围时直接尝试。

### 12.10 升级依赖：`pnpm update`

`pnpm update` 的职责不是“重新安装”，而是：

- 按指定规则把依赖提升到新的可解析版本
- 并把更新结果写回 lockfile
- 在需要时写回依赖声明位置

官方文档对它的 TL;DR 非常清楚：

```bash
pnpm up
pnpm up --latest
pnpm up foo@2
pnpm up "@babel/*"
```

它们分别对应：

- 按当前声明范围更新全部依赖
- 更新到最新稳定版，可能跨 major
- 把某个包更新到指定主版本线
- 用模式更新一批包

### 12.11 `pnpm update` 到底会写哪里

这一点官方文档讲得很关键：

- 会更新 `pnpm-lock.yaml`
- 会把新的范围写回依赖声明位置

但写回位置取决于声明方式。

**普通依赖声明**

如果依赖直接写在 `package.json` 中，更新时：

- 解析结果会进入 lockfile
- 声明范围会保留原来的运算符

例如：

- 原来是 `^1.1.0`
- 更新后仍然保持 caret range，只是版本基线抬高

**catalog 依赖声明**

如果依赖使用的是 `catalog:`，官方文档明确说明：

- 当前包的 `package.json` 不会被改写
- 被更新的是 `pnpm-workspace.yaml` 中对应的 catalog 条目

这条规则非常关键，因为它直接体现了 catalog 的治理价值：

> 版本升级的位置，不再分散在各个业务包清单里，而是回到统一版本入口表。

### 12.12 `--no-save`：只动 lockfile，不动声明范围

`pnpm update --no-save` 的作用是：

- 只更新 lockfile
- 不改 `package.json` 中声明的范围

这在两类场景里比较有价值：

- 想验证更高解析结果是否能工作
- 暂时不想改仓库的声明策略，只想推进一次实际解析

它本质上是在把“声明层”与“解析层”临时拆开。

### 12.13 查这个包为什么会在仓库里：`pnpm why`

`pnpm why` 用来回答的不是“装了什么”，而是：

> 谁依赖了这个包，它是沿哪条依赖链被带进来的。

官方定义是：

- 输出指定包的 reverse dependency tree

例如：

```bash
pnpm why vite
pnpm why -r vite
```

区别在于：

- 单包执行
  - 看当前项目里的反向依赖树
- `-r`
  - 在整个 workspace 范围内看反向依赖树

这对排查以下问题非常有用：

- 一个包为什么还没法删
- 某个版本为什么还存在
- 某个间接依赖是从哪条链路被带进来的

### 12.14 查当前装了什么：`pnpm list`

`pnpm list` 回答的是另一个问题：

> 当前安装结果长什么样。

它默认列出当前项目的直接依赖，`--depth` 控制展开深度：

```bash
pnpm ls --depth 0
pnpm ls --depth Infinity
pnpm ls -r --depth -1
```

几种典型含义如下：

- `--depth 0`
  - 看直接依赖
- `--depth Infinity`
  - 展开完整依赖树
- `-r --depth -1`
  - 在 workspace 内只列项目本身，不展开依赖树

官方文档还有一个很实用的选项：

```bash
pnpm ls --lockfile-only
```

它会从 lockfile 读取包信息，而不检查实际 `node_modules`。  
这在以下场景很有价值：

- 还没安装
- 安装目录不完整
- 只想快速看“理论上会装什么”

### 12.15 `why` 与 `list` 的分工

这两个命令很容易被混用，但它们解决的是不同问题：

- `pnpm why`
  - 看“为什么存在”
- `pnpm list`
  - 看“现在有什么”

如果问题是：

- “这个包是谁带进来的”
  - 优先 `why`
- “当前到底装成了什么样”
  - 优先 `list`

把两者分清，排错效率会高很多。

### 12.16 收敛 lockfile：`pnpm dedupe`

官方文档对 `pnpm dedupe` 的定义是：

> 执行一次 install，在可能的情况下移除 lockfile 中较旧的依赖版本。

这里最重要的是两个关键词：

- 它处理的是 lockfile 中的冗余解析结果
- 前提是“新版本在语义上可复用”

这说明 `dedupe` 不是强制“全仓只能剩一个版本”，而是：

- 在不违反版本范围和解析约束的前提下
- 尽量让 lockfile 更收敛

官方还提供了一个非常适合 CI 的选项：

```bash
pnpm dedupe --check
```

它不会真正安装或改 lockfile，只会检查：

- 如果执行 dedupe，是否会产生变更

一旦会变更，就以非零状态码退出。

### 12.17 预热依赖缓存：`pnpm fetch`

`pnpm fetch` 的官方定义是：

> 从 lockfile 把包抓取到 virtual store，忽略 package manifest。

这条定义很容易被低估。它说明 fetch 做的不是“完整安装”，而是：

- 只根据 lockfile 和配置
- 把所需包预先放进 virtual store

官方文档专门把它设计成 Docker / Monorepo 场景下的优化命令。因为它只依赖：

- `pnpm-lock.yaml`
- `pnpm-workspace.yaml`
- 以及必要的 patch / config 文件

因此，它非常适合把“下载依赖”与“真正落安装结果”拆成两个阶段。

当前章先记住它的职责边界。离线环境、Docker 分层、`install --offline` 的完整链路，会放到第 14 章系统展开。

### 12.18 管理全局 store：`pnpm store`

`pnpm store` 不是操作当前项目的依赖图，而是操作机器上的全局包存储。

官方文档列出的几个核心子命令是：

```bash
pnpm store path
pnpm store status
pnpm store prune
pnpm store add <pkg>
```

它们分别解决：

- `path`
  - 当前 store 在哪里
- `status`
  - 检查 store 中包内容是否被修改
- `prune`
  - 删除没有被任何项目引用的包
- `add`
  - 直接把包加入 store，而不改当前项目

其中最常用的是 `pnpm store prune`，但它的语义要和下一节的 `pnpm prune` 分开。

### 12.19 `pnpm store prune` 与 `pnpm prune` 不是一回事

这两个名字很像，但处理对象完全不同。

**`pnpm store prune`**

- 处理的是全局 store
- 删除的是不再被任何项目引用的包

**`pnpm prune`**

- 处理的是当前项目安装结果
- 删除的是当前安装里不需要的包，例如 dev 依赖或 optional 依赖

如果把两者混了，就会误判影响范围。

### 12.20 清理当前项目安装结果：`pnpm prune`

`pnpm prune` 的官方定义是：

- 删除不必要的包

最常见的两种写法是：

```bash
pnpm prune --prod
pnpm prune --no-optional
```

对应的含义是：

- `--prod`
  - 移除 `devDependencies`
- `--no-optional`
  - 移除 `optionalDependencies`

但官方文档同时给了一个非常重要的 warning：

> `pnpm prune` 当前不支持在 Monorepo 上递归执行。

这意味着在 workspace 场景里，不能把它当成“整仓生产化裁剪”的一把梭工具。  
官方建议更稳的做法是：

- 删除现有 `node_modules`
- 然后重新执行 `pnpm install --prod`

### 12.21 审批安装脚本：`pnpm approve-builds`

从 `pnpm 10.1.0` 开始，`pnpm approve-builds` 成为一个很重要的供应链治理入口。

官方定义是：

- 审批哪些依赖在安装过程中可以执行脚本

审批结果会写入：

- `pnpm-workspace.yaml`
  - `allowBuilds`

例如：

```bash
pnpm approve-builds esbuild fsevents !core-js
```

它的含义是：

- 显式允许 `esbuild`
- 显式允许 `fsevents`
- 显式拒绝 `core-js`

因此，这个命令不只是交互工具，而是把“安装脚本是否可信”落成可提交的仓库配置。

### 12.22 当前项目已经把一部分 build 审批结果写进了配置

当前 `daisy-ui` 的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 已经包含：

- `allowBuilds`

这说明当前项目并不是“安装时临时看提示再决定”，而是已经把一部分构建脚本许可固化到仓库治理配置中。

这也解释了为什么第 10 章把 `allowBuilds` 放在治理/安全语境，而不是普通安装选项语境里。

### 12.23 执行依赖自带命令：`pnpm exec`

`pnpm exec` 的职责是：

- 在当前项目作用域内执行 shell 命令
- 自动把 `node_modules/.bin` 加进 `PATH`

例如：

```bash
pnpm exec jest
pnpm exec changeset
```

这类命令的价值在于：

- 不需要全局安装工具
- 直接使用当前项目锁定版本的二进制

当前 `daisy-ui` 根脚本里就有多个典型例子：

```json
{
  "scripts": {
    "changeset": "pnpm exec changeset",
    "prepare": "is-ci || pnpm exec lefthook install"
  }
}
```

这说明当前项目把 `exec` 用在了：

- 工作流工具调用
- 本地开发准备流程

### 12.24 在 workspace 里批量执行命令：`pnpm -r exec`

官方文档明确说明：

```bash
pnpm -r exec jest
```

会在每个 workspace 项目中执行该命令。

这里最容易出错的是参数位置。官方给出的对比例子非常典型：

**正确**

```bash
pnpm -r exec jest
```

**错误**

```bash
pnpm exec jest -r
```

后者不是“递归执行 jest”，而是把 `-r` 传给了 `jest` 本身。

这条规则很小，但在大仓里非常实用。

### 12.25 临时运行一个远程工具：`pnpm dlx`

`pnpm dlx` 的语义是：

- 从 registry 临时抓取一个包
- 不把它安装成当前项目依赖
- 直接执行它暴露的默认二进制

例如：

```bash
pnpm dlx create-vue my-app
```

它适合的场景是：

- 一次性脚手架
- 临时 codemod
- 不希望污染当前项目依赖树的短期工具

当前 `daisy-ui` 根脚本里就有一个真实例子：

```json
{
  "scripts": {
    "catalog": "pnpm dlx codemod pnpm/catalog"
  }
}
```

这说明项目把 `dlx` 用在了“临时执行 catalog 相关 codemod”的场景，而不是把这类工具长期放进依赖树。

### 12.26 当前项目的命令使用画像

把当前 `daisy-ui` 根脚本收在一起看，可以得到一个很典型的 Monorepo 使用画像：

- `bootstrap`
  - `pnpm install`
- `dev:*`
  - `pnpm -F <pkg> run dev`
- `changeset`
  - `pnpm exec changeset`
- `prepare`
  - `pnpm exec lefthook install`
- `catalog`
  - `pnpm dlx codemod pnpm/catalog`
- `version`
  - `pnpm exec changeset version && pnpm install --no-frozen-lockfile`

这组脚本说明，当前项目并没有把 pnpm 只当作“装依赖工具”，而是把它当成：

- workspace 调度入口
- 本地二进制执行入口
- 版本治理入口
- 临时工具执行入口

### 12.27 本章结论

常用命令这一章的重点，不是记住更多子命令，而是建立稳定的任务映射关系。

这一章可以收敛为九个结论：

1. `pnpm install` 是 workspace 的基础入口，在 Monorepo 根执行时默认安装整个 workspace
2. `pnpm add` / `remove` 同时会改依赖声明和安装结果，不能把它们当成纯下载或纯删除动作
3. 在 workspace 根新增依赖需要显式确认根包语义，不能把根包当成所有依赖的默认落点
4. `pnpm update` 同时会推进 lockfile，并按声明方式写回 `package.json` 或 `pnpm-workspace.yaml` 的 catalog 条目
5. `pnpm why` 解决“为什么存在”，`pnpm list` 解决“现在有什么”
6. `pnpm dedupe` 用于在不破坏语义的前提下收敛 lockfile，`--check` 很适合放进 CI
7. `pnpm fetch` 负责预热 virtual store，不等于完整安装；它和离线安装、Docker 分层的组合会在第 14 章展开
8. `pnpm store prune` 清的是全局 store，`pnpm prune` 清的是当前项目安装结果，两者不能混淆
9. `pnpm exec` 与 `pnpm dlx` 分别对应“执行项目内已安装工具”和“临时执行远程工具”

## 13. 项目案例：`daisy-ui` / `vue-vben-admin` 的 `pnpm` 落地

这一章不再讲抽象定义，而把前面第 1 至 12 章的内容收回到当前仓库。

观察对象是当前 [`daisy-ui`](file:///Users/apple/code/WebProjects/daisy-ui) 仓库。  
从目录结构、包命名、脚本风格和 CI 配置可以看出，这个仓库沿用了 `vue-vben-admin` 的 Monorepo 组织方式，并在此基础上继续维护。

本章只回答一个问题：

> 前面讲过的 `pnpm` 机制，在这个真实仓库里到底是怎样组合起来的。

### 13.1 workspace 边界不是一句“这是个 Monorepo”就结束了

当前仓库的 workspace 边界定义在 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml)：

```yaml
packages:
  - internal/*
  - internal/lint-configs/*
  - packages/*
  - packages/@core/base/*
  - packages/@core/ui-kit/*
  - packages/@core/forward/*
  - packages/@core/*
  - packages/effects/*
  - packages/business/*
  - apps/*
  - scripts/*
  - docs
  - playground
```

这组规则说明，当前仓库不是简单的“两层 packages + apps”结构，而是一个多职责工作区：

- `apps/*`
  - 最终应用入口
- `packages/*`
  - 通用共享包
- `packages/@core/*`
  - 更细分的核心能力包
- `internal/*`
  - 内部工具和配置承载点
- `scripts/*`
  - 脚本也作为 workspace 包参与治理
- `docs` / `playground`
  - 文档与试验场同样被纳入统一工作区

这意味着该仓库对 `pnpm workspace` 的使用不是“方便本地联调”这么简单，而是把：

- 应用
- 共享库
- 构建配置
- 工具链
- 文档站点
- 辅助脚本

全部放进同一套依赖解析与版本治理体系。

### 13.2 目录分层对应的是职责分层

从当前目录树可以直接看到三类最重要的 workspace 成员：

```text
apps/
  web-antd
  web-antdv-next
  web-ele
  web-naive
  web-tdesign
  backend-mock

packages/
  stores
  types
  utils
  styles
  icons
  locales
  preferences
  constants
  @core/...

internal/
  tailwind-config
  tsconfig
  vite-config
  node-utils
  lint-configs/...
```

这三层的职责区分很清楚：

- `apps`
  - 产品入口和具体运行时组合层
- `packages`
  - 可复用业务与基础能力层
- `internal`
  - 工具链、配置和工程基建层

因此，当前仓库里的本地依赖关系不是“随便互相引用”，而是有明显层级的：

- 应用依赖共享包
- 共享包依赖核心包
- 工具配置包作为顶层工程基础设施被其他包复用

这正是 workspace 最适合处理的结构。

### 13.3 根包不是业务汇总包，而是工作区调度中心

当前根 [`package.json`](file:///Users/apple/code/WebProjects/daisy-ui/package.json) 的 `name` 是：

```json
{
  "name": "vben-admin-monorepo"
}
```

它不是一个准备发布给业务方消费的运行时包，而是整个工作区的控制平面。

根包当前承担的职责非常集中：

- 持有 workspace 级脚本入口
- 持有构建、检查、测试、发版相关依赖
- 固定 `pnpm` / Node 版本基线
- 作为 `catalog` 与 `workspace:*` 的主要汇合点

因此，根包里的依赖不能简单理解成“项目最核心的业务依赖”，更准确地说，它们是：

- 工作区级工具链依赖
- 跨包共用的构建与治理依赖

### 13.4 根脚本把 `pnpm` 当成工作区调度入口

当前根脚本有几类非常典型的模式。

**第一类：整仓入口**

```json
{
  "scripts": {
    "bootstrap": "pnpm install",
    "build": "cross-env NODE_OPTIONS=--max-old-space-size=8192 turbo build",
    "dev": "turbo-run dev",
    "lint": "vsh lint"
  }
}
```

这类脚本说明：

- `pnpm` 负责工作区安装与二进制分发
- `turbo` 负责任务图调度
- `vsh` 负责部分工程检查封装

也就是说，`pnpm` 在这里既是包管理器，也是整个任务系统的底层命令入口。

**第二类：按包过滤执行**

```json
{
  "scripts": {
    "dev:antd": "pnpm -F @vben/web-antd run dev",
    "dev:docs": "pnpm -F @vben/docs run dev",
    "build:antd": "pnpm run build --filter=@vben/web-antd"
  }
}
```

这里可以直接看到工作区过滤能力已经成为日常操作的一部分：

- 不是先 `cd` 到某个子包再执行
- 而是在根包统一调度，并用 `--filter` / `-F` 精确选中目标包

这正是第 12 章所说的“Monorepo 下命令不只是单包命令，而是工作区命令”的真实落地。

**第三类：基于项目内二进制的工具执行**

```json
{
  "scripts": {
    "changeset": "pnpm exec changeset",
    "prepare": "is-ci || pnpm exec lefthook install"
  }
}
```

这里体现的是另一条原则：

- 不依赖全局安装工具
- 直接执行当前仓库锁定版本的二进制

### 13.5 这个仓库的依赖声明模式是“双协议分工”

当前根 [`package.json`](file:///Users/apple/code/WebProjects/daisy-ui/package.json)、应用包 [`apps/web-antd/package.json`](file:///Users/apple/code/WebProjects/daisy-ui/apps/web-antd/package.json) 和共享包 [`packages/stores/package.json`](file:///Users/apple/code/WebProjects/daisy-ui/packages/stores/package.json) 放在一起看，可以非常清楚地看到一套稳定模式：

- 本地 workspace 包
  - 使用 `workspace:*`
- 外部 registry 包
  - 大量使用 `catalog:`

这不是语法偏好，而是治理策略。

### 13.6 `workspace:*` 在当前仓库里的作用不是“本地引用方便”

先看根包中的一组依赖：

```json
{
  "devDependencies": {
    "@vben/commitlint-config": "workspace:*",
    "@vben/eslint-config": "workspace:*",
    "@vben/turbo-run": "workspace:*",
    "@vben/vite-config": "workspace:*",
    "@vben/vsh": "workspace:*"
  }
}
```

再看应用包 [`apps/web-antd/package.json`](file:///Users/apple/code/WebProjects/daisy-ui/apps/web-antd/package.json)：

```json
{
  "dependencies": {
    "@vben/access": "workspace:*",
    "@vben/common-ui": "workspace:*",
    "@vben/locales": "workspace:*",
    "@vben/stores": "workspace:*",
    "@vben/utils": "workspace:*"
  }
}
```

再看共享包 [`packages/stores/package.json`](file:///Users/apple/code/WebProjects/daisy-ui/packages/stores/package.json)：

```json
{
  "dependencies": {
    "@vben-core/preferences": "workspace:*",
    "@vben-core/shared": "workspace:*",
    "@vben-core/typings": "workspace:*"
  }
}
```

这三组样本说明，`workspace:*` 在当前仓库里承担的是两层职责：

1. 明确声明“这就是工作区内部依赖，不允许解析成 registry 包”
2. 让应用层、共享包层、工程配置层都能在同一仓库内形成稳定本地依赖图

因此，这里的 `workspace:*` 不是补充写法，而是工作区内部依赖的主协议。

### 13.7 `catalog:` 在当前仓库里承担的是统一外部版本入口

当前根包中可以直接看到大量 `catalog:`：

```json
{
  "devDependencies": {
    "@types/node": "catalog:",
    "eslint": "catalog:",
    "lefthook": "catalog:",
    "tailwindcss": "catalog:",
    "turbo": "catalog:",
    "typescript": "catalog:",
    "vite": "catalog:",
    "vue": "catalog:"
  }
}
```

应用包和共享包同样沿用这一策略：

- `apps/web-antd`
  - `vue`、`pinia`、`dayjs`、`ant-design-vue` 都使用 `catalog:`
- `packages/stores`
  - `pinia`、`secure-ls`、`vue`、`vue-router` 使用 `catalog:`

这意味着当前仓库已经把外部依赖版本治理清晰地上移到了 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml)：

- 包内部主要表达“依赖谁”
- 版本入口集中由 `catalog` 决定

因此，这个仓库的依赖声明模型可以直接压缩成一句话：

> 内部依赖看 `workspace:*`，外部依赖看 `catalog:`。

### 13.8 `catalog` 与 `overrides` 在当前仓库里是联动关系，不是二选一

当前 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 同时包含：

```yaml
overrides:
  '@ast-grep/napi': 'catalog:'
  '@ctrl/tinycolor': 'catalog:'
  clsx: 'catalog:'
  pinia: 'catalog:'
  vue: 'catalog:'
```

和一份非常大的 `catalog` 版本表。

这说明当前仓库的治理模式不是：

- 只用 `catalog` 管入口

而是：

- 用 `catalog` 统一版本入口
- 再用 `overrides` 把部分关键依赖的最终解析也强制收敛回同一入口

这套组合尤其适合下面这类依赖：

- 运行时宿主
- 构建链路核心依赖
- 被大量 workspace 包共同消费的基础库

当前配置里的 `vue`、`pinia` 就是最典型的例子。

### 13.9 `publicHoistPattern` 在当前仓库里只服务工具链兼容

当前 `publicHoistPattern` 配置如下：

```yaml
publicHoistPattern:
  - lefthook
  - eslint
  - oxfmt
  - oxlint
  - stylelint
  - '*postcss*'
  - '@commitlint/*'
  - czg
```

这个列表有一个很稳定的特征：

- 几乎全部是工具链相关包
- 不是业务运行时共享库

这说明当前仓库对 `publicHoistPattern` 的使用非常克制：

- 不是为了把整个依赖树重新平铺到根目录
- 而是为了让 lint / commit / hook / postcss 相关工具更容易在根脚本和外部工具环境里被解析到

这与第 11 章的判断完全一致：  
当前项目仍然站在 `nodeLinker=isolated` 的默认模型上，只对个别工具链包做公开提升。

### 13.10 `allowBuilds` 说明这个仓库已经把安装脚本治理固化了

当前 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 里可以直接看到：

```yaml
allowBuilds:
  '@parcel/watcher': true
  core-js: true
  esbuild: true
  lefthook: true
  vue-demi: true
```

这说明当前仓库并没有把安装脚本执行权限交给“每次安装时临场决定”，而是已经把一部分许可结果提交进仓库。

从工程治理角度看，这意味着：

- 某些带构建脚本的依赖被明确允许
- 安装行为不再完全依赖开发者本地交互
- CI、本地开发、多人协作对安装脚本的判断更容易保持一致

### 13.11 根包依赖结构证明它更像“工程中枢”而不是应用包

根包的依赖结构也能印证这一点。

根 `dependencies` 只有：

```json
{
  "dependencies": {
    "sortablejs": "catalog:"
  }
}
```

而大量真正重要的依赖都放在 `devDependencies`：

- `turbo`
- `typescript`
- `vite`
- `vitest`
- `eslint`
- `lefthook`
- `playwright`
- `@vben/*` 系列工程配置包

这说明根包的职责不是承载业务运行时，而是：

- 组织开发流程
- 组织构建流程
- 组织测试、检查和发布流程

这也是 Monorepo 中非常典型的一种根包形态。

### 13.12 内部工程配置也被做成了 workspace 包

当前仓库的 `internal/lint-configs/*`、`internal/tailwind-config`、`internal/tsconfig`、`internal/vite-config` 并不是散落的配置文件目录，而是被纳入 workspace 的包。

以 [`internal/lint-configs/commitlint-config/package.json`](file:///Users/apple/code/WebProjects/daisy-ui/internal/lint-configs/commitlint-config/package.json) 为例：

```json
{
  "name": "@vben/commitlint-config",
  "dependencies": {
    "@commitlint/cli": "catalog:",
    "@commitlint/config-conventional": "catalog:",
    "@vben/node-utils": "workspace:*",
    "commitlint-plugin-function-rules": "catalog:",
    "cz-git": "catalog:",
    "czg": "catalog:"
  }
}
```

这说明当前仓库不仅把业务代码包化，也把工程配置本身包化。这样带来的直接结果是：

- 配置可以被其他 workspace 包以标准依赖方式复用
- 配置自身也进入统一版本治理
- 配置依赖关系同样受 lockfile、catalog、workspace 规则约束

这比“根目录复制多份配置文件”稳定得多。

### 13.13 CI 的安装路径与本地保持同一套 `pnpm` 语义

当前 CI 主入口在 [`.github/workflows/ci.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/workflows/ci.yml) 和 [`.github/actions/setup-node/action.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/actions/setup-node/action.yml)。

其中 `setup-node` 这个复合动作做了几件事：

1. 安装 `pnpm`
2. 按 `.node-version` 安装 Node.js
3. 开启 `pnpm` 缓存
4. 读取 `pnpm store path`
5. 对 store 做 GitHub Actions cache
6. 最后执行：

```bash
pnpm install --frozen-lockfile
```

这条链路非常典型，也非常重要。它说明当前仓库在 CI 里坚持的是：

- `pnpm` 作为唯一包管理器
- 复用 store 缓存，而不是缓存整个 `node_modules`
- 用 `--frozen-lockfile` 强制 lockfile 一致性

换句话说，这个仓库并没有为 CI 单独设计另一套依赖安装语义，而是把本地开发和流水线都统一在同一套 `pnpm` 事实模型上，只是 CI 更严格。

### 13.14 构建、检查、测试都通过同一套安装基线展开

当前 [`ci.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/workflows/ci.yml) 中的几个关键 job 是：

- `test`
  - `pnpm run test:unit`
- `lint`
  - `pnpm run lint`
- `check`
  - `pnpm check:type`

另一个 [`build.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/workflows/build.yml) 中则直接执行：

```bash
pnpm run build
```

这说明当前仓库的 CI 不是“每个步骤各自偷偷装一套依赖”，而是：

- 先统一安装
- 再在同一套依赖基线上执行测试、检查、构建

因此，`pnpm-lock.yaml` 对这个仓库的重要性不是抽象概念，而是所有流水线任务的前置基础。

### 13.15 当前仓库的 `pnpm` 落地模式可以压缩成一张图

把前面所有事实收拢起来，当前仓库的落地模式可以压缩成下面这组规则：

1. `pnpm-workspace.yaml`
   - 定义工作区边界
2. 根 `package.json`
   - 作为工作区调度中心
3. `workspace:*`
   - 承载内部包依赖契约
4. `catalog:`
   - 承载外部依赖统一版本入口
5. `overrides`
   - 对关键基础依赖做最终解析收敛
6. `publicHoistPattern`
   - 只对工具链做有限公开提升
7. `allowBuilds`
   - 固化安装脚本执行许可
8. CI
   - 复用 store 缓存，并通过 `pnpm install --frozen-lockfile` 保持安装一致性

这组规则彼此不是孤立的，它们组合起来形成的其实是一套完整工程策略：

- 目录边界清晰
- 本地依赖契约明确
- 外部版本入口集中
- 关键依赖最终收敛
- 工具链兼容性受控
- CI 与本地安装语义统一

### 13.16 本章结论

当前 `daisy-ui` 仓库里的 `pnpm` 使用方式，不是零散配置项的堆积，而是一套已经成形的 Monorepo 工程组织方案。

这一章可以收敛为七个结论：

1. 该仓库把应用、共享库、工程配置、脚本、文档和 playground 全部纳入同一 workspace 管理
2. 根包的角色不是业务运行时包，而是工作区调度中心和工具链承载点
3. 内部依赖统一使用 `workspace:*`，外部依赖大量使用 `catalog:`，二者分工清晰
4. `catalog` 与 `overrides` 组合使用，说明该仓库既治理版本入口，也治理最终解析结果
5. `publicHoistPattern` 只服务工具链兼容，而不是回退到全面平铺的 `node_modules`
6. `allowBuilds` 说明安装脚本治理已经被提交进仓库配置
7. CI 通过缓存 store 并执行 `pnpm install --frozen-lockfile`，把本地与流水线统一到同一套 `pnpm` 安装语义

## 14. 私服、离线环境、缓存与 CI/CD

本章讨论离线开发环境中的迁移与落地问题。

场景设定如下：

- 公网机器可以访问 npm registry
- 内网开发机不能直接访问公网
- 需要把一个已经存在的 `pnpm` Monorepo 导入内网
- 目标不是简单解压源码，而是在内网能稳定执行 `pnpm install`、`pnpm dev`、`pnpm build`

`vue-vben-admin` 与当前 `daisy-ui` 这类仓库，最容易在这一类场景中暴露安装链路问题。

### 14.1 问题本质与场景

“离线环境里 `pnpm install` 跑不起来”通常不是某一个命令参数没写对，而是安装链路上至少有一个环节并不存在。

一条完整的安装链路至少包含：

1. registry 路由  
   包请求要发到哪里
2. 认证信息  
   如果是私有源，是否有权限读取
3. lockfile  
   当前项目到底要安装哪些包、哪些版本、哪些 peer 变体
4. store  
   安装时需要的 tarball 和元数据是否已在本地
5. 安装脚本与原生构建环境  
   安装后是否还能正确完成 `postinstall`、二进制下载或本地编译

离线失败，本质上就是这条链的某一段断了。

**三种离线场景**

离线环境并不是一个单一问题。至少应区分三类：

**单机临时离线**

- 一台机器暂时无法访问外网
- 只要求当前项目能装起来

**内网长期开发**

- 团队成员的开发机、CI、构建机长期处于内网
- 未来还会持续导入新项目和新版本

**离线镜像构建**

- 最终目标不是本机开发，而是在 Docker / CI 中稳定构建

这三类场景对应的最佳方案并不相同。如果不先分场景，就会把“长期方案”和“应急方案”混用。

### 14.2 推荐方案与基本原则

**长期内网开发的首选方案：内网私有 npm 代理仓**

对于长期内网开发，最推荐的方案不是拷贝 `node_modules`，也不是每次从公网手工导入一批 tarball，而是部署一个内网 npm 代理仓或制品仓。

常见实现有三类：

- Verdaccio
  - 适合 JavaScript/TypeScript 为主、希望部署轻量的团队
- Nexus Repository
  - 适合需要统一管理多种制品格式的团队
- JFrog Artifactory
  - 适合已经在用 JFrog 体系、需要 npm 远程仓、本地仓、虚拟仓和审计能力的团队

这类方案的共同点不是“名字不同”，而是它们都把依赖获取改造成了：

- 外部源由内网仓库统一代理或缓存
- 开发机和 CI 只访问内网地址
- 私有包与第三方包走同一套仓库入口

这类方案之所以是首选，是因为它解决的是**长期入口统一**问题，而不是一次性安装问题。

**为什么内网代理仓是首选**

长期内网环境中，最难维护的不是某一次导入，而是后续持续演进。

如果没有内网代理仓，后面几件事都会反复出现：

- 新分支切换后又缺一批包
- 新人机器需要重新导入依赖
- CI、开发机、Docker 使用不同的临时缓存
- 某次升级只导入了根依赖，没有导入新的传递依赖
- 某些包版本被清理后再也无法重建

内网代理仓把这些问题统一收口到一个稳定入口：

- 所有安装请求只打到内网源
- 公网依赖由代理仓按需拉取并缓存
- 私有包直接发布到内网源
- 同一个 lockfile 在开发机和 CI 中指向同一套包来源

这比“每次搬一次依赖”稳定得多。

**代理仓并不等于自动拥有全量离线包**

这一点非常容易误解。

无论是 Verdaccio 还是 Artifactory 的 remote repository，本质上都是**proxy cache**，不是一上来就拥有全量镜像。

它们的共同特点是：

- 第一次被请求时才会去上游拉取
- 拉到之后缓存
- 后续相同请求优先从缓存返回

因此，刚搭好的代理仓并不天然适合“完全断网的第一天”。  
如果目标是彻底断开公网后仍能安装，必须先经历一次**预热或导入**过程，把目标项目需要的包先填进代理仓缓存或本地仓。

**只做一次项目导入时的应急方案：预热 store 后离线安装**

如果当前目标不是建设长期内网仓库，而是把一个项目从公网导入内网并尽快恢复可用环境，那么更现实的方案是：

1. 在公网机器上按 lockfile 预热 store
2. 把源码、lockfile、workspace 配置和 store 一起导入内网
3. 在内网机器上执行离线安装

这套方案的核心不是复制安装结果，而是复制**安装所需输入**。

对应的对象至少包括：

- 仓库源码
- `pnpm-lock.yaml`
- `pnpm-workspace.yaml`
- 根 `package.json`
- 所有 workspace 子包源码与各自 `package.json`
- 预热后的 pnpm store

如果只拷源码，不拷 store，离线安装不会成功。  
如果只拷 `node_modules`，后续重建、切分支、重新安装都会很脆弱。

**`--offline` 的真实含义**

`pnpm install --offline` 的语义不是“进入离线模式后自动补齐依赖”，而是：

- 安装阶段不允许访问远程 registry
- 所需包必须已经存在于本地 store

因此，这个命令适合放在：

- 依赖已被完整预热之后
- 或内网代理仓已经具备所需包之后

如果把它当成“离线环境专用万能修复开关”，结果只会是更早失败。

**`pnpm fetch` 是离线导入链路的关键步骤**

官方把 `pnpm fetch` 设计成专门用于：

- 根据 lockfile 把依赖先抓取到 store
- 把“下载”与“安装”拆成两个阶段

它的关键价值在离线迁移里非常直接：

- 公网机器负责把 lockfile 需要的第三方依赖全部下载到 store
- 内网机器只负责根据本地 store 还原安装结果

这比直接执行一次普通 `pnpm install` 更适合作为导入步骤，因为 `fetch` 的输入更稳定：

- 它围绕 lockfile 工作
- 不依赖当前机器已经存在的 `node_modules`
- 更适合被打包、传输和复用

### 14.3 单项目导入内网的实操命令

针对 `pnpm` Monorepo，一个更稳的导入顺序如下：

**第一步：公网机器固定版本基线**

至少确认以下版本与目标环境一致或尽量接近：

- Node.js
- `pnpm`
- 操作系统
- CPU 架构

当前 `daisy-ui` 的基线已经写在根 [`package.json`](file:///Users/apple/code/WebProjects/daisy-ui/package.json) 中：

- `packageManager: pnpm@11.16.0`
- `engines.node: ^22.18.0 || ^24.12.0`

这意味着如果内网环境不是这条基线，迁移时就不能假设结果完全一致。

**第二步：在公网机器获取完整源码**

必须是完整 workspace，而不是只拿根目录的少量文件。  
至少需要包含：

- `apps/*`
- `packages/*`
- `internal/*`
- `scripts/*`
- `docs`
- `playground`

以及定义工作区边界的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml)。

**第三步：在公网机器预热 store**

常见方式有两种：

```bash
pnpm fetch
```

或：

```bash
pnpm install --frozen-lockfile
```

用于离线导入时，更推荐先以 `fetch` 为主，因为它更清楚地把“下载依赖”和“生成安装结果”分开。

**第四步：导出 store 与源码**

需要一起导出的不是单独的 `node_modules`，而是：

- 仓库源码
- `pnpm-lock.yaml`
- store 目录

**第五步：内网机器恢复 store 后再执行离线安装**

恢复完成后，再执行：

```bash
pnpm install --offline --frozen-lockfile
```

这里 `--offline` 保证不再访问外网，`--frozen-lockfile` 保证不会因为环境差异重新改写解析结果。

下面给出一套可直接执行的 **Unix-like shell** 示例命令。  
示例目标是把当前这类 `pnpm` Monorepo 从公网机器导入内网机器。Windows 环境需要替换成等价的 PowerShell 或批处理命令，但流程对象不变。

#### 公网机器示例命令

**1. 固定并记录版本基线**

```bash
node -v
pnpm -v
uname -s
uname -m
```

如果版本与目标内网环境差异过大，不应继续往下做导入包，而应先把版本基线对齐。

**2. 获取完整仓库源码并进入项目**

```bash
git clone <your-repo-url> daisy-ui
cd daisy-ui
```

如果不是通过 `git clone` 获取，而是直接拷工作目录，也必须保证整个 workspace 都在，不能只带根目录少量文件。

**3. 用 lockfile 预热 store**

```bash
pnpm fetch
```

如果需要先验证公网环境本身能正确安装，再执行一次冻结安装：

```bash
pnpm install --frozen-lockfile
```

这一条不是离线导入的必需步骤，但对排除“项目本身当前就装不起来”很有价值。

**4. 记录当前 store 路径**

```bash
pnpm store path
```

当前机器上查到的实际路径是：

```text
/Users/apple/Library/pnpm/store/v11
```

不同系统默认值会不同，因此导出前应始终以 `pnpm store path` 的真实输出为准，不要手猜目录。

**5. 打包源码与 store**

```bash
STORE_DIR="$(pnpm store path)"
EXPORT_DIR="$PWD/.offline-export"

rm -rf "$EXPORT_DIR"
mkdir -p "$EXPORT_DIR/project" "$EXPORT_DIR/store"

tar -czf "$EXPORT_DIR/project-src.tgz" \
  --exclude .git \
  --exclude node_modules \
  .

tar -C "$STORE_DIR" -czf "$EXPORT_DIR/pnpm-store.tgz" .

cat > "$EXPORT_DIR/ENVIRONMENT.txt" <<EOF
node=$(node -v)
pnpm=$(pnpm -v)
os=$(uname -s)
arch=$(uname -m)
store_dir=$STORE_DIR
EOF
```

导出目录中至少应包含：

- `project-src.tgz`
- `pnpm-store.tgz`
- `ENVIRONMENT.txt`

然后再通过 U 盘、内网摆渡机或其他合规介质传入内网。

#### 内网机器示例命令

**1. 先检查内网机器版本**

```bash
node -v
pnpm -v
uname -s
uname -m
```

至少应保证：

- `pnpm` 主版本与外网准备时一致
- Node 版本满足项目根 `package.json` 的 `engines.node`
- OS / CPU 架构尽量一致

**2. 恢复源码**

```bash
mkdir -p /data/workspace/daisy-ui
tar -xzf project-src.tgz -C /data/workspace/daisy-ui
cd /data/workspace/daisy-ui
```

**3. 恢复 store 并固定 store-dir**

```bash
mkdir -p /data/pnpm-store
tar -xzf pnpm-store.tgz -C /data/pnpm-store
pnpm config set store-dir /data/pnpm-store --global
pnpm store path
```

这里的目标是让 `pnpm store path` 指向刚恢复出来的目录，而不是继续落回系统默认路径。

**4. 处理 registry 配置**

如果当前内网完全不走私有 registry，只依赖恢复出来的 store，那么至少要确保安装时使用 `--offline`。  
如果当前内网已经有私有 npm 源，则应把项目根 `.npmrc` 改成内网地址，例如：

```bash
cat > .npmrc <<'EOF'
registry=http://your-internal-registry/
EOF
```

如果仍然保留：

```ini
registry=https://registry.npmmirror.com
```

那么任何未命中本地 store 的请求都会继续打向外网镜像。

**5. 执行离线安装**

```bash
pnpm install --offline --frozen-lockfile
```

对 Monorepo 来说，这一步会按 workspace 根统一安装全部项目依赖。  
如果当前命令失败，不应立刻删除 lockfile 重新安装，而应先检查：

- store 是否完整恢复
- `.npmrc` 是否仍指向外网
- `pnpm` 主版本是否一致
- 平台相关安装脚本是否失败

**6. 最小验证**

```bash
pnpm store path
pnpm -r exec node -v
pnpm run build --filter=@vben/web-antd
```

如果目标是本地开发而不是构建，也可以替换成：

```bash
pnpm run dev --filter=@vben/web-antd
```

#### 失败时先执行的排查命令

离线导入失败时，优先执行下面这组命令，而不是先删 `node_modules`：

```bash
node -v
pnpm -v
pnpm store path
cat .npmrc
test -f pnpm-lock.yaml && echo "lockfile exists"
test -f pnpm-workspace.yaml && echo "workspace file exists"
```

如果怀疑某个 workspace 包没有一起带入，再检查：

```bash
find apps packages internal scripts -name package.json | sed -n '1,40p'
```

如果怀疑 store 没有恢复到当前 `pnpm` 正在使用的位置，再检查：

```bash
pnpm config get store-dir
pnpm store path
```

当 `pnpm config get store-dir` 没有返回值时，不代表没有 store，只代表没有显式覆盖；此时应以 `pnpm store path` 的实际输出为准。

### 14.4 实操中的关键风险点

**为什么不推荐直接复制 `node_modules`**

直接复制 `node_modules` 看起来最省事，但对 `pnpm` 来说通常不是最稳做法，原因至少有四个：

1. `pnpm` 的安装结果依赖 symlink、virtual store 和真实路径关系  
   复制后的路径变化可能让部分引用关系失真
2. `node_modules` 只能代表某一次安装结果  
   不能稳定支持后续重新安装、切分支、升级依赖
3. 原生模块与安装脚本副产物强依赖当前平台  
   外网机器和内网机器平台不一致时更容易失效
4. 无法形成可重复的导入流程  
   下一次升级时仍要重新摸索

对 `pnpm` 项目来说，store 是更接近“依赖输入”的对象，`node_modules` 更接近“当前机器的一次安装输出”。

**Monorepo 离线导入时最容易漏掉的不是根依赖，而是 workspace 本身**

`vue-vben-admin` 和当前 `daisy-ui` 这类仓库并不是单包项目，很多关键依赖是工作区内部依赖。

当前仓库在第 13 章已经看到两条主线：

- 内部依赖大量使用 `workspace:*`
- 外部依赖大量使用 `catalog:`

这对离线导入意味着两件事：

**第一，内部包不是从 registry 下载的**

例如：

- `@vben/stores`
- `@vben/utils`
- `@vben-core/preferences`
- `@vben/vite-config`

它们的来源是当前仓库源码，不是外部 registry。  
因此，离线失败有时不是“缺第三方包”，而是“workspace 子包根本没一起带进来”。

**第二，外部版本入口集中在 workspace 配置里**

当前仓库的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 里包含：

- `catalog`
- `overrides`
- `allowBuilds`
- `minimumReleaseAge`

如果只拿部分源码、或拿错这份配置文件，内网重建出来的依赖图就可能已经不是同一棵树。

**当前仓库的 `.npmrc` 在离线环境里是一个直接风险点**

当前仓库的 [`.npmrc`](file:///Users/apple/code/WebProjects/daisy-ui/.npmrc) 只有一行：

```ini
registry=https://registry.npmmirror.com
```

在公网或可访问镜像站的网络里，这不是问题。  
但在完全离线的内网里，这一行配置意味着：

- 默认 registry 仍然指向外部地址
- 任何未命中本地 store 的请求都会直接失败

因此，项目导入内网后，必须明确决定以下两种模式之一：

**模式 A：完全依赖本地 store**

- 保留现有 `.npmrc`
- 但安装时必须保证 `--offline`
- 且本地 store 已完整预热

**模式 B：改指向内网 registry**

- 把 `.npmrc` 中的 `registry` 改成内网源
- 让开发机与 CI 统一走内网入口

长期看，模式 B 更稳。

**内网私有源的配置边界**

私有源场景里，需要始终把下面三类配置分开：

**项目级配置**

- `registry`
- `@scope:registry`

这类配置决定包请求打到哪个源，可以进入项目级 `.npmrc`。

**运行环境凭证**

- `_authToken`
- `username`
- `_password`
- `always-auth`

这类配置更适合通过：

- 用户级 `~/.npmrc`
- 专门的 auth 文件
- CI secret 注入

处理，而不是直接提交到仓库。

**工作区治理配置**

- `catalog`
- `overrides`
- `allowBuilds`
- `minimumReleaseAge`

这些属于 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml)，与私服认证是不同层问题。

**原生模块和安装脚本是离线迁移里最容易晚爆的问题**

很多离线迁移在“下载依赖”阶段看起来已经成功，但真正失败发生在安装脚本阶段。

当前仓库的 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 已显式允许这些包执行构建脚本：

- `@parcel/watcher`
- `core-js`
- `esbuild`
- `lefthook`
- `vue-demi`

这至少说明两件事：

1. 当前仓库依赖树中确实存在需要安装脚本的包
2. 离线迁移不能只关心 tarball 是否下载到了本地

其中最典型的是这类依赖：

- 会在安装阶段下载或选择平台相关二进制的包
- 会在本地编译 native addon 的包

如果外网机器和内网机器的平台不同，可能出现：

- 外网预热成功
- 内网安装阶段仍失败

因此，离线导入时应优先保持以下条件一致：

- OS
- CPU 架构
- Node 版本
- `pnpm` 版本

**同版本基线不是形式要求，而是安装成功率的前提**

当前仓库除了 `packageManager: pnpm@11.16.0` 外，还要求：

- `engines.node: ^22.18.0 || ^24.12.0`

CI 里则通过 [`.github/actions/setup-node/action.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/actions/setup-node/action.yml) 和 [`.node-version`](file:///Users/apple/code/WebProjects/daisy-ui/.node-version) 把 Node 基线固定在同一条路径上。

这对离线迁移的含义非常直接：

- 外网准备依赖时用的版本基线
- 内网实际安装与运行的版本基线

不应相差过大，更不应跨出仓库声明范围。

否则即使 store 完整，也可能在：

- `engines` 校验
- 安装脚本
- 原生模块 ABI
- 最终运行时行为

上出现偏差。

**当前仓库已经具备两类可复用经验**

从当前仓库可以直接提炼出两类对离线环境有价值的经验。

**第一类：安装一致性经验**

CI 的 [`.github/actions/setup-node/action.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/actions/setup-node/action.yml) 采用的是：

1. 安装 `pnpm`
2. 安装 Node
3. 通过 `pnpm store path --silent` 获取 store 路径
4. 对 store 做缓存
5. 执行 `pnpm install --frozen-lockfile`

这说明当前仓库已经把“缓存 store 而不是缓存整个 `node_modules`”和“用冻结 lockfile 安装”作为标准策略。

**第二类：Docker 缓存经验**

当前 [`scripts/deploy/Dockerfile`](file:///Users/apple/code/WebProjects/daisy-ui/scripts/deploy/Dockerfile) 采用：

```dockerfile
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile
```

这说明当前仓库已经在容器环境中显式缓存 store。  
虽然它还没有完全切成 `fetch + offline install` 的两阶段模式，但“store 是缓存主体”这一点已经是对的。

**Docker 与完全离线内网不是同一个问题**

这一点需要单独强调。

Docker 构建里的 cache mount 主要解决的是：

- 重复下载慢
- 构建层缓存命中率差

而完全离线内网要解决的是：

- 根本没有外部 registry 可以访问
- 开发机和 CI 不能依赖临时网络连通性

因此，下面两种情况不能混为一谈：

- Docker 中缓存 store
- 内网中已经拥有完整可用的依赖来源

前者可以提升速度；后者才决定能不能真正离线。

**`minimumReleaseAge` 与离线导入不是同一问题**

当前仓库在 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 中设置了：

```yaml
minimumReleaseAge: 0
```

这表示当前仓库没有启用“新发布版本冷静期”。

这个字段与离线导入的关系不大。它更多控制的是：

- 当 registry 可访问时
- 是否允许立即安装刚发布的新版本

离线迁移失败的主要原因通常不在这里，而在：

- registry 路由错误
- store 未预热
- 版本基线不一致
- 安装脚本或 native 构建失败

因此，在诊断离线失败时，不应把精力优先放在这类供应链策略位上。

### 14.5 最终推荐路径与检查清单

把前面的讨论压缩成决策，可以得到两条明确路径。

**路径 A：长期团队内网开发**

首选：

- 部署内网 npm 代理仓或制品仓
- 项目 `.npmrc` 指向内网 registry
- 外网与内网之间建立受控的预热、同步或导入流程

这条路径最适合：

- 团队多人协作
- 多项目长期维护
- 内网 CI、构建机、开发机都要稳定运行

**路径 B：单项目应急导入**

次优但可行：

1. 外网机器拉完整源码
2. 固定 Node / `pnpm` 基线
3. 用 `pnpm fetch` 预热 store
4. 一并导出源码、lockfile、workspace 配置和 store
5. 内网机器执行 `pnpm install --offline --frozen-lockfile`

这条路径适合：

- 临时导入一个项目
- 先求能跑起来
- 暂时没有内网制品仓条件

**当前场景下不推荐的路径**

以下做法成功率通常较低，或后续维护成本很高：

- 只复制源码，不复制 store
- 只复制根包，不复制完整 workspace
- 直接复制 `node_modules` 作为长期方案
- 外网与内网使用不同的 Node / `pnpm` 主版本
- 项目 `.npmrc` 仍指向公网镜像，却又不使用 `--offline`
- 未评估安装脚本与原生模块，就假设离线导入后一定可运行

这些做法即使某次碰巧成功，也很难形成可重复的工程流程。

**离线导入 `vue-vben-admin` / `daisy-ui` 时的检查清单**

这类仓库导入内网前，至少应逐项检查：

1. `pnpm-workspace.yaml` 是否完整带入
2. 所有 workspace 子包源码是否完整带入
3. `pnpm-lock.yaml` 是否与源码版本匹配
4. Node 与 `pnpm` 版本是否符合根 `package.json` 要求
5. store 是否已经在公网机器上按 lockfile 预热
6. `.npmrc` 是否仍指向外部镜像
7. `allowBuilds` 中涉及的构建脚本包是否会在目标平台重新执行
8. 内网机器是否具备必要的系统构建环境

只要其中任一项缺失，离线失败就很常见。

### 14.6 本章结论

离线环境中的 `pnpm` 问题，不能靠单个命令理解，必须放回完整安装供应链中理解。

这一章可以收敛为十个结论：

1. 长期内网开发的首选方案是内网 npm 代理仓或制品仓，而不是反复手工搬运依赖
2. 代理仓是按需缓存的 proxy，不是天然自带全量包镜像，断公网前需要先完成预热或导入
3. 单项目临时导入内网时，更稳的对象是源码、lockfile、workspace 配置和预热后的 store，而不是 `node_modules`
4. `pnpm install --offline` 的前提是本地 store 已完整具备所需依赖
5. `pnpm fetch` 的价值在于把“下载依赖”和“生成安装结果”拆开，是离线导入链路的关键步骤
6. `vue-vben-admin` / `daisy-ui` 这类 Monorepo 的离线难点，不只在第三方包，还在完整 workspace 是否一并带入
7. 当前仓库的 `.npmrc` 指向外部镜像，进入内网后必须明确改为“完全依赖本地 store”或“改指向内网 registry”中的一种
8. 原生模块和安装脚本是离线迁移中最容易晚爆的问题，平台、Node、`pnpm` 版本越一致，成功率越高
9. 当前仓库已经证明“缓存 store + `pnpm install --frozen-lockfile`”是可行基线，但这不等于已经具备完全离线能力
10. 判断离线方案是否可靠的标准，不是某次安装碰巧成功，而是它能否在开发机、CI、Docker 和后续版本升级中重复成立

## 15. 依赖治理与长期维护

这一章不再讲“怎么安装”，而讲“怎么长期维护”。

如果说前面几章解决的是：

- 怎么把依赖装对
- 怎么把工作区组织好
- 怎么把流水线跑起来

那么这一章处理的是另一个更慢、也更关键的问题：

> 当一个 Monorepo 长期演进时，如何避免依赖版本漂移、lockfile 失控、供应链边界失守，以及团队成员各自形成不同的安装习惯。

### 15.1 长期维护的目标不是“永远最新”，而是“持续可控”

依赖治理最常见的误区，是把“升级勤快”误当成“治理好”。

真实工程里，更重要的目标通常是：

- 升级路径可追踪
- 版本入口集中
- 解析结果可复现
- 风险动作有审查点
- 团队成员遵循同一套工作流

也就是说，长期维护关注的不是某次 install 成不成功，而是：

- 三个月后依赖树是否仍然可解释
- 半年后 lockfile 是否还能被正常评审
- 版本升级是否有稳定入口
- 安装脚本权限是否处于受控状态

### 15.2 当前仓库已经把“版本入口治理”放到明确位置

当前仓库的长期治理基础，不是从 `pnpm update` 命令开始，而是从两个对象开始：

- [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 中的 `catalog` / `overrides`
- 根 [`package.json`](file:///Users/apple/code/WebProjects/daisy-ui/package.json) 中把大量外部依赖写成 `catalog:`

这意味着版本治理的第一原则已经固定：

- 不在每个子包里各自维护一份外部版本范围
- 尽可能把外部版本入口收敛到 workspace 级配置

从长期维护角度看，这一点比“某次升级用了什么命令”更重要。因为它直接决定：

- 升级动作是否集中
- 评审是否有稳定落点
- merge conflict 是否容易控制

### 15.3 当前仓库已经预留了专门的升级入口

根 [`package.json`](file:///Users/apple/code/WebProjects/daisy-ui/package.json) 里有一个很明确的脚本：

```json
{
  "scripts": {
    "update:deps": "npx taze -r -w"
  }
}
```

这里反映出的不是“只能用 taze 升级”，而是：

- 仓库已经把“依赖更新”作为一项独立维护动作固定下来
- 它不是靠开发者随手 `pnpm add foo@latest` 零散推进

这种做法对长期维护的价值在于：

- 升级动作更容易被识别成一个独立变更面
- 团队可以围绕固定入口建立审查习惯
- 更新后配套的 lockfile 变化、类型检查、构建验证更容易形成完整链路

### 15.4 版本发布链路也已经被做成仓库内工作流

当前根脚本里还有一条非常关键的发布相关命令：

```json
{
  "scripts": {
    "version": "pnpm exec changeset version && pnpm install --no-frozen-lockfile"
  }
}
```

配合：

- [`.changeset/config.json`](file:///Users/apple/code/WebProjects/daisy-ui/.changeset/config.json)
- [`.github/workflows/changeset-version.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/workflows/changeset-version.yml)
- [`.github/workflows/release-tag.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/workflows/release-tag.yml)

可以看出当前仓库并不是把版本发布视为“改完 package.json 手工打 tag”。

它的治理链路更接近：

1. changeset 记录变更意图
2. workflow 生成版本变更 PR
3. 执行版本写回
4. 更新 lockfile
5. 通过 release workflow 生成 tag / release

这类发布流程的长期价值在于：

- 版本变更有专门记录入口
- 内部包联动升级更可控
- 发布动作与依赖解析结果保持同步

### 15.5 `.changeset/config.json` 透露出当前仓库的联动发布策略

当前 [`.changeset/config.json`](file:///Users/apple/code/WebProjects/daisy-ui/.changeset/config.json) 中有几个很关键的点：

```json
{
  "fixed": [["@vben-core/*", "@vben/*"]],
  "updateInternalDependencies": "patch",
  "privatePackages": { "version": true, "tag": true },
  "baseBranch": "main"
}
```

这些设置说明当前仓库的发布治理并不是“每个包完全独立，各发各的”，而是已经定义了：

- 某些包组需要联动版本
- 内部依赖更新应如何传播
- 私有包是否参与 version / tag 流程
- 主发布基线是哪条分支

这类配置的意义不在于“会不会发包”，而在于：

- 版本传播规则是否显式
- 多包协同时是否还能维持一致性

### 15.6 lockfile 审查是长期维护的中心动作之一

长期维护里最容易被低估的对象是 `pnpm-lock.yaml`。

很多团队在 code review 时会重点看：

- 代码变更
- 配置变更

但对 lockfile 只做“自动生成文件，略过”。这在 `pnpm` 体系里通常是不够的。

因为 lockfile 记录的是：

- 最终解析结果
- registry 来源
- peer 上下文分化后的包实例
- 当前工作区所有依赖图的物化状态

因此，长期维护里至少应该把 lockfile 视为：

- 安装行为的审查对象
- 供应链变化的审查对象
- 版本治理是否真正生效的审查对象

### 15.7 当前仓库已经把 lockfile 一致性放进 CI 硬约束

这一点在 [`.github/actions/setup-node/action.yml`](file:///Users/apple/code/WebProjects/daisy-ui/.github/actions/setup-node/action.yml) 里非常明确：

```bash
pnpm install --frozen-lockfile
```

这条命令的长期维护意义非常直接：

- 如果某次变更要求修改 lockfile
- 但 lockfile 没被同步提交
- CI 会直接失败

也就是说，当前仓库已经把“lockfile 不是可选产物”这件事制度化了。

对长期治理来说，这是一个非常核心的动作，因为它把：

- 本地解析结果
- 仓库提交结果
- CI 解析结果

三者强制压在同一条线上。

### 15.8 审查 lockfile 时真正该看的是什么

lockfile 审查不是把整个文件从头到尾读一遍，而是要有稳定关注点。

在 `pnpm` 语境里，更值得重点看的是：

- 新增了哪些包
- 关键基础依赖是否被意外抬升或分裂
- registry / source 是否发生变化
- peer 上下文是否新增了大量实例
- 某次升级是否真的只影响预期范围

这也是为什么前面第 10、11 章要把：

- `catalog`
- `overrides`
- peer 分裂
- 布局与解析策略

讲清楚。因为不理解这些机制，就很难读懂 lockfile 变化到底意味着什么。

### 15.9 版本收敛不是“只剩一个版本”，而是“有意识地收敛”

长期治理里另一个常见误区，是把“全仓所有包都只有一个版本”当成唯一目标。

对 `pnpm` 来说，更准确的目标应该是：

- 能共享的尽量共享
- 必须分裂的知道为什么分裂
- 关键宿主和基础库的入口尽量集中

当前仓库已经在这方面做了几层动作：

- `catalog`
  - 统一外部版本入口
- `overrides`
  - 收敛关键依赖的最终解析
- `dedupePeerDependents: true`
  - 尽量减少非必要 peer 实例分化

因此，版本收敛的正确理解不是“机械去重”，而是：

- 把本来应该一致的部分收敛起来
- 把必须不一致的部分保持可解释

### 15.10 安装脚本信任必须进入长期治理面

从 `pnpm v10` 开始，安装脚本治理已经不是边缘问题。

官方供应链安全文档明确建议：

- 不要无差别允许所有依赖执行 `postinstall`
- 更稳的方式是显式维护 `allowBuilds`

当前仓库已经这样做了，在 [`pnpm-workspace.yaml`](file:///Users/apple/code/WebProjects/daisy-ui/pnpm-workspace.yaml) 中维护了：

- `allowBuilds`

这说明当前仓库已经把一个关键治理点固化下来：

- 哪些包的构建脚本被信任
- 不是安装时临时拍脑袋决定

从长期维护角度看，这一点非常重要。因为依赖树会变，供应链风险面也会变，而安装脚本正是最敏感的执行入口之一。

### 15.11 `minimumReleaseAge` 是另一类供应链治理位

官方供应链安全文档还明确指出：

- `minimumReleaseAge` 可以延迟安装新发布版本
- 在 `pnpm v11` 中默认值为 `1440` 分钟，也就是 1 天

当前仓库虽然已经配置了这个字段，但值为：

```yaml
minimumReleaseAge: 0
```

这代表当前仓库的选择不是“完全没有这项能力”，而是：

- 治理位已经存在
- 当前策略上选择不启用冷静期

这在长期维护上意味着，若将来团队需要提高供应链保守程度，当前配置结构已经有现成落点，不需要重新设计配置模型。

### 15.12 团队规则必须把“唯一包管理器”写死

当前根脚本里有一条很典型的约束：

```json
{
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

这条规则的重要性非常高，因为 Monorepo 一旦允许：

- 有人用 `npm`
- 有人用 Yarn
- 有人用 `pnpm`

那么：

- lockfile 会分裂
- 安装布局会分裂
- CI 与本地行为会分裂

当前仓库显式禁止这一点，实际上是在保护：

- lockfile 唯一性
- 安装结果可复现性
- 团队协作的一致操作面

### 15.13 团队规则还应包括“什么时候允许改依赖入口”

对这类 Monorepo 来说，长期维护里真正敏感的，不只是升级版本，而是修改依赖入口本身。

例如：

- 把一个依赖从 `catalog:` 改成直接版本号
- 把 `workspace:*` 改成 registry 版本
- 删除某条 `overrides`
- 扩大 `publicHoistPattern`
- 新增一个 `allowBuilds`

这些动作的影响通常比“把 `vite` 从 `8.2.1` 升到 `8.2.2`”更大。

因此，团队规则至少应默认把这些动作视为：

- 需要明确评审理由的治理级变更

当前仓库虽然没有把这些规则写成单独文档，但它已经通过集中配置把这些入口聚拢到了少数几个文件中：

- `pnpm-workspace.yaml`
- 根 `package.json`
- `.changeset/config.json`

这本身就为治理提供了很好的审查落点。

### 15.14 “重装一遍看看”不能代替治理

当前根脚本里有：

```json
{
  "scripts": {
    "reinstall": "pnpm run clean -- --del-lock && pnpm install"
  }
}
```

这类脚本在工程里是有价值的，因为它能帮助：

- 清空残留状态
- 重新建立安装结果

但从长期维护角度看，它不能代替治理本身。

原因很简单：

- 重装只能重建当前状态
- 不能回答版本入口是否合理
- 不能回答 lockfile 变化是否可接受
- 不能回答供应链权限是否应被放行

因此，`reinstall` 是维护工具，不是治理策略。

### 15.15 长期维护最终要落到固定检查点

把这一章压缩到可执行层，长期维护至少应形成几类固定检查点：

- **升级前**
  - 看改动是否应走 `catalog` / `overrides` 入口
- **升级后**
  - 看 lockfile 变化是否符合预期
- **合并前**
  - 让 CI 用 `--frozen-lockfile` 验证结果
- **新增脚本执行权限时**
  - 明确审查 `allowBuilds`
- **调整供应链保守度时**
  - 审查 `minimumReleaseAge` 等策略位
- **发布前**
  - 让 changeset、版本写回和 lockfile 同步推进

这些检查点一旦固定，团队对依赖树的控制才会从“凭经验”变成“凭制度”。

### 15.16 本章结论

依赖治理与长期维护的核心，不是掌握更多命令，而是把少数关键入口和关键检查点稳定下来。

这一章可以收敛为九个结论：

1. 长期维护关注的是版本入口、解析结果、信任边界和团队协作一致性，而不只是某次安装是否成功
2. 当前仓库已经把外部版本入口集中到 `catalog`，并用 `overrides` 收敛关键依赖的最终解析
3. `update:deps`、changeset 和 release workflow 共同构成了当前仓库的升级与发布入口
4. `pnpm-lock.yaml` 是长期治理的核心审查对象，不能把它当成可忽略的自动生成文件
5. `pnpm install --frozen-lockfile` 已经把 lockfile 一致性固化进当前仓库的 CI 规则
6. 版本收敛的目标不是机械去成单版本，而是让应一致的部分一致、必须分裂的部分可解释
7. `allowBuilds` 与 `minimumReleaseAge` 分别代表安装脚本信任治理和新版本冷静期治理
8. `preinstall: only-allow pnpm` 这种规则是 Monorepo 长期保持安装一致性的基础约束
9. 真正需要重点评审的，往往不是普通版本号变更，而是修改 `catalog`、`workspace:*`、`overrides`、`publicHoistPattern`、`allowBuilds` 这类治理入口的动作

## 16. 附录：参数归属索引

这一章不是重新解释原理，而是回答一个非常具体的问题：

> 某个字段到底写在哪个文件里，它影响哪一层，正文应该看哪一章。

为了避免附录再次写成一份混乱清单，下面的索引统一回答四件事：

1. 字段写在哪
2. 它解决什么问题
3. 它影响哪一层
4. 应该回看本书哪一章

### 16.1 `package.json` 相关字段

| 字段 | 写在哪 | 作用 | 影响层 | 正文章节 |
|---|---|---|---|---|
| `dependencies` | `package.json` | 声明运行时直接依赖 | 单包依赖声明 | 第 2、5 章 |
| `devDependencies` | `package.json` | 声明开发、构建、测试依赖 | 单包依赖声明 | 第 2、5 章 |
| `peerDependencies` | `package.json` | 声明宿主依赖约束 | 单包依赖声明 / peer 解析 | 第 5、9 章 |
| `peerDependenciesMeta` | `package.json` | 补充 peer 元信息，例如 `optional` | 单包依赖声明 | 第 5、9 章 |
| `optionalDependencies` | `package.json` | 声明可选依赖 | 单包依赖声明 | 第 2、5 章 |
| `dependenciesMeta` | `package.json` | 为已声明依赖补充额外行为信息 | 单包依赖声明 | 第 5 章 |
| `dependenciesMeta.<dep>.injected` | `package.json` | 控制 workspace 依赖是否以注入副本形式安装 | peer 上下文 / workspace 依赖落点 | 第 5、9 章 |
| `packageManager` | `package.json` | 固定项目使用的包管理器与版本 | 工具链版本基线 | 第 4 章 |
| `devEngines.packageManager` | `package.json` | 约束开发环境的包管理器版本 | 工具链版本基线 | 第 4 章 |
| `engines.node` | `package.json` | 声明 Node.js 兼容范围 | 运行时 / 工具链版本基线 | 第 4 章 |
| `engines.pnpm` | `package.json` | 声明 pnpm 兼容范围 | 工具链版本基线 | 第 4 章 |
| `workspace:*` / `workspace:^` / `workspace:~` | `package.json` | 显式声明工作区内部依赖 | workspace 本地依赖契约 | 第 8、13 章 |
| `catalog:` | `package.json` | 引用 workspace 级统一版本入口 | 外部依赖版本治理 | 第 10、13 章 |
| `scripts` | `package.json` | 定义项目或工作区的命令入口 | 日常操作 / CI / 发布 | 第 12、13、15 章 |

### 16.2 `pnpm-workspace.yaml` 相关字段

| 字段 | 写在哪 | 作用 | 影响层 | 正文章节 |
|---|---|---|---|---|
| `packages` | `pnpm-workspace.yaml` | 定义 workspace 成员匹配规则 | 工作区边界 | 第 8、13 章 |
| `catalog` | `pnpm-workspace.yaml` | 定义默认版本入口表 | 外部依赖版本治理 | 第 10、13、15 章 |
| `catalogs` | `pnpm-workspace.yaml` | 定义多组命名版本入口表 | 外部依赖版本治理 | 第 10 章 |
| `catalogMode` | `pnpm-workspace.yaml` | 控制 `pnpm add` 如何使用 catalog | 版本入口治理流程 | 第 10 章 |
| `catalogPrune` | `pnpm-workspace.yaml` | 清理不再被引用的 catalog 条目 | 版本入口治理整洁度 | 第 10 章 |
| `overrides` | `pnpm-workspace.yaml` | 改写最终依赖图，可统一版本、替换来源、删除依赖 | 最终解析结果治理 | 第 10、13、15 章 |
| `packageExtensions` | `pnpm-workspace.yaml` | 修补上游包清单元数据 | 包元数据治理 | 第 10 章 |
| `patchedDependencies` | 顶层清单 / workspace 配置体系 | 指定要应用的 patch 文件 | 上游源码治理 | 第 10 章 |
| `publicHoistPattern` | `pnpm-workspace.yaml` | 把匹配依赖公开提升到根 `node_modules` | 工具链兼容 / 公开入口层 | 第 11、13、15 章 |
| `hoist` | `pnpm-workspace.yaml` | 控制是否提升到隐藏提升层 `node_modules/.pnpm/node_modules` | 默认安装布局兼容 | 第 11 章 |
| `nodeLinker` | `pnpm-workspace.yaml` | 选择总安装模型：`isolated` / `hoisted` / `pnp` | 总安装布局 | 第 11 章 |
| `strictPeerDependencies` | `pnpm-workspace.yaml` | peer 不满足时是否直接失败 | peer 错误策略 | 第 11 章 |
| `autoInstallPeers` | `pnpm-workspace.yaml` | 缺失的非 optional peer 是否自动补装 | peer 安装策略 | 第 11 章 |
| `dedupePeerDependents` | `pnpm-workspace.yaml` | 解析后尽量收敛 peer 相关实例 | peer 结果收敛 | 第 11、15 章 |
| `verifyDepsBeforeRun` | `pnpm-workspace.yaml` | `pnpm run` / `pnpm exec` 前是否检查依赖状态 | 运行前依赖一致性 | 第 11 章 |
| `allowBuilds` | `pnpm-workspace.yaml` | 审批哪些依赖安装时允许执行脚本 | 安装脚本信任治理 | 第 10、12、13、15 章 |
| `minimumReleaseAge` | `pnpm-workspace.yaml` | 设置新版本发布后需等待多久才允许安装 | 供应链冷静期治理 | 第 11、14、15 章 |
| `minimumReleaseAgeExclude` | `pnpm-workspace.yaml` | 指定不受 `minimumReleaseAge` 限制的包 | 供应链冷静期豁免 | 第 14、15 章 |

### 16.3 `.npmrc` 相关字段

| 字段 | 写在哪 | 作用 | 影响层 | 正文章节 |
|---|---|---|---|---|
| `registry` | `.npmrc` | 指定默认 registry 或镜像源 | 下载源路由 | 第 2、14 章 |
| `@scope:registry` | `.npmrc` | 为指定 scope 指向特定 registry | 多源 / 私服路由 | 第 14 章 |
| `//registry...:_authToken` | `.npmrc` / 用户级 auth 文件 | 为某个 registry 设置 token | 私服认证 | 第 14 章 |
| `_auth` / `_password` / `username` | `.npmrc` / 用户级 auth 文件 | 旧式或特定 registry 的认证方式 | 私服认证 | 第 14 章 |
| `always-auth` | `.npmrc` | 控制访问 registry 时是否总附带认证信息 | 私服认证行为 | 第 14 章 |

说明：

- 非敏感源配置可以进入项目级 `.npmrc`
- 敏感凭证更适合放在用户级 auth 文件或 CI secret 注入链路

### 16.4 `pnpm-lock.yaml` 中应重点认识的对象

`pnpm-lock.yaml` 不是“可手工随便写配置”的地方，它是安装解析结果快照。  
因此，这一节列的是观察对象，而不是建议手工编辑的参数。

| 对象 | 所在文件 | 含义 | 影响层 | 正文章节 |
|---|---|---|---|---|
| `lockfileVersion` | `pnpm-lock.yaml` | 当前 lockfile 格式版本 | 安装结果快照格式 | 第 6 章 |
| `settings` | `pnpm-lock.yaml` | 安装解析相关设置快照 | 解析结果上下文 | 第 6 章 |
| `catalogs` | `pnpm-lock.yaml` | catalog 解析结果在 lockfile 中的记录 | 版本入口解析结果 | 第 6、10 章 |
| `importers` | `pnpm-lock.yaml` | 每个 workspace 项目的依赖快照 | 工作区项目安装结果 | 第 6、13 章 |
| `packages` | `pnpm-lock.yaml` | 具体解析到的包实例集合 | 最终依赖图物化结果 | 第 6、7、9 章 |

### 16.5 常用命令选项索引

这一节只补充那些非常容易忘记“它会改哪里”的高频选项。

| 命令 / 选项 | 改动对象 | 作用 | 正文章节 |
|---|---|---|---|
| `pnpm install --frozen-lockfile` | 只允许重建安装结果，不允许改 lockfile | 保证解析结果与提交一致 | 第 6、12、13、14、15 章 |
| `pnpm install --lockfile-only` | 只写 `pnpm-lock.yaml` | 更新解析结果，不落安装目录 | 第 6、12 章 |
| `pnpm install --offline` | 基于本地 store 安装 | 禁止联网安装 | 第 12、14 章 |
| `pnpm add --save-peer` | `package.json` | 把依赖写入 `peerDependencies`，并作为 dev dependency 安装 | 第 12 章 |
| `pnpm add --save-catalog` | `pnpm-workspace.yaml` + `package.json` | 直接把新依赖纳入 catalog 入口治理 | 第 10、12 章 |
| `pnpm add -w` | 根 `package.json` | 显式允许把依赖加到 workspace 根包 | 第 12 章 |
| `pnpm update --no-save` | 主要改 lockfile | 不改声明范围，只推进解析结果 | 第 12 章 |
| `pnpm dedupe --check` | 不写文件，做检查 | 判断 lockfile 是否还能收敛 | 第 12、15 章 |
| `pnpm fetch --prod` | store / virtual store | 预热生产依赖缓存 | 第 12、14 章 |
| `pnpm approve-builds` | `pnpm-workspace.yaml` | 更新 `allowBuilds` 信任列表 | 第 12、15 章 |

### 16.6 如何使用这份索引

如果遇到的是下面几类问题，可以直接按这个顺序定位：

- **“这个字段写在哪个文件里？”**
  - 先看本章
- **“它影响的是声明、解析还是布局？”**
  - 看“影响层”列
- **“应该回去看哪一段原理解释？”**
  - 看“正文章节”列

附录的目的不是取代正文，而是给正文建立一张稳定坐标表。只要字段归属和影响层不再混乱，整本 `pnpm` 文档就不会重新退回“术语突然冒出来、读者不知道它属于谁”的状态。
