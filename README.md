# Tech Insights

AI Agent 记忆策略层技术洞察报告仓库。

## 目录

| 文件 | 内容 |
|------|------|
| [超长轮次Agent记忆最新技术洞察报告_2026-08.md](./超长轮次Agent记忆最新技术洞察报告_2026-08.md) | 2026年7-8月最新论文+产品动态，覆盖15篇arXiv论文、Mem0/Letta/Graphiti/Cognee开源项目更新、六大技术趋势分析 |
| [Agent记忆冲突解决技术深度分析.md](./Agent记忆冲突解决技术深度分析.md) | 新旧记忆冲突解决六大策略深度对比，含源码级分析（Mem0/Graphiti/Letta/Cognee）+ 超越Mem0 94.4分的实施路径 |
| [Mem0商业版Supersede冲突解决机制调研_2026-08.md](./Mem0商业版Supersede冲突解决机制调研_2026-08.md) | Mem0 Platform v3 Dream Supersede 的公开文档、API 契约和读写语义调研 |
| [Dream-Supersede记忆冲突解决与召回设计_2026-08.md](./Dream-Supersede记忆冲突解决与召回设计_2026-08.md) | 基于 Supersede 思路设计可审计、可撤销的新旧记忆冲突解决和双视图召回系统 |
| [Mem0商业版Supersede实现全流程与实验取证_2026-08.md](./Mem0商业版Supersede实现全流程与实验取证_2026-08.md) | 结合 Mem0 OSS 与多组 Platform 黑盒实验，推断关联召回、生命周期关系、数据模型和一致性边界 |
| [ADD-only记忆Supersede双向关联落地方案_2026-08.md](./ADD-only记忆Supersede双向关联落地方案_2026-08.md) | 面向当前 ADD-only 管线的可实施方案：权威关系表、双向投影、双路召回、事务一致性、撤销、迁移与灰度 |

## 关注方向

- **超长会话轮次记忆准确度**：基于 LongMemEval 基准测试攻关，目标超越 Mem0 的 94.4 分
- **记忆冲突解决**：新旧信息矛盾时的检测、取代、撤销机制
- **技术栈**：Mem0（记忆模块）、硅基流动（Embedding）、Hermes Agent（微信网关）

## 数据来源

arXiv API、GitHub 源码分析、项目官方文档与发布页面。
