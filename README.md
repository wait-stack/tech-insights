# Tech Insights

AI Agent 记忆策略层技术洞察报告仓库。

## 目录

| 文件 | 内容 |
|------|------|
| [Pi-Agent与DeepSeek-Harness技术洞察_2026-09.md](./Pi-Agent与DeepSeek-Harness技术洞察_2026-09.md) | 固定源码对比 Pi 与 DSH 的运行循环、插件、会话和模型层复用，补充 OpenCode / Claude Code 产品对照，给出双宿主记忆适配与验证方案 |
| [DeepSeek-Harness场景化记忆接入洞察与实施方案_2026-09.md](./DeepSeek-Harness场景化记忆接入洞察与实施方案_2026-09.md) | 基于固定源码核实 Harness/DSH 原理、Mem0 原生工具插件与 MemOS/Mem9 接入；给出现有记忆系统的场景、接口、异步一致性、四周实施与评测方案，并校正早期接口示例 |
| [Agent记忆模块全景与最新进展_2026-09.md](./Agent记忆模块全景与最新进展_2026-09.md) | 截至2026年9月的Agent记忆全景调研，覆盖最新论文、开源项目、Benchmark、云厂商与商业产品，并给出可落地的分层架构和实施路线 |
| [超长轮次Agent记忆最新技术洞察报告_2026-08.md](./超长轮次Agent记忆最新技术洞察报告_2026-08.md) | 2026年7-8月最新论文+产品动态，覆盖15篇arXiv论文、Mem0/Letta/Graphiti/Cognee开源项目更新、六大技术趋势分析 |
| [Agent记忆冲突解决技术深度分析.md](./Agent记忆冲突解决技术深度分析.md) | 新旧记忆冲突解决六大策略深度对比，含源码级分析（Mem0/Graphiti/Letta/Cognee）+ 超越Mem0 94.4分的实施路径 |
| [Mem0商业版Supersede冲突解决机制调研_2026-08.md](./Mem0商业版Supersede冲突解决机制调研_2026-08.md) | Mem0 Platform v3 Dream Supersede 的公开文档、API 契约和读写语义调研 |
| [Dream-Supersede记忆冲突解决与召回设计_2026-08.md](./Dream-Supersede记忆冲突解决与召回设计_2026-08.md) | 基于 Supersede 思路设计可审计、可撤销的新旧记忆冲突解决和双视图召回系统 |
| [Mem0商业版Supersede实现全流程与实验取证_2026-08.md](./Mem0商业版Supersede实现全流程与实验取证_2026-08.md) | 结合 Mem0 OSS 与多组 Platform 黑盒实验，推断关联召回、生命周期关系、数据模型和一致性边界 |
| [两阶段LLM新旧记忆关联更新方案_v3_2026-08.md](./两阶段LLM新旧记忆关联更新方案_v3_2026-08.md) | 两次 LLM 调用的新 Fact 抽取与新旧记忆关联方案，定义 NONE、REPLACE、MERGE 及三个关系字段 |
| [同步与异步记忆关联判断调研_2026-08.md](./同步与异步记忆关联判断调研_2026-08.md) | 对比关联判断的触发时机、一致性和成本，分析异步新旧倒挂问题并给出同步主链路建议 |
| [记忆关系感知召回方案_v1_2026-08.md](./记忆关系感知召回方案_v1_2026-08.md) | 根据查询意图折叠或展开新旧记忆关系，覆盖默认、最新、指定时间、变化历史和原始证据召回 |
| [LongMemEval_Assistant题型意图分类与典型样例_2026-08.md](./LongMemEval_Assistant题型意图分类与典型样例_2026-08.md) | 统计 `single-session-assistant` 题型的 10 类原始任务意图，含角色标注偏差、英文证据对话、中文翻译和记忆设计启示 |
| [LongMemEval_MultiSession关联拓扑分类与多跳标准_2026-08.md](./LongMemEval_MultiSession关联拓扑分类与多跳标准_2026-08.md) | 对 133 道 `multi-session` 题逐题编码，区分同话题直接关联、同锚点组合、严格桥接多跳与信息不足对照题 |
| [LongMemEval_TemporalReasoning题型分类与时间推理标准_2026-08.md](./LongMemEval_TemporalReasoning题型分类与时间推理标准_2026-08.md) | 对 133 道 `temporal-reasoning` 题逐题编码，覆盖查询时点回溯、事件间隔、持续时间、排序、时间反查、窗口聚合与信息不足 |

## 关注方向

- **超长会话轮次记忆准确度**：基于 LongMemEval 基准测试攻关，目标超越 Mem0 的 94.4 分
- **记忆冲突解决**：新旧信息矛盾时的检测、取代、撤销机制
- **技术栈**：Mem0（记忆模块）、硅基流动（Embedding）、Hermes Agent（微信网关）

## 数据来源

arXiv API、GitHub 源码分析、项目官方文档与发布页面。
