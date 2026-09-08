# Pi Agent vs DeepSeek Harness：终端线束 vs 插件宇宙

> **文档定位**：调研 Pi Agent（pi-mono）和 DeepSeek Harness（dsh）两个 Agent 框架的架构差异、设计哲学、扩展模型和生态，形成对比分析。
>
> **撰写日期**：2026年9月7日
> **信息来源**：Pi 官方仓库 README + pi.dev 官网 + DeepSeek API 文档 + dsh 架构文档 + Mem0 插件实现对比

---

## 目录

1. [两个框架是什么](#1-两个框架是什么)
2. [设计哲学对比：根本分歧](#2-设计哲学对比根本分歧)
3. [架构对比：两种完全不同的路线](#3-架构对比两种完全不同的路线)
4. [扩展模型对比](#4-扩展模型对比)
5. [运行模式对比](#5-运行模式对比)
6. [记忆插件对比：Mem0 在两个平台的实现](#6-记忆插件对比mem0-在两个平台的实现)
7. [能力矩阵](#7-能力矩阵)
8. [总结：应该选哪个](#8-总结应该选哪个)

---

## 1. 两个框架是什么

### Pi Agent（pi-mono）

| 属性 | 值 |
|---|---|
| **开发者** | Mario Zechner（badlogic），Earendil Inc. |
| **仓库** | `github.com/badlogic/pi-mono` |
| **语言** | TypeScript |
| **许可证** | MIT |
| **定位** | 极简终端编码 Agent Harness |
| **安装** | `npm install -g @earendil-works/pi-coding-agent` |
| **官网** | pi.dev |

一句话：**Pi 是一个极简的终端编码 Agent——核心只给模型四个工具（read/write/edit/bash），其余一切通过 TypeScript 扩展、技能、提示模板和主题按需添加。**

### DeepSeek Harness（dsh）

| 属性 | 值 |
|---|---|
| **开发者** | DeepSeek AI |
| **仓库** | `github.com/deepseek-ai/deepseek-harness` |
| **语言** | TypeScript |
| **许可证** | MIT |
| **定位** | "Everything is a Plugin" 架构范式 Agent Harness |
| **安装** | `npx @deepseek-ai/dsh web` |
| **官网** | deepseek.com/harness |

一句话：**dsh 是一个没有特权核心的 Agent 框架——模型适配器、工具注册表、会话日志、Agent 循环本身都是可替换的插件，通过 Cordis 内核在启动时组装。**

---

## 2. 设计哲学对比：根本分歧

这是两个框架最核心的差异——不是功能多寡，而是**对"Agent 框架应该是什么"的根本回答不同**。

### Pi 的哲学：极简核心 + 激进扩展

> "Pi is a minimal agent harness. Adapt Pi to your workflows, not the other way around."

Pi 的回答是：**核心越小越好，功能通过扩展实现**。

| 主张 | 含义 |
|---|---|
| **No MCP** | 不内置 MCP 协议。写 CLI 工具 + README（Skills）就够了，或用扩展加 MCP 支持 |
| **No sub-agents** | 不内置子 Agent。用 tmux 起 Pi 实例，或用扩展自己建 |
| **No permission popups** | 不内置权限弹窗。跑容器里，或用扩展自己建确认流程 |
| **No plan mode** | 不内置计划模式。写到文件里，或用扩展建 |
| **No built-in to-dos** | 不内置 TODO。它们会混淆模型，用 TODO.md 文件 |
| **No background bash** | 不内置后台 bash。用 tmux，完全可观察 |

Pi 把其他 Agent 框架（Claude Code、Codex）内置的功能都**故意不做**，而是让用户通过 Extensions 自己实现。核心只保留四个工具和一个漂亮的 TUI。

### dsh 的哲学：一切皆插件 + 无特权核心

> "Everything is a plugin. Every run is traceable."

dsh 的回答是：**核心不应该是"最小的"，而应该是"不存在的"——连 Agent 循环都是插件**。

| 主张 | 含义 |
|---|---|
| **无特权核心** | 没有不可替换的代码。模型适配器、工具、会话日志、Agent 循环、UI 都是插件 |
| **每次运行可追溯** | 模型看到的一切都写进只追加的会话日志 |
| **Cordis 内核** | 通用插件框架，管挂载/卸载/依赖，非特定于 Agent |
| **配置即组合** | 不改源码，通过 YAML patch 层替换任何能力 |

### 哲学差异的本质

| 维度 | Pi | dsh |
|---|---|---|
| **核心策略** | 极简（故意不做） | 不存在（一切皆插件） |
| **扩展方式** | TypeScript 扩展模块 | Cordis 插件 + YAML patch 层 |
| **设计原点** | "让用户自己加" | "让用户替换一切" |
| **UI 形态** | 终端 TUI（一等公民） | Web UI（也是插件） |
| **Agent 循环** | 固定（核心代码） | 可替换的插件 |
| **模型适配器** | 内置支持 30+ provider | 一个插件行 |
| **类比** | 精简改装车——底盘最小，自己加件 | 乐高——连底盘都是积木 |

**一句话总结**：Pi 说"我不做这些功能，你来加"；dsh 说"我做的所有功能你都可以换掉"。

---

## 3. 架构对比：两种完全不同的路线

### Pi 的架构

```
┌─────────────────────────────────────────────┐
│              Pi Agent Harness                 │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │        核心运行时（固定）              │   │
│  │  · Agent 循环（固定代码）             │   │
│  │  · 4 个内置工具：read/write/edit/bash│   │
│  │  · Session 管理（JSONL 树结构）       │   │
│  │  · Compaction（上下文压缩）           │   │
│  │  · 30+ Provider 适配（pi-ai 包）      │   │
│  │  · TUI 渲染引擎（pi-tui 包）         │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Extensions│ │  Skills  │ │ Prompts  │  │
│  │ (TS 模块) │ │ (SKILL.md)│ │ (.md 文件)│  │
│  └──────────┘ └──────────┘ └──────────┘  │
│                                             │
│  ┌──────────┐ ┌──────────────────────┐    │
│  │  Themes  │ │  Pi Packages (npm/git)│    │
│  └──────────┘ └──────────────────────┘    │
└─────────────────────────────────────────────┘
```

**关键特点**：
- 核心运行时是**固定的代码**，不可从配置替换
- 扩展通过 `ExtensionAPI` 注册工具/命令/事件/UI
- Session 存为 JSONL 树结构，支持原地分支
- 30+ Provider 内置在 `pi-ai` 包中
- TUI 是一等公民，不是附加品

### dsh 的架构

```
┌──────────────────────────────────────────────┐
│              DeepSeek Harness (dsh)            │
│                                              │
│  启动时从空列表 [] 组装：                      │
│                                              │
│  Layer 1: dsh-base bundle                    │
│    → 模型适配器（插件）                        │
│    → 工具注册表（插件）                        │
│    → 会话日志（插件）                          │
│    → 沙箱策略（插件）                          │
│    → 审批策略（插件）                          │
│    → 设置/凭据/遥测（插件）                    │
│                                              │
│  Layer 2: dsh-web-app bundle                 │
│    → Web UI（插件）                           │
│                                              │
│  Layer 3: 用户 cordis.patch.yml              │
│    → 替换/插入特定行                           │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │        Cordis 内核（通用框架）          │    │
│  │  · 挂载/卸载/依赖追踪                  │    │
│  │  · 时空可组合性                       │    │
│  │  · 注册即效果，卸载即撤销              │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │        模型层（可选任意模型）           │    │
│  │  DeepSeek V4 / GPT / Claude / ...     │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

**关键特点**：
- **根配置是空列表 `[]`**——dsh 是什么完全由插件组合决定
- Agent 循环本身是插件，可以替换
- Cordis 内核不特定于 Agent，是通用插件框架
- 会话日志是唯一真相源——模型可见 = 已记录
- Patch 按行替换整行配置，不深度合并

### 架构差异核心

| 维度 | Pi | dsh |
|---|---|---|
| **Agent 循环** | 固定核心代码 | 可替换插件 |
| **模型适配器** | pi-ai 包内置 30+ | 一个配置行 |
| **工具注册表** | 核心管理 | 插件（`ctx.tools`） |
| **会话日志** | JSONL 树（pi 自有格式） | 只追加事件流（Cordis 事件） |
| **UI** | TUI 内置（pi-tui） | Web UI 是一个 bundle 插件 |
| **沙箱** | 不内置，靠容器 | 插件（dsh-base 提供） |
| **权限** | 不内置，靠扩展 | 插件（dsh-base 提供） |
| **启动方式** | 直接跑 `pi` | 从空列表组装插件树 |

---

## 4. 扩展模型对比

### Pi 的扩展模型

Pi 有四种扩展方式，按复杂度递增：

| 类型 | 形式 | 用途 | 示例 |
|---|---|---|---|
| **Prompt Templates** | Markdown 文件 | 可复用提示，`/name` 展开 | `/review` 代码审查提示 |
| **Skills** | SKILL.md 文件 | 按需加载的能力包，遵循 Agent Skills 标准 | `/skill:deploy` 部署技能 |
| **Themes** | 主题文件 | TUI 外观热重载 | 暗色/亮色主题 |
| **Extensions** | TypeScript 模块 | 完整扩展——工具、命令、事件、UI | 自定义权限门、MCP 集成、子 Agent |

Extensions 的 API：
```typescript
export default function (pi: ExtensionAPI) {
  pi.registerTool({ name: "deploy", ... });
  pi.registerCommand("stats", { ... });
  pi.on("tool_call", async (event, ctx) => { ... });
}
```

**扩展发现**：从 `~/.pi/agent/`（全局）、`.pi/`（项目）、Pi Packages（npm/git）三个位置自动发现。

### dsh 的扩展模型

dsh 的扩展只有一种形式：**Cordis 插件**。

| 概念 | 说明 |
|---|---|
| **Bundle** | 声明了 `dsh.bundle` 的 npm 包，携带 cordis.patch.yml |
| **Plugin** | 导出 `apply(ctx)` 的模块 |
| **Patch** | YAML 配置行，按 id 替换整行 |
| **Profile** | `~/.dsh/profiles/` 下的组装配置 |

Plugin 的 API：
```typescript
export function apply(ctx: Context, config: Config): void {
  ctx.tools.register(defineTool({ name: "search_memory", ... }));
  ctx.events.on("agent/step", (event) => { ... });
  ctx.effect(() => { /* 可逆效果 */ });
}
```

**关键差异**：

| 维度 | Pi Extensions | dsh Cordis Plugins |
|---|---|---|
| **能替换什么** | 只能加/覆盖工具，不能替换核心 | 可以替换一切，包括 Agent 循环 |
| **卸载行为** | 扩展移除，但可能留残余状态 | **完全撤销**——所有副作用自动回退 |
| **依赖管理** | npm 包，手动管 | Cordis 自动追踪，兄弟变化时重新解析 |
| **配置方式** | `package.json` 的 `pi` 字段 | `dsh.bundle` + `cordis.patch.yml` |
| **热重载** | `/reload` 命令 | 某些 Profile 支持 live patch reload |
| **UI 可替换** | 扩展可以替换编辑器、加 widget | UI 本身是 bundle，整个换 |

---

## 5. 运行模式对比

### Pi 的四种模式

| 模式 | 命令 | 用途 |
|---|---|---|
| **Interactive** | `pi` | 完整 TUI 体验 |
| **Print** | `pi -p "query"` | 脚本化，输出后退出 |
| **JSON** | `pi --mode json` | 事件流 JSON lines |
| **RPC** | `pi --mode rpc` | stdin/stdout JSONL 协议，非 Node 集成 |
| **SDK** | `import { createAgentSession }` | 嵌入自己的 Node.js 应用 |

### dsh 的四种模式（Profile）

| Profile | 命令 | 用途 |
|---|---|---|
| **web** | `dsh web` / `dsh --profile web` | Web 工作台（127.0.0.1:3080） |
| **headless** | `dsh --profile headless "task"` | 无头一次性任务 |
| **sdk** | `dsh --profile sdk` | SDK JSON-RPC 服务器 |
| **acp** | `dsh --profile acp` | 自动化 ACP 服务器 |
| **sdk-minimal** | `dsh --profile sdk-minimal` | 最小 SDK 树（不应用 dsh-base） |

还有四种 Agent 预设（在 Profile 内选择）：

| 预设 | 工具集 |
|---|---|
| **Standard** | 完整工具集 |
| **Code** | Standard + Code Mode SDK |
| **Minimal** | 仅 bash + 文件编辑器（基准测试用） |
| **Creator** | Standard + 运行时检查 + 插件实验 |

### 差异核心

| 维度 | Pi | dsh |
|---|---|---|
| **主界面** | 终端 TUI | Web UI |
| **终端 UI** | 一等公民（pi-tui 包） | 社区插件（dsh-cc-tui 等） |
| **会话分支** | `/tree` 原地分支 | 从日志 fork |
| **消息队列** | Enter=steering, Alt+Enter=follow-up | 收件箱模型 |
| **Compaction** | 内置（自动+手动，可扩展） | 靠插件实现 |
| **子 Agent** | 不内置，用扩展或 tmux | 插件支持 |

---

## 6. 记忆插件对比：Mem0 在两个平台的实现

这是最有说服力的对比——同一个 Mem0 团队，给两个平台各做了一个插件，设计完全不同。

### Mem0 官方 Pi 插件（`@mem0/pi-agent-plugin`）

| 维度 | 值 |
|---|---|
| **安装** | `pi install npm:@mem0/pi-agent-plugin` |
| **工具数** | 1 个（`mem0_memory`——搜索+存储合一） |
| **命令数** | 8 个斜杠命令 |
| **技能数** | 8 个 SKILL.md |
| **自动捕获** | ✅ 自动从对话中提取记忆（user+assistant） |
| **Dream 整合** | ✅ 合并重复、解决矛盾、清理过期 |
| **作用域** | project / session / global |
| **记忆分类** | 10 个自动分类 |
| **后端** | Mem0 Cloud Platform |
| **配置** | `~/.pi/agent/mem0-config.json` |
| **代码量** | ~10 个源文件 + 8 个技能 |

### Mem0 官方 dsh 插件（`@mem0/deepseek-plugin`）

| 维度 | 值 |
|---|---|
| **安装** | `dsh plugin add` + `cordis.example.yml` |
| **工具数** | 2 个（`search_memory` + `add_memory`） |
| **命令数** | 0（dsh 无斜杠命令概念） |
| **技能数** | 0 |
| **自动捕获** | ❌ 计划中，等 dsh 事件 API 稳定 |
| **Dream 整合** | ❌ |
| **作用域** | userId / agentId / runId（per-call 覆盖） |
| **记忆分类** | 无（依赖 Mem0 后端分类） |
| **后端** | Mem0 Cloud Platform |
| **配置** | `cordis.patch.yml` 中的 config 字段 |
| **代码量** | 6 个源文件，~500 行 |

### 差异分析：为什么同一个团队做的东西差这么多

| 差异点 | Pi 版 | dsh 版 | 原因 |
|---|---|---|---|
| **自动化程度** | 全自动 | 纯手动 | Pi 有成熟的事件钩子（`pi.on("tool_call")`）；dsh 事件 API 不稳定 |
| **功能丰富度** | 8 命令+8 技能 | 2 工具 | Pi 的扩展模型更成熟；dsh 是 Developer Preview |
| **Dream 整合** | 有 | 无 | Pi 的事件钩子允许后台异步整合；dsh 的异步能力还在建 |
| **记忆分类** | 10 类自动 | 无 | Pi 版在客户端做分类；dsh 版依赖后端 |
| **UI 集成** | 斜杠命令+技能 | 仅工具卡 | Pi 有 TUI 斜杠命令系统；dsh 没有 |

**核心洞察**：同一个团队（Mem0 官方），面对不同平台的成熟度，做出了完全不同的设计选择。Pi 平台更成熟 → 插件做全自动；dsh 平台还在 Developer Preview → 插件做最小可用，等 API 稳定再加。

---

## 7. 能力矩阵

| 能力 | Pi | dsh |
|---|---|---|
| **内置工具** | read, write, edit, bash, grep, find, ls | 通过插件（dsh-base） |
| **Provider 数量** | 30+（含 DeepSeek、OpenAI、Claude、Google 等） | 任意（通过插件适配） |
| **终端 UI** | ✅ 一等公民（pi-tui） | ❌ 社区插件 |
| **Web UI** | ❌ | ✅ 一等公民（dsh-web-app） |
| **会话分支** | ✅ 原地树结构（`/tree`） | ✅ 从日志 fork |
| **会话导出** | HTML / JSONL / GitHub Gist | 通过插件 |
| **Compaction** | ✅ 内置（自动+手动，可扩展） | 通过插件 |
| **子 Agent** | ❌ 故意不做 | ✅ 插件支持 |
| **MCP** | ❌ 故意不做（用扩展加） | ✅ 内置支持 |
| **权限系统** | ❌ 故意不做（用容器） | ✅ 插件（dsh-base） |
| **沙箱** | ❌ 故意不做 | ✅ 插件（bwrap/Landlock/Seatbelt） |
| **定时任务** | ❌ | ✅ 插件 |
| **多平台网关** | ❌ | ✅ webhook 插件 |
| **Agent 循环可替换** | ❌ | ✅ 本身是插件 |
| **模型可热切换** | ✅ `/model` 或 Ctrl+L | ✅ 替换配置行 |
| **SDK** | ✅ Node.js SDK | ✅ TypeScript + Python SDK |
| **RPC 模式** | ✅ stdin/stdout JSONL | ✅ JSON-RPC |
| **离线模式** | ✅ `--offline` | ❌ |
| **llama.cpp 本地模型** | ✅ `/llama` 命令 | 通过插件 |
| **Session 共享** | ✅ GitHub Gist | 通过导出 |
| **Skill 标准** | ✅ Agent Skills 标准（agentskills.io） | ✅ SKILL.md |
| **供应链安全** | ✅ 依赖锁定+shrinkwrap+审计 | ✅ pnpm 锁定 |
| **Stars** | ~15k（pi-mono） | ~214k |

---

## 8. 总结：应该选哪个

### 选 Pi 如果你：

- **工作在终端**——Pi 的 TUI 是一等公民，体验远超 dsh 的 Web UI
- **喜欢极简哲学**——不想要子 Agent、plan mode、权限弹窗等"多余"功能
- **要自己定制一切**——TypeScript Extensions 能力极强，可以替换内置工具
- **用 DeepSeek 模型**——Pi 原生支持 DeepSeek V4，DeepSeek 官方文档有集成教程
- **做记忆模块**——Pi 的事件钩子更成熟，Mem0 的 Pi 插件已经实现全自动
- **在本地跑**——不需要 Web 服务器，终端直接跑

### 选 dsh 如果你：

- **要 Web UI**——dsh 的 Web 工作台是核心体验
- **要替换 Agent 循环**——这是 dsh 独有的能力
- **要沙箱/权限系统**——dsh 内置（通过 dsh-base），Pi 需要容器
- **要多平台网关**——dsh 支持 webhook、Slack/Telegram 等
- **要定时任务**——dsh 有 Scheduling 插件
- **要子 Agent 和工作流**——dsh 内置支持
- **要 MCP**——dsh 内置支持
- **做企业部署**——dsh 有 Python SDK、ACP 自动化服务器
- **追求"无特权核心"理念**——dsh 的 Cordis 架构是学术级的

### 对你做记忆模块的启示

| 如果你在 Pi 上做 | 如果你在 dsh 上做 |
|---|---|
| 事件钩子成熟，可以做全自动记忆 | 事件 API 不稳定，先做手动工具，等稳定再加自动化 |
| 用 `pi.on("tool_call")` 捕获对话 | 用 `agent/step` 事件捕获，但注意平台时序约束 |
| 可以做斜杠命令 | 没有斜杠命令系统 |
| 可以做 SKILL.md 引导模型 | 可以在 system prompt section 引导 |
| `pi install` 一键安装 | `dsh plugin add` + cordis.patch.yml |
| 参考 `@mem0/pi-agent-plugin` 的全自动实现 | 参考 `@mem0/deepseek-plugin` 的最小实现 + `runfali/dsh-mem0-plugins` 的全自动实现 |

---

## 附录：关键参考

| 资源 | 链接 |
|---|---|
| Pi 仓库 | `github.com/badlogic/pi-mono` |
| Pi 官网 | `pi.dev` |
| Pi 文档 | `pi.dev/docs/latest` |
| DeepSeek API Pi 集成 | `api-docs.deepseek.com/quick_start/agent_integrations/pi_mono` |
| dsh 仓库 | `github.com/deepseek-ai/deepseek-harness` |
| dsh 官网 | `deepseek.com/harness` |
| dsh 架构文档 | `github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md` |
| Mem0 Pi 插件 | `github.com/mem0ai/mem0/tree/main/integrations/pi-agent-plugin` |
| Mem0 dsh 插件 | `github.com/mem0ai/mem0/tree/main/integrations/deepseek-plugin` |
| Cordis 论文 | `arxiv.org/abs/2608.25512` |

---

> **文档版本**：v1.0
> **核心结论**：Pi 和 dsh 代表了两种正交的 Agent 框架设计哲学——Pi 是"极简核心 + 激进扩展"（故意不做的功能让用户自己加），dsh 是"无特权核心 + 一切皆插件"（做的所有功能你都可以换掉）。同一个 Mem0 团队在两个平台上的插件实现差异（Pi 版全自动 vs dsh 版纯手动），直接反映了两个平台成熟度的差距。
