# Harness 是什么架构？——DeepSeek Harness 架构洞察分析

> **2026-09-07 源码复核补充**：本文部分接口示例与版本描述需校正，不能直接作为可编译模板。请优先阅读[场景化记忆接入洞察与实施方案](./DeepSeek-Harness场景化记忆接入洞察与实施方案_2026-09.md)，其中第 3、4、12 节提供当前源码契约、Mem0 原生 DSH 插件状态及具体勘误。

> **文档定位**：回答"Harness 是一个架构吗？"这个问题，从架构范式、内核设计、核心不变量、对比分析四个维度做深度洞察。
>
> **撰写日期**：2026年9月7日
> **信息来源**：DeepSeek Harness 官方架构文档（`docs/architecture.md`）、README、社区技术分析（dshkit.dev、springbrand.ai、deepseekdocs.com）、Cordis 学术论文引用

---

## 目录

1. [结论：Harness 不是一个架构，是一种架构范式](#1-结论harness-不是一个架构是一种架构范式)
2. [三层分离：Model vs Harness vs Kernel](#2-三层分离model-vs-harness-vs-kernel)
3. [Cordis 内核：时空可组合性编程范式](#3-cordis-内核时空可组合性编程范式)
4. [核心不变量：四个架构定理](#4-核心不变量四个架构定理)
5. [插件树组装：从空列表到运行时](#5-插件树组装从空列表到运行时)
6. [Turn 流：Agent 循环的精确规范](#6-turn-流agent-循环的精确规范)
7. [能力缝接：Seam 设计模式](#7-能力缝接seam-设计模式)
8. [对比分析：Harness vs Framework vs Runtime](#8-对比分析harness-vs-framework-vs-runtime)
9. [架构洞察：为什么这个设计重要](#9-架构洞察为什么这个设计重要)
10. [对记忆模块开发的架构含义](#10-对记忆模块开发的架构含义)

---

## 1. 结论：Harness 不是一个架构，是一种架构范式

**Harness 不是一个具体的软件架构（如 MVC、微服务），而是一种围绕 LLM 构建智能体的架构范式（Paradigm）。**

具体来说，"Harness 架构"包含三个层面的主张：

| 层面 | 主张 | 含义 |
|---|---|---|
| **职责分离** | Agent = Model + Harness | 模型只负责生成下一个 token；Harness 负责其余一切——工作空间、工具、权限、记忆、循环 |
| **无特权核心** | Everything is a Plugin | 不存在不可替换的核心代码；模型适配器、工具注册表、会话日志、Agent 循环本身都是插件 |
| **可组合性** | 时序可组合 + 空间可组合 | 插件可以挂载/卸载/热替换，卸载时所有副作用自动撤销；插件在兄弟变化时重新解析依赖 |

DeepSeek 自己的表述更直接：

> **"Harness 是你套在模型外面的东西；DeepSeek 的赌注是你应该能够重新套上它的每一个部分，而不需要 fork。"**

这不是一个功能描述，而是一个**架构声明**——"harness" 比 "app" 更准确，因为它不是应用程序，而是围绕模型的可重构外壳。

---

## 2. 三层分离：Model vs Harness vs Kernel

dsh 的设计明确区分三个层级，每一层都有清晰的"做什么"和"不做什么"：

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  01  模型层 (Model Layer)                                    │
│                                                             │
│  ✓ 做什么：生成下一个 token、推理任务、决定调用哪个工具       │
│  ✗ 不做什么：不能打开文件、不能运行命令、不能自己记住上一轮     │
│  实现：DeepSeek V4-Pro / V4-Flash / GPT / Claude / 任意模型   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  02  线束层 (Harness Layer = dsh)                             │
│                                                             │
│  ✓ 做什么：                                                  │
│     · 给模型一个工作空间                                      │
│     · 维护工具注册表和受保护的执行管道                          │
│     · 管理沙箱和审批策略                                      │
│     · 维护只追加的会话日志                                     │
│     · 驱动让工作持续推进的 Agent 循环                          │
│  ✗ 不做什么：不训练、不托管、不替代模型；不自带权重            │
│  实现：TypeScript / Node.js，所有能力来自插件组合              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  03  内核层 (Kernel Layer = Cordis)                           │
│                                                             │
│  ✓ 做什么：                                                  │
│     · 挂载、卸载、重连插件                                     │
│     · 追踪插件依赖关系                                        │
│     · 分发插件间通信的类型化事件                               │
│     · 保证注册效果在插件卸载时自动撤销                         │
│  ✗ 不做什么：不特定于 DeepSeek、不特定于 Agent                 │
│  实现：通用插件框架，设计发表于学术论文                         │
│     "A Programming Paradigm for Spatiotemporal Composability" │
│     (arXiv:2608.25512)                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**关键洞察**：Cordis 不是 DeepSeek 为这个项目专门构建的东西，而是一个通用的插件框架。dsh 只是恰好构建在它之上。这意味着 Harness 架构范式理论上可以被复制到任何语言和领域。

---

## 3. Cordis 内核：时空可组合性编程范式

### 3.1 学术基础

Cordis 的设计发表为学术论文：**《A Programming Paradigm for Spatiotemporal Composability》**（arXiv:2608.25512）。这个标题精确地定义了 Cordis 贡献的核心：

- **时间可组合性（Temporal Composability）**：卸载一个插件完全撤销其副作用。移除一个能力，系统回到添加它之前的状态。
- **空间可组合性（Spatial Composability）**：插件在兄弟插件变化时重新解析其依赖。

### 3.2 五个原语

Cordis 围绕五个原语构建：

| 原语 | 作用 | 对应 dsh 中的 |
|---|---|---|
| **Context**（上下文） | 插件共享的公共空间，服务注入和依赖解析的中介 | `ctx` 参数 |
| **ctx.tools** | 工具注册表，模型可调用的能力 | 工具注册和执行管道 |
| **ctx.llm** | 模型适配器接缝，流式消息词汇 | 模型路由和流式传输 |
| **ctx.sessions** | 会话日志和内存存储 | 只追加事件日志 |
| **inject**（注入） | 依赖注入机制，插件声明它需要什么服务 | `ctx.inject([...], handler)` |

### 3.3 插件契约

一个 Cordis 插件就是一个导出 `apply(ctx)` 函数的模块：

```typescript
export function apply(ctx: Context) {
  // 1. 注册服务
  ctx.services.register('my-capability', { ... })

  // 2. 订阅事件
  ctx.events.on('agent/step', (event) => { ... })

  // 3. 注册可逆效果（卸载时自动撤销）
  ctx.effect(() => {
    const timer = setInterval(() => { ... }, 5000)
    return () => clearInterval(timer)  // ← 卸载时执行
  })
}
```

**核心设计规则**：这个机制对 dsh 的每个部分一视同仁——模型适配器、工具注册表、会话日志、Agent 循环都是插件，都可以从配置中替换。

### 3.4 为什么这不只是"插件系统"

传统框架（如 VS Code、Eclipse）有插件系统，但它们有一个**受保护的核心**，插件只能通过核心暴露的 API 扩展功能。

Cordis 的不同之处在于：**没有受保护的核心**。

| 维度 | 传统插件系统 | Cordis |
|---|---|---|
| 核心 | 受保护的特权代码 | 无特权核心，一切皆插件 |
| 扩展方式 | 通过核心暴露的 API | 在其他插件旁边挂载一个插件 |
| 卸载 | 注册可能留下残余状态 | 注册是效果，卸载时完全撤销 |
| 依赖 | 静态声明 | 兄弟变化时重新解析 |
| Agent 循环 | 框架固有的固定代码 | 本身是一个可替换的插件 |

---

## 4. 核心不变量：四个架构定理

dsh 的架构文档明确定义了四个必须始终成立的不变量：

### 定理 1：无特权核心

> **"没有可打补丁的特权核心：你通过在其他插件旁边挂载一个插件来扩展 dsh，注册是插件卸载时撤销的效果。"**

含义：任何能力——包括 Agent 循环本身——都可以被替换。这不是 API 契约，这是架构约束。

### 定理 2：模型可见 = 已记录

> **"任何到达模型请求的内容都必须能从日志重建。运行时不变量会断言这一点。"**

含义：没有侧信道可以让信息进入模型上下文。如果你想注入记忆，你必须通过会话事件系统完成，确保可追溯、可回放。

执行机制：
1. **结构性保证**：模型请求只能从日志推导，没有第二条代码路径
2. **写入前校验**：每个事件在被追加到日志前都经过验证
3. **运行时断言**：断言插件逐字节比对外发请求与从日志重建的请求，不匹配则中止

### 定理 3：后层覆盖前层

> **"补丁按行替换整个配置值，不做深度合并。后层胜出。"**

含义：配置组合是按层的、有序的。Patch 替换整行配置，不深度合并键值。要改一个字段必须重述该行所有键。

层叠顺序：
```
1. 各 bundle 的 cordis.patch.yml（按 profile 列出的顺序）
2. Profile 自身的 cordis.patch.yml
3. Home 级 cordis.patch.yml
4. 命令行 --patch 叠加
```

### 定理 4：一个 Bundle 永远是起始位置

> **"Bundle 插入的任何内容都被上层保留可补丁性。Bundle 是起始位置，永远不是密封的盒子。"**

含义：你安装的任何插件包都不能"锁定"自己的行为。上层的 Patch 总是可以覆盖它插入的配置行。这是架构保证，不是可选的。

---

## 5. 插件树组装：从空列表到运行时

dsh 的启动过程是架构范式最精确的体现：

```
启动 dsh --profile web
       │
       ▼
┌──────────────────────────────────────────┐
│  Step 1: 读取 Profile 配置                 │
│  ~/.dsh/profiles/web/                     │
│  · bundle 列表（有序）：[dsh-base, dsh-web-app] │
│  · 已安装的 npm 包（第三方插件）            │
│  · cordis.patch.yml（用户自定义补丁）       │
│                                          │
│  根配置文件包含一个空条目列表：[]            │
│  ← 这不是占位符，这是设计哲学              │
│  整个插件树由应用补丁层产生               │
└──────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────┐
│  Step 2: 按序应用 Patch 层                 │
│                                          │
│  Layer 1: dsh-base 的 cordis.patch.yml    │
│    → 插入：模型适配器、工具、持久化、       │
│      沙箱和审批策略、设置、凭据、遥测       │
│                                          │
│  Layer 2: dsh-web-app 的 cordis.patch.yml │
│    → 插入：浏览器应用程序（Web UI）         │
│                                          │
│  Layer 3: Profile 的 cordis.patch.yml    │
│    → 用户自定义：替换/插入特定行            │
│                                          │
│  Layer 4: Home 级 cordis.patch.yml       │
│    → 机器级覆盖                           │
│                                          │
│  Layer 5: 命令行 --patch 叠加             │
│    → 临时覆盖                             │
│                                          │
│  规则：后层按行覆盖前层，整行替换，不深度合并 │
└──────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────┐
│  Step 3: 组装插件树                        │
│                                          │
│  最终的配置树 = 所有层叠加的结果           │
│  每一行都可以用 --dump-config 看到         │
│  每一行都可以被你的 patch 替换             │
│                                          │
│  dsh --profile web --dump-config          │
│  → 打印整个运行系统的插件树                │
└──────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────┐
│  Step 4: 挂载所有插件                      │
│                                          │
│  Cordis 按配置树挂载每个插件               │
│  · 解析依赖（空间可组合性）                │
│  · 注册服务、事件、效果                    │
│  · 效果是可逆的（时间可组合性）             │
└──────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────┐
│  Step 5: 运行 Agent 循环                   │
│  感知 → 推理 → 行动 → 学习 ↺              │
└──────────────────────────────────────────┘
```

**关键洞察**：根配置文件包含 `[]`（空列表）不是技术细节，而是**设计哲学的具体体现**。dsh 是什么，完全由哪些补丁被应用、以什么顺序应用决定。没有默认的"dsh"——只有"这些补丁组合出来的 dsh"。

### Profile 的四种预设

| Profile | 叠加的 Bundle | 产物 |
|---|---|---|
| `web` | dsh-base + dsh-web-app | Web 工作台（127.0.0.1:3080） |
| `headless` | dsh-base + dsh-headless | 无头一次性运行器 |
| `sdk` | dsh-base + dsh-sdk-app | SDK JSON-RPC 服务器 |
| `sdk-minimal` | dsh-sdk-minimal（独立） | 最小 SDK 树，不应用 dsh-base |

`web` 和 `headless` 的**唯一区别**是堆叠了 dsh-web-app 还是 dsh-headless。同一个内核、同一个 dsh-base，不同的 UI bundle。

---

## 6. Turn 流：Agent 循环的精确规范

dsh 架构文档精确定义了 Agent 循环的事件流，这是理解"Harness 做什么"的核心：

### 6.1 基本概念

| 概念 | 定义 |
|---|---|
| **Step（步骤）** | 一次模型请求 + 它调用的工具 |
| **Turn（轮次）** | 零个或多个步骤：在第一个输入被认领前打开，在不欠任何工作时关闭 |

### 6.2 完整事件流

```
turn/start
  │
  ├─ 认领下一步输入 + 一个排队消息
  ├─ 组装提示词段落 + 工具 schema
  │
  ├─→ agent/pre-step                    ← 拦截点：可重写或拒绝消息
  │   ├─ reject → 关闭 turn（不执行步骤）
  │   └─ enter(messages, startsRequestSeries?)
  │       └─ 如果 enter 被重写为空 → 关闭 turn（不执行步骤）
  │
  ├─ step/start
  │   ├─ 将 entered messages 作为 user/message 追加
  │   ├─ 从日志推导模型历史
  │   │
  │   ├─→ agent/request → llm/stream → agent/assistant-stream start
  │   │    ├─ agent/assistant-stream chunk* (瞬时帧)
  │   │    ├─ assistant/message | assistant/attempt
  │   │    └─ agent/assistant-stream end
  │   │
  │   ├─ tool/call* → tools/pre-execute → tools/execute → tools/post-execute → tool/result*
  │   │
  │   └─ step/end
  │
  ├─ 如果工具欠另一个请求，或下一步输入到达 → 认领 → 下一步
  │
  ├─→ agent/turn-stopping               ← 串行事件，无 next()
  │
  └─ turn/end
```

### 6.3 事件分类

| 类型 | 事件 | 特征 |
|---|---|---|
| **持久会话事件** | `turn/*`、`step/*`、`user/message`、`assistant/message`、`assistant/attempt`、`tool/*` | 写入只追加日志，跨重载存活 |
| **实时扩展点** | `agent/*`、`tools/*`、`llm/stream` | 进程内事件，用于观察/拦截进行中的工作 |
| **瀑布型** | `agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` | 监听器必须调用 `next()` 委托 |
| **串行型** | `agent/turn-stopping` | 无 `next()`，串行执行 |

### 6.4 关键设计：注入机制

> **"输入通过一个收件箱到达驱动器。一些消息立即唤醒它；注入的上下文在收件箱中等待，直到另一条消息到来。"**

`agent/pre-step` 决定模型看到什么：
- 监听器可以**重写**认领的消息
- 监听器可以**拒绝**消息（被拒绝或空的首个认领仍然关闭一个不执行步骤的 turn，日志记录尝试）
- enter 决策可以设置 `startsRequestSeries` 开始新的模型消息系列

`agent.inject()` 是添加模型可见上下文的唯一正规路径——它落在下一个被准入的请求中。

---

## 7. 能力缝接：Seam 设计模式

### 7.1 什么是 Seam

**Seam（缝接）** 是 dsh 中可替换能力的抽象接口，有三个角色：

| 角色 | 职责 | 类比 |
|---|---|---|
| **Service Definition（服务定义）** | 声明接口 | 接口/Protocol |
| **Service Provider（服务提供者）** | 实现接口 | 具体类 |
| **Consumer（消费者）** | 使用接口 | 调用方 |

> **"一个包可以合并角色，但只有一个角色不是 Seam；添加一个能力意味着设计全部三个。"**

### 7.2 核心 Seam 列表

| Seam | ctx 键 | 作用 |
|---|---|---|
| 模型适配 | `ctx.llm` | 消息和流词汇 + 适配器接缝 |
| 工具注册 | `ctx.tools` | 作用域工具注册表和受保护执行管道 |
| 会话日志 | `ctx.sessions` | 只追加 SessionEvent 日志和内存存储 |
| Agent 管理 | `ctx.agents` | Agent 接口、活跃注册表和 `agent/*` 事件 |
| Agent 循环 | `ctx.agentLoop` | 实现 Agent 接口的默认驱动器 |
| 文件系统 | `ctx.fs` | 文件系统访问和策略 |
| 子进程 | `ctx.subprocess` | 进程生成 |
| Shell | `ctx.shell` | Shell 执行后端 |
| 终端 | `ctx.terminals` | 持久终端执行 |
| 沙箱 | `ctx.sandbox` | 进程限制 |
| 命令 | `ctx.commands` | 人类命令分发（无需模型轮次） |
| 后台任务 | `ctx.jobs` | 后台工作注册 |
| Webhook | `ctx.webhookRuntime` | 认证投递分发和会话创建 |
| 会话标题 | `ctx.sessionTitle` | 会话标题生成 |
| 目标管理 | `ctx.goals` | 同会话目标管理 |
| 会话投影 | `ctx.sessionProjections` | 增量折叠已提交事件 |

### 7.3 Seam 的连锁效应

> **"文件系统和子进程提供者共享一个执行世界，所以将它们指向远程沙箱会将 Bash、PTY 和 LSP 一起移动，而不需要任何提供者 fork。"**

这就是 Seam 设计的威力——**一个提供者替换改变整个产品**。你不需要为远程沙箱分别 fork Bash 工具、PTY 工具和 LSP 工具；你只需要替换文件系统和子进程提供者，所有依赖它们的工具自动跟随。

---

## 8. 对比分析：Harness vs Framework vs Runtime

### 8.1 概念对比

| 维度 | Framework（框架） | Runtime（运行时） | Harness（线束） |
|---|---|---|---|
| **核心隐喻** | 地基——你在上面建房子 | 引擎——驱动汽车跑 | 外壳——套在模型身上 |
| **模型关系** | 模型是框架的一个组件 | 运行时包含模型 | Harness 包裹模型，模型是被动方 |
| **可替换性** | 核心通常不可替换 | 运行时通常是完整栈 | 每个部分可重新套上 |
| **扩展方式** | 通过核心暴露的 API | 配置和插件 | 挂载插件 beside 其他插件 |
| **典型代表** | LangChain、AutoGen | Claude Code（固定外壳） | DeepSeek Harness（dsh） |

### 8.2 dsh vs Claude Code vs Codex CLI

| 维度 | Claude Code / Codex CLI | DeepSeek Harness (dsh) |
|---|---|---|
| **Harness** | 固定外壳，通过 MCP/API 扩展 | Harness 本身由插件构成 |
| **模型** | 引力中心，不可替换 | 一个插件行，删除即换 |
| **Agent 循环** | 框架固有代码 | 可替换的插件 |
| **切换模型** | 权限重置、记忆丢失、命令失效 | 删除模型行，其他不动 |
| **UI** | 产品固有 | 一个 Bundle，可替换 |
| **沙箱策略** | 固有行为 | dsh-base 插入的一行，可替换 |
| **切换成本** | 高——整个栈耦合 | 低——只换你需要换的行 |

> **社区观察**："从 Claude Code 切换到 GPT 驱动的 agent，你很快发现：模型从来不是整个产品。你的权限重置了。你的记忆没了。你的命令不工作了。几个月的习惯一夜之间变得无用。"——这是 dsh 要解决的问题。

### 8.3 dsh 的四种运行模式

| 模式 | 工具集 | 适用场景 |
|---|---|---|
| **Standard** | 完整工具集（文件编辑、shell、搜索、技能、规划、子 agent、工作流） | 日常编码 |
| **Code** | Standard + Code Mode SDK（模型在 TypeScript 中编排多步操作） | 多工具组合 |
| **Minimal** | 仅 bash + str_replace_editor | 模型基准测试 |
| **Creator** | Standard + 运行时检查 + 插件实验 + 预设编写指导 | 编写自定义预设 |

---

## 9. 架构洞察：为什么这个设计重要

### 9.1 "一切皆插件"不是营销——是启动过程的字面描述

> **"在 DeepSeek Harness 中，'一切皆插件'更接近启动过程的字面描述，而不是营销。"**

当 dsh 启动时，它组合一个配置行列表，**每一行都来自一个插件**。`dsh --profile web --dump-config` 打印整个运行系统，里面每一行都是某个 patch 可以定位的目标。这不是 debug 工具——这是架构的核心工作方式。

### 9.2 自修改 Agent 成为可能

Cordis 的时空可组合性买到了两个普通应用永远不给你的保证：

1. **时间可组合性**：卸载插件完全撤销副作用 → Agent 可以在对话中途写一个插件并加载到自身
2. **空间可组合性**：插件在兄弟变化时重新解析依赖 → 替换一个能力不需要 fork 整个系统

这意味着 **Agent 可以自修改**——不是在 prompt 层面，而是在架构层面。一个 Agent 可以在运行中卸载自己的工具、加载新工具、替换自己的循环策略，而所有这些都不需要重启。

### 9.3 会话日志作为唯一真相源的深层含义

"模型可见 = 已记录"这个不变量的含义远超"有日志可查"：

- **可复现**：任何模型请求都可以从日志精确重建
- **可分叉**：从任意 turn 边界 fork 会话
- **可回放**：重放事件流重现完整交互
- **可审计**：逐字节验证外发请求

这对记忆模块的含义是：**记忆注入必须通过事件系统完成**。你不能绕过日志直接修改模型上下文。你的记忆插件注入的内容必须成为会话事件，被记录、可回放、可审计。

### 9.4 代价：组合调试

架构文档和社区分析都诚实地指出了代价：

> **"灵活性不是免费的。"**
>
> 1. 每个插件都是可以移动的 API 表面
> 2. 调试是组合形状的——问题不是"这段代码做了什么"，而是"哪一层设置了这个"
> 3. 没有什么阻止你组合出不一致的东西

`--dump-config` vs `--dump-default-config` 的 diff 是回答"哪一层设置了这个"的工具。

### 9.5 为什么叫 "Harness" 而不是 "App"

> **"Harness 是你套在模型外面的东西；你应该能够重新套上它的每一个部分，而不需要 fork。App 是一个密封的产品；Harness 是一个可重构的外壳。"**

命名是精确的：
- **App**（应用）：密封的产品，有固定功能集
- **Framework**（框架）：你在上面构建的地基
- **Harness**（线束/马具）：套在模型外面的可重构外壳，每一部分可独立替换

---

## 10. 对记忆模块开发的架构含义

基于以上架构洞察，记忆模块开发需要遵循的架构约束：

### 10.1 必须通过事件系统注入

记忆注入到模型上下文**必须**通过 `agent.inject()` 或 `agent/pre-step` 事件完成。不能绕过会话日志直接修改模型上下文——这违反定理 2（模型可见 = 已记录）。

```
合法路径：
  记忆插件 → agent/pre-step 事件 → enter(messages) → 会话日志 → 模型请求

非法路径：
  记忆插件 → 直接修改模型上下文 ← 违反不变量
```

### 10.2 必须作为 Cordis 插件实现

记忆模块不是一个"外挂"或"集成"，而是一个**与其他插件平起平坐的 Cordis 插件**。它通过 `apply(ctx)` 挂载，通过 `ctx.services` 提供服务，通过 `ctx.events` 订阅事件，通过 `ctx.effect` 注册可逆效果。

### 10.3 对接点与事件域映射

| 记忆功能 | 对接的事件 | 事件类型 |
|---|---|---|
| 启动时预取记忆注入 | `session/start` | 持久会话事件 |
| 步骤前检索注入 | `agent/pre-step` | 瀑布型实时扩展点 |
| 步骤后异步提取 | `agent/step` / `step/end` | 持久会话事件 |
| 会话结束批量提取 | `session/end` | 持久会话事件 |
| 主动搜索工具 | 工具注册（`ctx.tools`） | 能力缝接 |
| 主动存储工具 | 工具注册（`ctx.tools`） | 能力缝接 |

### 10.4 必须设计为可卸载

基于时间可组合性定理，记忆插件的卸载必须完全撤销其副作用：
- 取消所有 `setInterval` / `setTimeout`
- 关闭所有数据库连接
- 注销所有服务和工具
- 不在全局状态中留下残余

### 10.5 配置必须通过 Patch 声明

记忆插件的默认配置通过 `cordis.patch.yml` 声明，用户通过 Profile 级 `cordis.patch.yml` 覆盖。记住：**Patch 替换整行，不深度合并**。

---

## 附录 A：核心术语词典

| 术语 | 英文 | 精确定义 |
|---|---|---|
| 线束 | Harness | 套在模型外面的可重构外壳，提供工作空间、工具、权限和运行记忆 |
| 内核 | Kernel (Cordis) | 管理插件挂载/卸载/依赖的通用框架，非特定于 Agent |
| 插件 | Plugin | 导出 `apply(ctx)` 的模块，注册服务/事件/效果 |
| 捆绑包 | Bundle | 声明了 `dsh.bundle` 清单的 npm 包，携带配置行和代码 |
| 配置文件 | Profile | `~/.dsh/profiles/` 下的命名组合体 |
| 补丁 | Patch | YAML 配置行，按 id 定位，替换整行 |
| 缝接 | Seam | 可替换能力的抽象接口，有三个角色 |
| 会话日志 | Session Log | 只追加的事件流，模型上下文的唯一真相源 |
| 步骤 | Step | 一次模型请求 + 它调用的工具 |
| 轮次 | Turn | 零或多个步骤，从认领输入到不欠工作 |
| 时空可组合性 | Spatiotemporal Composability | Cordis 的学术贡献：卸载撤销副作用 + 兄弟变化时重新解析 |

## 附录 B：参考资源

| 资源 | 链接 |
|---|---|
| dsh 架构文档（源） | `github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md` |
| dsh README | `github.com/deepseek-ai/deepseek-harness` |
| Cordis 论文 | `arxiv.org/abs/2608.25512` |
| Cordis 仓库 | `github.com/cordiverse/cordis` |
| DSHKit 社区指南 | `dshkit.dev/plugins/everything-is-a-plugin` |
| SpringBrand 深度分析 | `springbrand.ai/deepseek-harness` |
| DeepSeekDocs 社区文档 | `deepseekdocs.com/en/docs/learn/intro/what-is-dsh` |
| DeepSeek 官方 Harness 页面 | `deepseek.com/harness/en` |
| 插件生态 | `github.com/topics/dsh-plugin` |

---

> **文档版本**：v1.0
> **核心结论**：Harness 不是一种"架构"（如 MVC），而是一种围绕 LLM 构建智能体的**架构范式**——以"无特权核心 + 一切皆插件 + 时空可组合性"为核心主张，以 Cordis 为通用内核，以会话日志为唯一真相源。它与传统"框架+插件API"模式的本质区别在于：**没有不可替换的核心**。
