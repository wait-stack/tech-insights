# DeepSeek Harness 记忆模块插件开发文档

> **2026-09-07 源码复核补充**：本文部分接口示例与版本描述需校正，不能直接作为可编译模板。请优先阅读[场景化记忆接入洞察与实施方案](./DeepSeek-Harness场景化记忆接入洞察与实施方案_2026-09.md)，其中第 3、4、12 节提供当前源码契约、Mem0 原生 DSH 插件状态及具体勘误。

> **文档定位**：为需要在 DeepSeek Harness（dsh）中开发记忆模块插件的开发者提供完整的背景知识、技术对接指南和生态参考。
>
> **撰写日期**：2026年9月7日  
> **信息来源**：DeepSeek 官方文档、GitHub 仓库、社区分析文章、Mem0 官方文档与论文

---

## 目录

1. [DeepSeek Harness 是什么](#1-deepseek-harness-是什么)
2. [DeepSeek Harness 怎么运作的](#2-deepseek-harness-怎么运作的)
3. [主流记忆系统架构分析](#3-主流记忆系统架构分析以mem0为代表)
4. [DeepSeek Harness 记忆插件开发对接指南](#4-deepseek-harness-记忆插件开发对接指南)
5. [现有 dsh 记忆插件生态盘点](#5-现有-dsh-记忆插件生态盘点)
6. [记忆插件设计建议与路线图](#6-记忆插件设计建议与路线图)

---

## 1. DeepSeek Harness 是什么

### 1.1 一句话定义

> **DeepSeek Harness（dsh）** 是 DeepSeek AI 于 2026年8月13日开源的 Agent Harness（智能体运行框架），核心理念是 **"Everything is a Plugin"**（一切皆插件）——模型适配器、工具注册表、会话日志、甚至 Agent 循环本身都是可替换的插件，无需修改 dsh 源码即可扩展或替换任何能力。

### 1.2 关键事实

| 属性 | 值 |
|---|---|
| **开发者** | DeepSeek AI（deepseek-ai GitHub 组织） |
| **开源日期** | 2026年8月13日 |
| **许可证** | MIT |
| **语言** | TypeScript，运行于 Node.js |
| **当前版本** | v0.1.0-rc.5（Developer Preview） |
| **GitHub Stars** | 198k+（截至2026年9月） |
| **仓库地址** | `github.com/deepseek-ai/deepseek-harness` |
| **底层框架** | Cordis（插件内核） |
| **安装方式** | `npx @deepseek-ai/dsh web` |
| **Node 要求** | ^22.19 或 >=24 |

### 1.3 它不是什么

dsh **不是**：
- ❌ 一个聊天机器人外壳
- ❌ IDE 代码补全插件
- ❌ DeepSeek 模型本身（它调用模型，不训练/托管模型）
- ❌ Python 的 `deepseek-harness` 包（那是第三方独立开发者的 API 客户端库，不是 dsh 本体）

dsh **是**：
- ✅ 一个独立的进程，有自己的配置（`~/.dsh/`）、会话（`~/.dsh/sessions/`）、权限和沙箱系统
- ✅ 可以读文件、跑命令、搜索、调用工具，支持多模型路由和多 Agent 协作
- ✅ 几乎所有能力（工具、服务、事件、UI 面板）都通过插件扩展

### 1.4 三层架构模型

dsh 的设计可以拆解为三层：

```
┌─────────────────────────────────────────────────────┐
│                    用户 / 任务                        │
│            (Web UI / CLI / Python SDK)               │
├─────────────────────────────────────────────────────┤
│              DeepSeek Harness (dsh)                  │
│  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐ │
│  │ 工作空间   │ │ 工具注册表 │ │ 沙箱策略 │ │ 会话日志  │ │
│  │ Workspace │ │ Tool Reg  │ │ Sandbox │ │ Session  │ │
│  └──────────┘ └──────────┘ └────────┘ └──────────┘ │
│  ┌──────────────────────────────────┐               │
│  │       Agent Loop（Agent 循环）     │               │
│  │  感知 → 推理 → 行动 → 学习 闭环    │               │
│  └──────────────────────────────────┘               │
├─────────────────────────────────────────────────────┤
│                 Cordis 插件内核                       │
│  加载、卸载、依赖追踪 —— 注册即生效，卸载即撤销       │
├─────────────────────────────────────────────────────┤
│               模型层 (Model Layer)                   │
│  DeepSeek V4-Pro / V4-Flash / GPT / Claude / ...     │
└─────────────────────────────────────────────────────┘
```

**核心设计原则**：
1. **无特权核心**：没有不可替换的核心代码，"新行为应该放在插件里，而不是这里"
2. **每次运行可追溯**：所有进入模型请求的内容都写入只追加的会话日志

---

## 2. DeepSeek Harness 怎么运作的

### 2.1 Cordis 插件内核

Cordis 是 dsh 底层的插件框架，围绕以下原语构建：

| 原语 | 作用 |
|---|---|
| **Plugin**（插件） | 一个导出 `apply(ctx)` 函数的模块，通过 `ctx` 注册服务、事件和效果 |
| **Service**（服务） | 插件提供的可调用能力（如搜索、存储），其他插件通过 `ctx` 获取 |
| **Event**（事件） | 插件间的通信机制，分为三大域：`session/*`、`agent/*`、`capability/*` |
| **Effect**（效果） | 可逆的副作用注册，插件卸载时自动撤销（如 `setInterval` 会被 `clearInterval`） |

### 2.2 插件生命周期

```
启动 dsh
  │
  ▼
读取 Profile 配置 (~/.dsh/profiles/<profile>/)
  │  - bundle 列表（有序）
  │  - cordis.patch.yml（用户自定义补丁）
  │  - 已安装的 npm 包
  ▼
按序应用 Patch 层
  │  1. 各 bundle 的 cordis.patch.yml（按 profile 列出的顺序）
  │  2. Profile 自身的 cordis.patch.yml
  │  3. Home 级别的 cordis.patch.yml
  │  4. 命令行 --patch 叠加
  │  （后层覆盖前层，按行替换，不深度合并）
  ▼
组装插件树 → 挂载所有插件
  │  - 模型适配器插件
  │  - 工具注册表插件
  │  - 会话日志插件
  │  - Agent 循环插件
  │  - 记忆/搜索/UI 等第三方插件
  ▼
运行 Agent 循环
  │  感知（用户输入/外部事件/时间触发）
  │  → 推理（意图理解/任务分解/策略选择）
  │  → 行动（工具调用/文件操作/命令执行）
  │  → 学习（会话日志记录/记忆提取）
  │  ↺ 循环
  ▼
会话结束 → 写入会话日志 → 提取记忆（如果插件支持）
```

### 2.3 三大事件域

插件通过订阅事件来融入 Agent 的工作流：

| 事件域 | 触发时机 | 典型用途 |
|---|---|---|
| **`session/*`** | 会话级别的持久事实 | 记忆写入、会话摘要、日志追加 |
| **`agent/*`** | Agent 工作中的实时事件 | 拦截/观察进行中的工作（inbox、step、status、request、validation、continuation） |
| **`capability/*`** | 能力缝接点 | 为文件系统(`fs/*`)、工具(`tools/*`)、遥测(`telemetry/*`)附加策略和适配器 |

### 2.4 会话日志：唯一真相源

**核心不变量**：任何进入模型请求的内容，都必须能从会话日志重建。

三个机制保证这一点：
1. **结构性保证**：模型请求只能从日志推导，没有第二条代码路径
2. **写入前校验**：每个事件在被追加到日志前都经过验证
3. **运行时断言**：断言插件逐字节比对外发请求与从日志重建的请求，不匹配则中止

> **对记忆插件的含义**：记忆注入到模型上下文的操作也必须通过事件系统完成，确保可追溯、可回放。

### 2.5 Profile 与 Bundle

**Profile**（配置文件）是 `~/.dsh/profiles/` 下的一个目录，描述一个可启动的组装体：
- 哪些 bundle 叠加
- 安装了哪些插件
- 你自己的 `cordis.patch.yml`

dsh 内置 `web`（Web UI）和 `headless`（无头运行）两个 Profile 模板。

**Bundle**（捆绑包）是声明了 `dsh.bundle` 清单的普通 npm 包：
```json
// package.json
{
  "name": "my-memory-plugin",
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    }
  }
}
```

### 2.6 启动与使用

```bash
# 零安装启动 Web UI（默认 127.0.0.1:3080）
npx @deepseek-ai/dsh web

# 已安装后
dsh web                                    # Web 工作台
dsh --profile headless "run the tests"     # 无头一次性任务

# 插件管理
dsh plugin --profile web add <plugin-name>   # 安装插件
dsh plugin --profile web list                 # 列出已安装
dsh plugin --profile web remove <plugin-name> # 移除插件

# 配置检查
dsh --profile web --dump-config              # 导出完整插件树配置
```

---

## 3. 主流记忆系统架构分析（以 Mem0 为代表）

在为 dsh 开发记忆插件之前，理解现有记忆系统的架构至关重要。以下是业界最具代表性的记忆系统分析。

### 3.1 Mem0

#### 3.1.1 核心定位

Mem0 是 AI Agent 的记忆层（Memory Layer），位于应用和模型之间。你向 `add` 发送对话轮次，然后在下一次模型请求前调用 `search` 获取相关上下文。

#### 3.1.2 两阶段架构

Mem0 的处理分为 **提取（Extraction）** 和 **检索（Retrieval）** 两个阶段：

```
┌──────────────────────────────────────────────────────┐
│                   写入路径 (add)                       │
│                                                      │
│  对话轮次 ──→ 1.上下文查找（查已有记忆，避免重复）       │
│            ──→ 2.事实提取（LLM 从对话中提取持久事实）   │
│            ──→ 3.去重 + 向量化（Hash去重 → Embedding） │
│            ──→ 4.实体提取（人/地/组织/概念 → 实体图）    │
│            ──→ 5.时间元数据（事件时间/状态/精度/类型）   │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                   读取路径 (search)                     │
│                                                      │
│  查询 ──→ 并行评分：                                   │
│           ├ 语义搜索（向量相似度）                      │
│           ├ BM25 关键词匹配（含词形归一化）              │
│           ├ 实体匹配（查询实体 → 关联记忆加成）          │
│           └ 时间推理（时间意图 → 时间元数据匹配）       │
│         ──→ 分数融合 → Top-K 结果                      │
└──────────────────────────────────────────────────────┘
```

#### 3.1.3 v2.x 关键架构变更（2026年4月）

| 维度 | 旧版协调引擎 | v2.x ADD-only 引擎 |
|---|---|---|
| **写入路径** | 读取已有记忆 → 多次 LLM 调用决定 ADD/UPDATE/DELETE | 单次 LLM 调用，仅 ADD——记忆累积，不覆盖 |
| **关系召回** | 外接图数据库（如 Neo4j） | 内置实体链接——实体提取+向量化+跨记忆关联 |
| **检索信号** | 语义向量搜索 | 语义 + BM25 + 实体匹配 三路并行融合 + 时间感知排序 |
| **重排序** | 无内置 | 可选 Rerank（Cohere/ZeroEntropy/Cross-encoder/LLM） |
| **失败模式** | 错误的协调可能静默删除正确记忆 | 无删除操作，矛盾事实共存直到手动清理 |

#### 3.1.4 存储架构

| 存储 | 持有内容 | 用途 |
|---|---|---|
| **SQL 数据库** | 事实和元数据 | 每条记忆的真相源 |
| **向量数据库** | 嵌入向量 | 语义相似度搜索 |
| **实体存储** | 从记忆文本提取的实体 | 实体匹配加成（Platform 上还支撑图记忆） |

#### 3.1.5 多租户与作用域

Mem0 通过四个维度隔离记忆，防止数据混用：

| 维度 | 字段 | 用途 | 示例 |
|---|---|---|---|
| 用户 | `user_id` | 持久人设或账户 | `"customer_6412"` |
| Agent | `agent_id` | 不同的 Agent 或工具 | `"meal_planner"` |
| 应用 | `app_id` | 产品面或部署 | `"ios_retail_app"` |
| 会话 | `run_id` | 短时流程或线程 | `"ticket-9241"` |

#### 3.1.6 基准测试表现

| 基准测试 | 旧版分数 | 新版分数 | 检索 Token 预算 |
|---|---|---|---|
| **LoCoMo** | 71.4 | **92.5** | 7.0K |
| **LongMemEval** | 67.8 | **94.4** | 6.8K |
| **BEAM (1M)** | — | **64.1** | 6.7K |
| **BEAM (10M)** | — | **48.6** | 6.9K |

> **关键洞察**：Mem0 的核心优势来自 ADD-only 架构（保留全部状态变更历史）+ 多信号融合检索 + 时间推理。在时间推理类别提升 +29.3 分，多跳推理提升 +25.2 分。

#### 3.1.7 三层记忆模型

| 层级 | 生命周期 | 管理方 | 用途 |
|---|---|---|---|
| **工作记忆** | 单轮 | 应用程序 | 轮内消息、工具调用、思维链 |
| **会话记忆** | 分钟到小时 | Mem0（via `run_id`） | 当前任务上下文、多步流程 |
| **用户记忆** | 周到永久 | Mem0（via `user_id`） | 个人偏好、账户状态、长期知识 |

### 3.2 其他记忆系统简析

| 系统 | 核心机制 | 优势 | 局限 |
|---|---|---|---|
| **Hindsight** (Vectorize) | 知识图谱 + 反思综合 | 跨工具/跨 Agent 可移植（同一 bank 服务 Claude Code/Cursor/dsh 等） | 依赖外部服务 |
| **Honcho** | 辩证推理 + 跨会话用户建模 | 多 Agent 系统上下文感知强 | 云端付费 |
| **OpenViking** | 文件系统式知识层级 + 分层加载 | 自托管、L0→L1→L2 分层加载省 Token | 需独立部署 server |
| **RetainDB** | Delta 压缩 | 10 个工具、API 简单 | $20/月 |
| **Holographic** | HRR 代数 + 信任评分 | 本地运行、无依赖 | 2 个工具，功能有限 |
| **Mnemon (dsh-mnemon)** | 三层记忆（Runtime/Documents/Memory Spaces） | 原生为 dsh 设计 | 社区项目，仍在早期 |
| **Memoria (dsh-memoria)** | 向量+图记忆层 + 命名空间隔离 | 自动写入（轮次结束 observe → 正反馈 remember）、热重载设置 | 社区项目 |
| **dsh-memory-evolve** | 跨会话长期记忆 + 后台自进化 | 5 轨道记忆/git 分支感知/技能进化 | 社区项目 |

---

## 4. DeepSeek Harness 记忆插件开发对接指南

### 4.1 Cordis 插件基础

一个 dsh 插件就是一个导出 `apply(ctx)` 函数的 TypeScript 模块：

```typescript
// src/index.ts
import type { Context } from '@deepseek-ai/dsh'

export function apply(ctx: Context) {
  // 1. 注册服务（其他插件可以调用）
  ctx.services.register('my-memory', {
    search: (query: string) => { /* ... */ },
    add: (messages: Message[]) => { /* ... */ },
  })

  // 2. 订阅事件
  ctx.events.on('agent/step', (event) => {
    // 在每个 Agent 步骤后写入记忆
  })

  ctx.events.on('session/end', (event) => {
    // 会话结束时提取记忆
  })

  // 3. 注册可逆效果（插件卸载时自动清理）
  ctx.effect(() => {
    const timer = setInterval(() => {
      // 定期维护任务
    }, 60000)
    return () => clearInterval(timer)  // 卸载时执行
  })
}
```

### 4.2 插件清单声明

在 `package.json` 中声明 `dsh.bundle`：

```json
{
  "name": "dsh-mymemory",
  "version": "1.0.0",
  "main": "dist/index.js",
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    }
  },
  "dependencies": {
    "@deepseek-ai/dsh": "^0.1.0"
  }
}
```

### 4.3 Cordis Patch 文件

`cordis.patch.yml` 声明插件在配置树中插入的行：

```yaml
# cordis.patch.yml
# 在配置树中注册此插件
- add:
  - id: my-memory-provider
    name: 'dsh-mymemory'
    config:
      # 默认配置
      recallMode: hybrid        # auto-inject + tools
      writeFrequency: async      # 后台异步写入
      maxRecallResults: 10
      embeddingModel: 'text-embedding-3-small'
      llmProvider: 'deepseek-official'
      llmModel: 'deepseek-v4-flash'
```

### 4.4 记忆插件的核心对接点

基于对 dsh 事件系统和现有记忆插件的分析，记忆插件需要对接以下关键环节：

```
┌─────────────────────────────────────────────────────────────┐
│                  记忆插件对接架构图                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    session/start    ┌──────────────────┐  │
│  │  会话启动     │ ──────────────────→ │ 初始化记忆上下文  │  │
│  └─────────────┘                      │ 注入历史记忆到     │  │
│                                       │ 系统提示          │  │
│  ┌─────────────┐    agent/step       └──────────────────┘  │
│  │  Agent 步骤  │ ──────────────────→ ┌──────────────────┐  │
│  │  (每轮对话)  │                     │ 1. 检索相关记忆    │  │
│  │             │ ←────────────────── │ 2. 注入到上下文    │  │
│  └─────────────┘                     └──────────────────┘  │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────┐    agent/response       ┌──────────────┐  │
│  │  响应生成后   │ ──────────────────→   │ 提取记忆       │  │
│  │             │                        │ - 事实提取     │  │
│  │             │                        │ - 去重         │  │
│  └─────────────┘                        │ - 向量化       │  │
│                                         │ - 实体链接     │  │
│  ┌─────────────┐    session/end    └──────────────┘     │
│  │  会话结束     │ ──────────────────→ ┌──────────────┐    │
│  └─────────────┘                     │ 批量记忆提取    │    │
│                                       │ 会话摘要生成    │    │
│                                       └──────────────┘    │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              记忆存储后端                              │  │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌───────────┐  │  │
│  │  │ SQL DB │  │ Vector │  │ Entity │  │ Graph DB  │  │  │
│  │  │ 事实源  │  │ 向量库  │  │ 实体库  │  │ 关系图(可选)│  │  │
│  │  └────────┘  └────────┘  └────────┘  └───────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

#### 对接点详解

| 对接点 | 事件 | 插件动作 | 关键设计 |
|---|---|---|---|
| **会话启动** | `session/start` | 初始化记忆上下文，预取相关记忆注入系统提示 | 非阻塞预取，避免延迟 |
| **Agent 步骤前** | `agent/step` (before) | 根据当前查询检索记忆，注入到模型上下文 | 这是**最核心的对接点**——决定记忆召回质量 |
| **Agent 步骤后** | `agent/step` (after) | 异步提取本轮对话中的事实，写入记忆存储 | ADD-only 模式，不覆盖旧记忆 |
| **会话结束** | `session/end` | 批量提取、生成会话摘要、清理临时记忆 | 异步执行不阻塞用户 |
| **工具注册** | 插件挂载时 | 注册记忆搜索/管理工具供 Agent 主动调用 | 如 `memory_search`、`memory_add` |

### 4.5 记忆插件骨架代码

以下是参考 Mem0 架构设计的 dsh 记忆插件骨架：

```typescript
// src/index.ts
import type { Context, Service, Event } from '@deepseek-ai/dsh'

interface MemoryRecord {
  id: string
  content: string
  embedding: number[]
  entities: Entity[]
  temporalMeta?: TemporalMeta
  userId?: string
  sessionId?: string
  createdAt: Date
}

interface PluginConfig {
  recallMode: 'hybrid' | 'tools' | 'auto'
  writeFrequency: 'async' | 'turn' | 'session'
  maxRecallResults: number
  embeddingModel: string
  llmModel: string
  vectorStoreUrl?: string
  sqlStorePath?: string
}

export function apply(ctx: Context) {
  const config = ctx.config as PluginConfig

  // ─── 1. 初始化存储后端 ───
  const vectorStore = createVectorStore(config.vectorStoreUrl)
  const sqlStore = createSqlStore(config.sqlStorePath || '~/.dsh/memories.db')
  const entityStore = createEntityStore()

  // ─── 2. 注册记忆服务 ───
  const memoryService = {
    async search(query: string, filters: MemoryFilters): Promise<MemoryRecord[]> {
      // 并行评分
      const [semantic, keyword, entity] = await Promise.all([
        vectorStore.search(query, filters, config.maxRecallResults * 3),
        bm25Search(query, filters),
        entityStore.match(query, filters),
      ])
      // 分数融合
      return fuseScores([semantic, keyword, entity], config.maxRecallResults)
    },

    async add(messages: Message[], filters: MemoryFilters): Promise<void> {
      // 单次 LLM 调用提取事实（ADD-only）
      const facts = await extractFacts(messages, config.llmModel)
      // Hash 去重
      const newFacts = await deduplicate(facts, sqlStore)
      // 向量化 + 实体提取
      for (const fact of newFacts) {
        const embedding = await embed(fact.content, config.embeddingModel)
        const entities = await extractEntities(fact.content, config.llmModel)
        const temporalMeta = await extractTemporal(fact, messages)
        const record: MemoryRecord = {
          id: hashId(fact.content),
          content: fact.content,
          embedding,
          entities,
          temporalMeta,
          userId: filters.userId,
          sessionId: filters.sessionId,
          createdAt: new Date(),
        }
        await sqlStore.insert(record)
        await vectorStore.insert(record)
        await entityStore.insert(record.entities, record.id)
      }
    },

    async recall(query: string, filters: MemoryFilters): Promise<string> {
      const memories = await this.search(query, filters)
      return formatMemories(memories)
    },
  }

  ctx.services.register('memory', memoryService)

  // ─── 3. 注册 Agent 可调用的工具 ───
  ctx.tools.register({
    id: 'dsh-memory-search',
    name: 'memory_search',
    description: 'Search long-term memory for relevant context',
    handler: async (args: { query: string }) => {
      const results = await memoryService.search(args.query, getCurrentFilters(ctx))
      return results.map(r => r.content).join('\n')
    },
  })

  ctx.tools.register({
    id: 'dsh-memory-add',
    name: 'memory_add',
    description: 'Store a fact in long-term memory',
    handler: async (args: { content: string }) => {
      await memoryService.add(
        [{ role: 'user', content: args.content }],
        getCurrentFilters(ctx)
      )
      return 'Memory stored.'
    },
  })

  // ─── 4. 事件订阅：自动记忆注入与提取 ───
  ctx.events.on('agent/step:before', async (event) => {
    if (config.recallMode === 'tools') return  // 仅工具模式不自动注入
    // 非阻塞预取相关记忆
    const query = event.userInput
    const filters = { userId: event.userId, sessionId: event.sessionId }
    const memoryContext = await memoryService.recall(query, filters)
    if (memoryContext) {
      event.injectSystemContext(memoryContext)
    }
  })

  ctx.events.on('agent/step:after', async (event) => {
    if (config.writeFrequency === 'session') return  // 会话结束才写入
    if (config.writeFrequency === 'async') {
      // 异步写入，不阻塞
      setImmediate(() => {
        memoryService.add(event.messages, {
          userId: event.userId,
          sessionId: event.sessionId,
        })
      })
    } else {
      // 同步写入
      await memoryService.add(event.messages, {
        userId: event.userId,
        sessionId: event.sessionId,
      })
    }
  })

  ctx.events.on('session/end', async (event) => {
    if (config.writeFrequency === 'session') {
      // 批量写入整轮会话
      await memoryService.add(event.allMessages, {
        userId: event.userId,
        sessionId: event.sessionId,
      })
    }
    // 生成会话摘要（可选）
    await generateSessionSummary(event, config.llmModel)
  })

  // ─── 5. 可逆效果 ───
  ctx.effect(() => {
    // 定期维护：清理过期实体链接、重建索引等
    const maintenance = setInterval(() => {
      vectorStore.compact()
      entityStore.prune()
    }, 3600000)  // 每小时
    return () => clearInterval(maintenance)
  })
}

// ─── 辅助函数 ───
function getCurrentFilters(ctx: Context): MemoryFilters {
  const session = ctx.services.get('session')
  return {
    userId: session?.userId || 'default',
    sessionId: session?.id,
  }
}

function formatMemories(memories: MemoryRecord[]): string {
  if (!memories.length) return ''
  return memories
    .map(m => `- ${m.content}`)
    .join('\n')
}

function fuseScores(results: MemoryRecord[][], topK: number): MemoryRecord[] {
  // 三个信号的分数融合为最终排名
  const scoreMap = new Map<string, { record: MemoryRecord; score: number }>()
  for (const resultSet of results) {
    for (let i = 0; i < resultSet.length; i++) {
      const r = resultSet[i]
      const rankScore = 1 / (i + 1)  // 倒数排名
      const existing = scoreMap.get(r.id)
      if (existing) {
        existing.score += rankScore
      } else {
        scoreMap.set(r.id, { record: r, score: rankScore })
      }
    }
  }
  return [...scoreMap.values()]
    .sort((a, b) => b.score - a.score)
    .slice(0, topK)
    .map(x => x.record)
}
```

### 4.6 安装与配置

```bash
# 安装你的记忆插件
dsh plugin --profile web add dsh-mymemory

# 或从 GitHub 安装
dsh plugin --profile web add github:yourname/dsh-mymemory

# 验证已加载
dsh --profile web --dump-config | grep mymemory
```

用户自定义配置（覆盖默认）：

```yaml
# ~/.dsh/profiles/web/cordis.patch.yml
- replace:
  - id: my-memory-provider
    name: 'dsh-mymemory'
    config:
      recallMode: hybrid
      writeFrequency: async
      maxRecallResults: 20
      embeddingModel: 'BAAI/bge-m3'
      llmModel: 'deepseek-v4-flash'
      vectorStoreUrl: 'http://localhost:6333'  # Qdrant
      sqlStorePath: '~/.dsh/memories.db'
```

> **注意**：Patch 替换整行配置值，不做深度合并。要改一个字段必须重述该行所有键。

### 4.7 关键设计决策清单

| 决策点 | 选项 | 推荐值 | 理由 |
|---|---|---|---|
| 写入策略 | ADD-only / UPDATE-DELETE 协调 | **ADD-only** | 避免协调循环静默删除正确记忆，保留状态变更历史 |
| 检索信号 | 语义 / 关键词 / 实体 / 时间 | **四路融合** | 不同查询类型依赖不同信号，融合优于单信号 |
| 写入时机 | 每轮同步 / 每轮异步 / 会话结束批量 | **异步** | 不阻塞 Agent 循环，用户体验好 |
| 注入模式 | 自动注入 / 工具调用 / 混合 | **混合** | 自动注入保底 + 工具让 Agent 主动深挖 |
| 向量库 | Qdrant / Chroma / pgvector / 本地 | **Qdrant** | 开源、高性能、dsh 生态已有集成 |
| 嵌入模型 | OpenAI text-embedding-3-small / BGE-M3 / 自托管 | **BGE-M3** | 开源、多语言、中文友好 |
| 事实提取 LLM | DeepSeek V4-Flash / GPT-5-mini / 本地模型 | **DeepSeek V4-Flash** | 与 dsh 生态一致、成本低 |

---

## 5. 现有 dsh 记忆插件生态盘点

截至2026年9月，dsh 的插件生态已有 700+ 仓库，其中记忆相关的主要插件：

### 5.1 主要记忆插件

| 插件 | 安装命令 | 特点 | 状态 |
|---|---|---|---|
| **dsh-memory** (by hyls9527) | `dsh plugin add @hyls9527/dsh-memory` | 从 Hermes Agent 移植的 MEMORY.md/USER.md 有界记忆系统 + 技能生命周期管理 | 活跃，168 测试 |
| **dsh-mnemon** | `dsh plugin add dsh-mnemon` | 三层记忆：Runtime Memory（轮次级偏好）、Documents（项目材料）、Memory Spaces（跨会话检索） | 活跃，社区推荐 |
| **dsh-memoria** | 见 GitHub | 向量+图记忆层，4 个工具（observe/remember/search/recall），命名空间隔离，自动写入 + 热重载 | 活跃 |
| **dsh-memory-evolve** | 见 GitHub | 跨会话长期记忆 + 后台自进化，5 轨道记忆 + git 分支感知 + 技能进化 | 活跃 |
| **dsh-simple-wiki-memory** | 见 GitHub | 极简 LLM-wiki 记忆：一个索引文档 + 按需读取的 markdown 文件，不烧 Token | 轻量 |
| **ReMe** | `dsh plugin add @agentscope-ai/reme` | 记忆 bundle：recall + capture + settings + skill guidance | 活跃 |
| **MemSearch** | `dsh plugin add @zilliz/memsearch-dsh` | 共享 Markdown 笔记捕获 + 步骤前上下文注入 + 候选技能审查 | 活跃 |
| **MemOS Local Memory** | `curl ... \| bash -s -- --agent dsh` | 本地 MemOS：分层召回 + 反思 + 策略归纳 + 技能结晶 | 早期 |
| **Hindsight for dsh** | `npx @vectorize-io/hindsight-coding-agents install dsh` | 知识图谱 + 反思综合，跨 Agent 可移植（同一 bank 服务 Claude Code/Cursor/dsh 等） | 成熟 |
| **dsh-memory-plugin (OpenViking)** | 见 GitHub | 连接 OpenViking 自进化上下文数据库，跨会话记忆 + 知识 RAG | 活跃 |

### 5.2 从其他平台移植的记忆方案

| 来源 | 对接方式 | 说明 |
|---|---|---|
| **Mem0** | 通过 Hindsight 或直接实现 | Mem0 的提取+检索架构可作为 dsh 插件的后端 |
| **OpenViking** | dsh-memory-plugin | 文件系统式知识层级 + L0→L1→L2 分层加载 |
| **Mnemosyne** | 社区讨论中 | 本地优先、BEAM 基准有验证、Fact 引擎 + SHA-256 ID |

### 5.3 生态趋势观察

1. **记忆插件是最活跃的品类之一**：dsh 发布不到一个月，已有 10+ 记忆相关插件
2. **两种设计哲学并存**：
   - **轻量有界记忆**（如 dsh-memory 的 MEMORY.md/USER.md）：简单、可控、Token 效率高
   - **全功能记忆系统**（如 dsh-mnemon、dsh-memoria）：多层、自动提取、语义检索
3. **跨平台可移植性**成为差异化优势（Hindsight 的同一 bank 服务多个 Agent 框架）
4. **ADD-only 架构**正在成为行业共识（Mem0 v2.x 率先采用，社区插件跟进）

---

## 6. 记忆插件设计建议与路线图

### 6.1 核心设计建议

基于对 Mem0 架构、dsh 事件系统和现有生态的分析，建议你的记忆插件采用以下设计：

#### 6.1.1 架构建议

```
推荐架构：分层记忆 + 多信号融合 + ADD-only 写入

写入：
  对话 → LLM 单次提取事实 → Hash 去重 → 向量化 + 实体链接 → 存储（仅 ADD）

检索：
  查询 → 并行 [语义 + BM25 + 实体] → 分数融合 → Top-K → 注入上下文

存储：
  SQLite（事实源） + Qdrant（向量） + 内置实体表（实体链接）
```

#### 6.1.2 差异化方向建议

| 方向 | 具体做法 | 对标 |
|---|---|---|
| **超长会话准确度** | 优化检索深度 + 时间推理 + 多跳推理 | 对标 Mem0 的 LongMemEval 94.4 分 |
| **Token 效率** | 分层加载（L0 摘要→L1 概览→L2 全文），按需展开 | 对标 OpenViking 的分层加载 |
| **跨 Agent 可移植** | 记忆存储独立于 dsh，可被其他框架召回 | 对标 Hindsight 的跨平台 bank |
| **中文优化** | 使用 BGE-M3 嵌入 + 中文实体识别优化 | 对标 GLM-5 的中文优势 |
| **数据集审计** | 集成 LongMemEval 基准测试，量化噪声影响 | 你的核心方法论 |

#### 6.1.3 开发路线图

```
Phase 1 (MVP, 2 周)
├── Cordis 插件骨架（apply + events + tools）
├── SQLite 事实存储 + Qdrant 向量存储
├── 单次 LLM 事实提取（ADD-only）
├── 语义检索（单信号）
├── agent/step:before 自动注入
└── memory_search / memory_add 工具

Phase 2 (增强检索, 2 周)
├── BM25 关键词匹配（含词形归一化）
├── 实体提取 + 实体匹配加成
├── 三路分数融合
├── 时间元数据提取 + 时间推理
└── 会话摘要生成

Phase 3 (优化与基准, 2 周)
├── LongMemEval 基准测试集成
├── 数据集噪声审计（GitHub Issues 深挖）
├── 检索深度调优（top_k, rerank）
├── 中文嵌入模型对比（BGE-M3 vs text-embedding-3-small）
└── 性能基准（延迟、Token 消耗）

Phase 4 (生态, 持续)
├── npm 发布
├── dsh 插件商店收录
├── 文档与示例
└── 社区反馈迭代
```

### 6.2 技术栈推荐

| 组件 | 推荐选择 | 理由 |
|---|---|---|
| 插件语言 | TypeScript | dsh 原生语言，Cordis SDK 一等支持 |
| 向量数据库 | Qdrant（本地 Docker） | 开源、高性能、Filter 支持好 |
| SQL 数据库 | SQLite（嵌入式） | 零配置、与 dsh 一致（会话日志也是 JSONL/SQLite） |
| 嵌入模型 | BAAI/bge-m3（自托管）或 text-embedding-3-small（API） | BGE-M3 中文友好、开源；OpenAI 简单 |
| 事实提取 LLM | DeepSeek V4-Flash | 与 dsh 生态一致、$0.14/M tokens、极低成本 |
| Rerank（可选） | BGE-Reranker-v2-m3 | 开源、中文友好、Cross-encoder |
| 基准测试 | LongMemEval + BEAM | 你已有 LongMemEval 调研基础 |

---

## 附录 A：关键术语对照表

| 术语 | 英文 | 含义 |
|---|---|---|
| 线束/框架 | Harness | 给模型提供工作空间、工具、权限和运行记忆的框架 |
| 插件 | Plugin | 导出 `apply(ctx)` 的模块，注册服务/事件/效果 |
| 捆绑包 | Bundle | 声明了 `dsh.bundle` 清单的 npm 包 |
| 配置文件 | Profile | `~/.dsh/profiles/` 下的可启动组装体 |
| 补丁 | Patch | YAML 配置行，后层覆盖前层 |
| 会话日志 | Session Log | 只追加的事件流，唯一真相源 |
| Cordis | Cordis | dsh 底层的插件内核框架 |
| 事件域 | Event Domain | session/*、agent/*、capability/* 三大类 |
| 效果 | Effect | 可逆副作用，卸载时自动撤销 |

## 附录 B：参考资源

| 资源 | 链接 |
|---|---|
| dsh 官方网站 | `deepseekharness.io` |
| dsh 官方文档 | `deepseekdocs.com/en/docs/learn/intro/what-is-dsh` |
| dsh GitHub 仓库 | `github.com/deepseek-ai/deepseek-harness` |
| dsh 架构文档 | `github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md` |
| 插件指南 | `dshplugins.co/en/dsh-plugins-guide` |
| 插件目录 | `dshpluginstore.com` |
| 插件生态盘点 | `github.com/0xsline/awesome-deepseek-harness` |
| Mem0 官方文档 | `docs.mem0.ai/core-concepts/how-it-works` |
| Mem0 论文 | `arxiv.org/abs/2504.19413` |
| Mem0 GitHub | `github.com/mem0ai/mem0` |
| Hindsight dsh 集成 | `hindsight.vectorize.io/blog/2026/08/14/deepseek-harness-persistent-memory` |

---

> **文档版本**：v1.0  
> **下次更新**：当 dsh 发布稳定版或 API 有破坏性变更时
