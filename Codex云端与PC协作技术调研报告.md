# Codex 云端与 PC 协作技术调研报告

> **报告日期**：2026-09-11
> **调研范围**：OpenAI Codex 云端（Codex cloud）与本地端（CLI / 桌面 App / IDE）协作机制，覆盖竞品对比与学术前沿
> **信息来源**：官方文档（learn.chatgpt.com / developers.openai.com）、GitHub 源码（openai/codex）、产品官方博客、arXiv 论文线索
> **分析维度**：产品视角（定位/差异化/演进时间线）+ 技术视角（架构/协议/机制/源码级细节）
> **说明**：本报告为原创调研综合，核心事实均出自上述公开来源；报告末尾附来源索引

---

## 一、核心发现概述

2026 年的编码 Agent 竞争焦点已从"谁的模型更强"转向"**谁能让任务在云和端之间无损流转**"。OpenAI Codex 以"**一套状态、多种表面**"为核心策略，把同一份会话状态（Thread/Turn/Item）通过 app-server 协议暴露给 CLI、桌面 App、IDE、云端和手机 Remote 五类客户端，形成目前最完整的云-端协作矩阵。

| 关键机制 | 实现方式 | 状态 |
|---|---|---|
| 云任务下发与回收 | `codex cloud exec/status/list/apply/diff` 五命令 | 实验性（CLI 内标注 EXPERIMENTAL） |
| 端侧执行隔离 | Git worktree（管理型 detached HEAD / 永久型） | 稳定 |
| Local↔Worktree 会话迁移 | Handoff（Codex 代管 Git 操作） | 稳定（2026-03-03 上线） |
| 手机遥控 PC | Codex Remote（QR 配对）+ `codex remote-control` 守护进程 | 稳定 |
| 跨机器远程会话 | `codex --remote ws://` 连接远端 app-server | 实验性 |
| 本地会话复刻到云端 | 导入 Claude Code / Cursor 的会话与配置 | 稳定（2026-08-11） |
| 环境复现 | 云容器 + setup script + 12h 缓存 / Environment Snapshot（竞品） | 稳定 |

**竞品同构性**：Claude Code 的 `--cloud`/`--teleport`/`remote-control`、Cursor 的 Cloud Agents（PR / Check out Locally / Apply locally）、Copilot 的 Local→Cloud handoff、Devin 的 `/handoff`，与 Codex 的五命令 + Remote 体系在能力上高度趋同——"**本地起任务 → 云端跑 → 本地收结果**"正在成为全行业标配的三段式工作流。

**对 dsh/ASL 的启示**：Codex 的 app-server 协议（Thread/Turn/Item + JSONL/WebSocket）与 Worktree Handoff 机制，是当前市面上对"会话状态跨表面迁移"最完整的工程化答案，与用户正在设计的 Agent State Layer（turn 复合检查点）方向高度互补，细节见第六章。

---

## 二、Codex 产品矩阵与三种环境模式

### 2.1 五大表面 + 两个辅助表面

OpenAI 官方文档以“表面（Surface）”为单位组织 Codex 文档，各表面共享同一套账户、配置体系（config.toml + requirements.toml）和技能/插件/MCP 生态：

| 表面 | 形态 | 定位 |
|---|---|---|
| **Codex CLI** | 终端 TUI，Rust 实现，开源（123k+ stars） | 交互式终端工作流 + 自动化 |
| **ChatGPT 桌面 App**（含 Codex） | macOS / Windows / Linux(preview) | 并行项目指挥中心 |
| **IDE 扩展** | VS Code / Cursor / Windsurf | 编辑器旁路协作 |
| **Codex cloud** | chatgpt.com/codex | 隔离云环境异步任务 |
| **SDK** | TS `@openai/codex-sdk` / Python `openai-codex` | 程序化控制本地 agent |
| Remote（辅助） | 手机 App → 连接的电脑 | 随时随地指挥本地 agent |
| app-server（辅助） | 协议层守护进程 | 所有本地表面的统一后端 |

### 2.2 三种环境模式（Environments）

桌面 App 新建 Codex 会话时，composer 下方可选运行位置——这是理解 Codex 云-端协作的入口：

```
┌────────────────────────────────────────────────────────┐
│  Local          Worktree           Cloud               │
│  当前项目目录     Git worktree 隔离    配置好的云环境      │
│  （前台）        （后台/并行）        （远程/异步）        │
└────────────────────────────────────────────────────────┘
```

- **Local**：直接在当前项目目录工作，交互最直接、反馈最快
- **Worktree**：在 `$CODEX_HOME/worktrees` 下创建 detached-HEAD worktree，不污染现有分支；默认保留最近 15 个管理型 worktree，删除前自动快照
- **Cloud**：容器环境（见下节），完全异步，从 Web/GitHub/Linear/Slack/CLI 均可发起

**产品设计洞察**：三种模式本质是同一个"任务"对象在不同执行位置的投影，而非三个产品。会话可以从 Worktree Handoff 回 Local，云任务结果可以 apply 回本地——**位置是属性，不是身份**。

### 2.3 Codex cloud 执行流水线

一次云任务的完整生命周期（官方文档 environments/cloud-environment）：

1. **容器创建**：Codex 创建容器并 checkout 选定的 branch/commit SHA
2. **环境装配**：运行 setup script（缓存容器恢复时额外跑 maintenance script）；常见包管理器（npm/yarn/pnpm/pip/pipenv/poetry）可自动安装依赖
3. **网络策略**：setup 阶段有网络；agent 阶段默认断网，可配置 limited / unrestricted
4. **Agent 循环**：终端命令循环，依据仓库内 AGENTS.md 发现 lint/test 命令
5. **产出回收**：展示 summary + diff，可 follow-up 或开 PR

关键工程细节：
- **默认镜像 `universal`**：预装常见语言与工具，openai/codex-universal 提供参考 Dockerfile，可本地拉取验证——**云环境本身开源可复现**
- **容器缓存 12 小时**：加速新任务与 follow-up，这直接解决"每次冷启动 9 分钟装依赖"的痛点（Cursor 3.4 的 70% 缓存提速是同一问题的不同解法）
- **Secrets 分阶段隔离**：额外加密存储，仅在 setup 阶段解密，agent 阶段开始前移除——比"全程注入环境变量"的朴素做法安全得多
- **Setup/Agent 双会话分离**：setup 在独立 Bash 会话跑，`export` 不穿透到 agent 阶段，要持久化得写 `~/.bashrc` 或环境配置

### 2.4 `codex cloud` CLI 命令族（源码级）

从 openai/codex 仓库 `codex-rs/cloud-tasks` crate 源码确认的完整命令面：

```bash
# 提交云任务（免 TUI，可脚本化）
codex cloud exec "修复 CI 失败" --env ENV_ID [--attempts 1-4] [--branch main]

# 任务管理
codex cloud list [--env ENV_ID] [--limit 20] [--cursor ...] [--json]
codex cloud status TASK_ID
codex cloud diff TASK_ID [--attempt N]

# 云 → 本地：把任务 diff 应用到本地工作树
codex cloud apply TASK_ID [--attempt N]
```

源码级发现（`codex-rs/cloud-tasks/src/lib.rs`）：

1. **后端端点**：`https://chatgpt.com/backend-api`，可用环境变量 `CODEX_CLOUD_TASKS_BASE_URL` 覆盖——意味着私有部署/代理场景可自定后端
2. **Best-of-N 尝试**：`--attempts` 1–4，每个 attempt 独立产出 diff，apply/diff 时按 attempt 号选择——用"多次采样 + 人工挑选"对冲单次生成质量方差
3. **Apply 预检机制**：`apply_task_preflight` 先行检查（工作树干净度等），`ApplyStatus` 三态（Success/Partial/Error），Partial 状态值得注意——它承认"云 diff 应用到本地可能只部分成功"这一现实
4. **认证复用**：与 CLI 同一 auth_manager（ChatGPT 账号 OAuth），云任务与本地任务共用身份
5. **TUI 浏览模式**：`codex cloud` 直接进入交互界面，可浏览环境列表、任务历史、提交新任务、应用结果

**与竞品的关键差异**：Codex cloud 的 apply 是拉 diff 打本地（`git apply` 语义），而 Claude Code `--teleport` 是拉**整段会话历史**（分支 + 对话记录）回终端。前者搬运"结果"，后者搬运"上下文"——这是两种不同的协作哲学，详见第四章对比。

---

## 三、云-端协作的四大机制详解

### 3.1 机制一：Worktree 隔离与 Handoff 迁移

**Worktree 是 Codex 端侧并行的基石**。git 的约束是"一个分支只能被一处 checkout"，Codex 用 detached-HEAD worktree 绕开：管理型 worktree 不占分支名，可任意并行。

三层 worktree 体系：
- **Codex 管理型**（默认）：一 chat 一 worktree，轻量可弃置，chat 归属固定（handoff 回来还是同一个 worktree）
- **永久型**：侧边栏创建，作为独立项目存在，多 chat 共享，不自动删除
- **Scheduled task 专用**：定时任务跑在专用后台 worktree，不干扰前台工作

**Handoff 的工程意义**：chat header 一键在 Local ↔ Worktree 之间迁移，Codex 代管全部 Git 操作（含分支占用冲突处理）。心智模型：**Local 是前台，Worktree 是后台**——与"云是后台"一脉相承，用户对"任务在哪执行"的认知负担被降到最低。

细节亮点：
- `.worktreeinclude` 文件声明 .gitignore 忽略但新 worktree 需要的本地文件（如 .env），创建时复制过去
- worktree 删除前自动快照，重开 chat 可恢复——**"后台执行环境"具备可恢复性**
- 值得注意的坑：`.worktreeinclude` 只对本地桌面 App 管理型 worktree 生效，不适用于远程 worktree 和手工 worktree

### 3.2 机制二：Remote Control（手机遥控 PC）

Codex Remote 打通"人不在电脑前"的场景：

```
手机 ChatGPT App ──QR 配对──> 桌面 App Connections
        │                          │
        ▼                          ▼
  Remote 面板              codex remote-control 守护进程
  - 查看任务进度            (app-server daemon + remote control)
  - 批准命令执行                     │
  - 审查 diff                 本地仓库/worktree
  - 发起新任务                （命令仍在本机执行）
```

CLI 侧的对称实现（源码 remote_control_cmd.rs）：
- `codex remote-control start`：启动 app-server 守护进程并启用远程控制
- `codex remote-control stop` / `pair`（短期手动配对码）
- 前台模式直接 `codex remote-control`

**安全模型**：手机端只做"遥控器"，仓库、worktree、命令执行全部留在 PC 上；批准动作（approval requests）在手机上呈现，本地执行。这避免了"代码上云"的数据边界问题——与 Codex cloud（代码必须 push 到 GitHub 才能跑）形成互补：
- **代码不出本机的异步** = Remote Control
- **代码在云端跑的异步** = Codex cloud

### 3.3 机制三：App Server 协议（跨表面状态总线）

所有本地表面（桌面 App、IDE、CLI TUI、SDK）共享同一个后端协议——**这是 Codex "一套状态、多种表面"策略的技术底座**：

- **核心原语**：Thread（会话）→ Turn（轮次）→ Item（工作单元：消息/命令/文件变更/工具调用）
- **传输**：JSONL-over-stdio（本地进程）/ WebSocket（`ws://`/`wss://`）/ Unix socket
- **跨机器远程**：`codex app-server --listen ws://IP:PORT` 在 A 机启动，`codex --remote ws://...` 从 B 机连接 TUI——**终端 UI 与 agent 核心可分机部署**，支持 bearer token 认证
- **关键 API**：`thread/shellCommand`（沙箱外用户命令）、`review/start`（多种 diff 目标）、`environment/info`（远程环境探测）、hooks、approvals、MCP elicitation 等

**与 dsh 的 Cordis 内核对照**：app-server 协议的角色类似于 dsh 的插件系统事件总线，但它更进一步——把"会话状态"本身作为一等公民序列化（Thread/Turn/Item 持久化存储、fork、resume），任何表面都能恢复任意 thread。这与 ASL 方案的"turn 复合检查点"目标一致，可作为协议设计参照。

### 3.4 机制四：会话与配置的跨 Agent 迁移（Import）

2026-08-11 上线的 Import 功能支持从 Claude Code / Claude Cowork / Cursor 导入：

| 迁移项 | 目的地 |
|---|---|
| 指令文件 | AGENTS.md |
| settings.json | config.toml |
| Skills / Slash commands | Skills |
| Plugins / MCP 配置 | Plugins / Codex MCP 配置 |
| 项目 memories | Memories |
| 近 30 天 chats（CLI 最多 50 个） | ChatGPT chats |
| Hooks / Subagents | Codex hooks / subagents |

**战略意图**：降低从竞品迁移的摩擦到接近零。配合"导入后自动同步更新"，Codex 事实上把竞争对手的生态资产（skills、配置、会话历史）变成了自己的获客渠道。这是云-端协作之上的"**跨 Agent 协作**"——争夺的是 Agent 时代的用户数据主权。

---

## 四、竞品对比分析

### 4.1 云-端协作能力矩阵

| 能力 | Codex | Claude Code | Cursor | Copilot | Devin | Jules |
|---|---|---|---|---|---|---|
| 云任务提交 | `codex cloud exec --env` | `claude --cloud "task"` | Cloud Agent（UI/CLI） | issue 指派 / chat 委派 | Web / API | Web / CLI / API |
| 云→本地结果回收 | `apply`（diff）/ PR | `--teleport`（会话+分支）/ PR | Check out Locally / Apply locally / PR | PR | `/handoff` 反向 / PR | PR |
| 本地→云会话迁移 | 桌面 App 内（无 CLI 一键迁移） | 桌面 App "Continue in" | ✅（chat 委派） | ✅（/delegate、chat handoff） | ✅ `/handoff` | ❌（纯云原生） |
| 手机遥控本地 | Remote + `remote-control` | Remote Control（server/--rc） | ❌ | ❌ | ❌ | ❌ |
| 远程环境探测 | `environment/info`（实验） | worktree spawn 模式 | ✅ | ❌ | snapshot 体系 | snapshot 体系 |
| Best-of-N | `--attempts 1-4` | ❌ | ❌ | ❌ | ❌ | ❌ |
| 环境缓存复用 | 12h 容器缓存 | 无（每次 clone） | Dockerfile 层缓存 70% 提速 | GitHub Actions runner | org 级 snapshot | Environment Snapshot |
| 断网沙箱（agent 阶段） | 默认断网可开 | — | — | — | 保留网络 | 保留网络 |
| 导入竞品配置 | Claude Code/Cursor | ❌ | ❌ | Claude/Codex partner | ❌ | ❌ |
| 开源核心 | ✅ CLI + app-server + universal 镜像 | ✅ CLI | ❌ | ❌ | ❌ | ❌ |

### 4.2 三种"云→本地"回收哲学

1. **Diff 哲学（Codex）**：云端产出 unified diff，本地 `git apply` + preflight。轻量、可审查、可部分应用（Partial 状态），但丢弃了云端的对话上下文
2. **会话哲学（Claude Code `--teleport`）**：拉回完整会话历史 + fetch/checkout 云端分支，本地"继承"云端的全部上下文继续工作。重，但连续性最强
3. **PR 哲学（Jules / Copilot / 默认路径）**：一切经由 GitHub PR，评审即回收。最符合团队协作习惯，但个人快速迭代场景多一跳

**观察**：Codex 的 apply 与 Claude 的 teleport 分别站在哲学光谱的两端，Cursor 居中（三选项全给）。行业尚未收敛，长期看"diff + 可选会话恢复"可能合并为默认。

### 4.3 环境复现的技术分层

竞品在"云环境冷启动"问题上的解法分层清晰：

| 层次 | 代表 | 机制 |
|---|---|---|
| 应用层缓存 | Codex | 12h 容器缓存 + maintenance script |
| 构建层缓存 | Cursor 3.4 | Dockerfile 层缓存（70% 提速） |
| 镜像层快照 | Devin / Jules | org 级 / repo 级 Environment Snapshot（可启动镜像） |
| 源码层复现 | Codex universal | 开源 Dockerfile，本地可验证 |

这与 ASL 方案的三层镜像 + 预热池设计同构——验证了该方向的行业共识。Codex 额外贡献了"**maintenance script**"概念：缓存容器恢复时执行增量维护，而非全量重建，这是对"缓存失效一致性"问题的精细处理。

### 4.4 产品策略差异

- **Codex**：协议统一派——一套 Thread 状态 + app-server 总线，表面最大化（CLI/App/IDE/云/手机/SDK），开源核心换生态
- **Claude Code**：工作流统一派——plan locally, execute remotely 的哲学（本地 plan mode 协作出方案，上云执行），teleport 保持会话连续性
- **Cursor**：编辑器优先派——云 agent 是编辑器的后台延伸，本地体验优先
- **Copilot**：平台整合派——GitHub 生态内闭环（issue→PR），VS Code 做统一会话管理器
- **Devin**：全托管派——CLI 只做入口，环境 snapshot 是核心资产，"给 Devin 一台电脑"
- **Jules**：极简异步派——无本地组件，纯云原生，计划自审（Planning Critic）控质量

---

## 五、学术前沿与评测演进

### 5.1 从静态基准到交互式会话评测

云-端协作 Agent 的兴起直接推动了评测范式的转变：

- **SWE-Together**（arXiv 2606.29957, 2026）：专门评估编码 Agent 在**交互式用户会话**中的表现——对应真实场景中"人监督 Agent 云端工作"的协作模式，而非单轮 issue 修复
- **SWE-chat**（Baumann et al., 2026）：真实世界编码 Agent 会话的大规模特征化数据集
- **SWE-Bench Pro**（arXiv 2509.16941, 2025）：SWE-Bench Verified 被 SOTA Agent 攻破 70%+ 后，转向**长时程**软件工程任务

### 5.2 数字背后的语义正确性危机

2026-05 时点：SWE-Bench Verified 上 Claude Mythos Preview 93.9%（但 19.78% 的"已解决"案例语义上是错的）、Claude Opus 4.7 87.6%、GPT-5.3 Codex 85%。**高分不等于可用**——这正解释了为什么全行业都在加"人工审查点"（diff review、plan review、PR review）：云-端协作模式的本质，是把 Agent 的产出放回人的评审回路。

### 5.3 对评测的三点推论

1. **长时程 + 交互式**是下一代评测的双重轴——单轮静态基准的信息量正在枯竭
2. **语义正确性审计**（用户基准测试方法论的核心）与 SWE-Together 类交互式评测天然契合：噪声标注在交互场景下会被人的 follow-up 自然暴露
3. 云-端协作 Agent 的"完成率"指标应按"任务最终被接受（PR merged / diff applied）"统计，而非"模型自认完成"——Codex 的 ApplyStatus 三态是朝这个方向的产品化雏形

---

## 六、对 dsh 记忆插件与 ASL 方案的启示

### 6.1 可直接借鉴的机制

| 机制 | 来源 | 对 ASL 的启示 |
|---|---|---|
| Thread/Turn/Item 三级原语 + 持久化 | app-server 协议 | ASL 的 turn 复合检查点可参照其序列化边界：turn 是原子单元，item 是事件粒度 |
| Worktree Handoff（Git 代管迁移） | Codex app | ckpt 四元组中的"文件快照 + git SHA"组合，正是 Handoff 的通用化 |
| 缓存容器 + maintenance script | Codex cloud | ASL 预热池的维护策略：恢复时增量维护优于全量重建 |
| Best-of-N attempts + 人工挑选 | `codex cloud exec --attempts` | 失败分支→程序化记忆闭环中，"多尝试 + 挑选"是天然的记忆素材来源 |
| `.worktreeinclude` 声明式文件复制 | Codex app | 跨环境状态迁移时"ignored-but-needed"文件的通用问题，声明式清单是优雅解 |
| Secrets 分阶段隔离 | Codex cloud | ASL 云沙箱的 secret 生命周期管理：setup 注入、agent 前清除 |
| Import 竞品配置 | Codex /import | Agent 状态层的互操作性：跨 Agent 迁移是刚需，标准化的迁移清单值得纳入 ASL 设计 |

### 6.2 市场空档分析

1. **会话 + diff 双通道回收**：Codex 只有 diff、Claude 只有会话，"云端产出回收"缺乏统一抽象——ASL 的检查点体系若能同时携带"结果状态 + 会话上下文"，是明确的产品化差异点
2. **跨 Agent 状态互操作**：Codex Import 证明了需求真实性，但只覆盖配置和 30 天会话。ASL 的异构 Agent 上下文装配服务（Tier 3/2/1 分级）瞄准的正是这个空档
3. **云-端记忆一致性**：现有产品的"记忆"（memories）都是端侧本地的；云端任务产出如何回流为持久记忆（而非一次性 diff），没有任何厂商解决——这恰是 dsh 记忆插件 + ASL 的交叉创新点：**云任务的结果摘要、失败分支、代码状态可以作为记忆代次的输入源**

### 6.3 风险与不确定性

- Codex cloud 命令族仍是 EXPERIMENTAL 标注，接口可能变动
- 竞品能力迭代极快（Cursor 3.4、Claude Code web 每月更新），本报告对比基于 2026-09 时点
- 学术评测（SWE-Together 等）尚在早期，结论待验证

---

## 附录 A：Codex 云-端协作时间线

| 时间 | 事件 |
|---|---|
| 2025-06 | Codex cloud（前身 codex-1）以 research preview 上线，正式开启"云端 Agent"竞争 |
| 2026-02-02 | Codex app（macOS）发布：并行项目 chat、内置 Git review、worktrees、skills、scheduled tasks、语音听写 |
| 2026-02-12 | GPT-5.3-Codex-Spark 研究预览（实时迭代模型）+ chat forking |
| 2026-03-03 | Local ↔ Worktree Handoff 上线 |
| 2026-03-04 | Codex app 登陆 Windows（原生 PowerShell + 沙箱） |
| 2026-03-05 | GPT-5.4 到达 Codex |
| 2026-05-13 | Cursor 3.4 云 Agent 缓存提速（竞品参照点） |
| 2026-08-11 | Import（Claude Code/Cursor → Codex）+ Linux 桌面 App preview |
| 2026-08-19 | GitLab 支持 beta（云任务可从 issue/MR 触发） |
| 2026-08-25 | 浏览器扩展（Edge/Brave/Opera/Vivaldi）+ GPT-5.6 Sol/Terra/Luna |
| 2026-09 | `codex cloud` 五命令族 + remote-control 体系成型（源码 0.155.0-alpha） |

## 附录 B：来源索引

**官方文档（learn.chatgpt.com / developers.openai.com，2026-09-11 抓取）**
- Codex cloud 总览：learn.chatgpt.com/docs/cloud.md
- 云环境配置：learn.chatgpt.com/docs/environments/cloud-environment.md
- 环境模式：learn.chatgpt.com/docs/environments/modes.md
- Worktrees：learn.chatgpt.com/docs/environments/git-worktrees.md
- Codex app：learn.chatgpt.com/docs/app.md
- App Server 协议：learn.chatgpt.com/docs/app-server.md
- Codex SDK：learn.chatgpt.com/docs/codex-sdk.md
- Codex CLI：learn.chatgpt.com/docs/codex/cli.md
- Remote：learn.chatgpt.com/docs/remote.md、remote-connections.md
- 命令参考：learn.chatgpt.com/docs/developer-commands.md?surface=cli
- What's new 周报：learn.chatgpt.com/docs/whats-new.md
- Import：learn.chatgpt.com/docs/import.md
- Long-running work / Glossary / app 页面

**GitHub 源码（api.github.com，2026-09-11 抓取）**
- openai/codex（123,149 stars）：README、codex-rs/cli/src/main.rs、cloud_config.rs、remote_control_cmd.rs、codex-rs/cloud-tasks/src/{cli,lib,env_detect}.rs
- openai/codex-universal（云环境参考镜像）

**竞品资料（2026-09-11 检索）**
- Claude Code on the web / Remote Control：code.claude.com/docs/en/claude-code-on-the-web、/docs/en/remote-control、/docs/en/sessions
- Cursor Cloud Agents：cursor.com 文档与社区评测（madewithlove.com、stevekinney.com、blink.new）
- Copilot 四类 Agent：code.visualstudio.com/docs/copilot/agents/cloud-agents、copilot-cloud-agent、agents-handoff-tutorial
- Devin：docs.devin.ai/work-with-devin/devin-cli、onboard-devin/environment、cognition.com/blog/devin-for-terminal
- Jules：jules.google、kie.ai、kdnuggets、dev.to 等第三方评测与 Google 官方公告转述

**学术文献**
- SWE-Together: Evaluating Coding Agents in Interactive User Sessions（arXiv 2606.29957）
- SWE-Bench Pro: Can AI Agents Solve Long-Horizon Software Engineering Tasks?（arXiv 2509.16941）
- OpenHands: An Open Platform for AI Software Developers as Generalist Agents（arXiv 2407.16741）
- Agentless: Demystifying LLM-Based Software Engineering Agents（arXiv 2407.01489）
- SWE-chat（Baumann et al., 2026，转引自 arXiv 2606.29957）
