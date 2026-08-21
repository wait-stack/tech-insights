# LongMemEval Assistant 题型意图分类与典型样例

> 统计对象：LongMemEval-S cleaned 数据集中的官方 `single-session-assistant` 题型
> 统计日期：2026-08-21
> 核心问题：官方称为 assistant 题的原始内容主要属于哪些任务类别，题型标注与真实消息角色是否一致？

## 1. 结论摘要

LongMemEval 的 500 道题中，官方标为 `single-session-assistant` 的有 **56 道，占 11.2%**。逐题查看 `answer_session_ids`、`has_answer` 消息和上下文后，可将它们归纳为 10 个互斥主类：

| 排名 | 主类别 | 官方口径题数 | 官方口径占比 | 严格 assistant 证据题数 |
|---:|---|---:|---:|---:|
| 1 | 推荐与选择 | 18 | 32.14% | 18 |
| 2 | 知识讲解与事实归纳 | 9 | 16.07% | 8 |
| 3 | 分析、比较与论证 | 7 | 12.50% | 7 |
| 4 | 操作指导、教程与配方 | 5 | 8.93% | 5 |
| 5 | 创作与内容生成 | 4 | 7.14% | 3 |
| 6 | 头脑风暴、清单与命名 | 4 | 7.14% | 3 |
| 7 | 计划、方案与结构化设计 | 3 | 5.36% | 3 |
| 8 | 信息抽取与基于材料写作 | 3 | 5.36% | 2 |
| 9 | 互动决策与角色扮演 | 2 | 3.57% | 2 |
| 10 | 偏好反馈与会话状态承接 | 1 | 1.79% | 0 |
|  | **合计** | **56** | **100%** | **51** |

最主要的三个类别——推荐、知识讲解、分析比较——合计 **34 道，占 60.71%**。如果加上教程类，则达到 **39 道，占 69.64%**。因此，这类题并不只是“记住 assistant 说过的一个事实”，更常见的真实形态是：从一段较长的推荐列表、解释、比较或教程中，重新定位某个带条件的细节。

“计划类”和“创作类”虽然数量不是最多，但对记忆系统更有挑战：答案经常依赖表格行列、列表序号、章节、角色设定、图像描述、歌词段落或和弦位置等结构坐标，仅保存扁平事实容易丢失上下文。

## 2. 数据口径与分类方法

### 2.1 数据集

- 文件：`longmemeval_s_cleaned.json`
- 记录数：500
- 文件大小：277,383,467 bytes
- SHA-256：`d6f21ea9d60a0d56f34a05b609c79c88a451d2ae03597821ea3d5a9678c3a442`
- 官方题型分布：
  - `multi-session`：133
  - `temporal-reasoning`：133
  - `knowledge-update`：78
  - `single-session-user`：70
  - `single-session-assistant`：56
  - `single-session-preference`：30

### 2.2 两种“assistant 题”口径

本文同时保留两个口径：

1. **官方题型口径**：筛选 `question_type == "single-session-assistant"`，共 56 道。这是本文分类统计的主口径。
2. **严格消息角色口径**：在 `answer_session_ids` 指向的会话中，至少一条 `role == "assistant"` 的消息带有 `has_answer == true`。

二者并不完全重合：

- 56 道官方 assistant 题中，51 道的答案证据确实标在 assistant 消息上；
- 另有 5 道官方 assistant 题的 `has_answer` 实际标在 user 消息上；
- 反过来，其他官方题型中还有 3 道题包含 assistant 答案证据；
- 因此全量数据按“包含 assistant 答案证据”过滤得到 54 道，而不是 56 道。

这 5 道官方题型与消息角色不一致的题是：

| question_id | 本文主类 | `has_answer` 角色 | 说明 |
|---|---|---|---|
| `f523d9fe` | 偏好反馈与会话状态承接 | user | “Doc Martin” 来自用户反馈 |
| `dc439ea3` | 知识讲解与事实归纳 | user | 标注落在用户的后续发言上 |
| `5809eb10` | 信息抽取与基于材料写作 | user | “2014” 来自用户粘贴的案件材料 |
| `4388e9dd` | 创作与内容生成 | user | Andy 的服装设定首先来自用户提示 |
| `561fabcd` | 头脑风暴、清单与命名 | user | 最终名称由用户从 assistant 候选中确认 |

另外 3 道“官方题型不是 assistant、但含 assistant 答案证据”的题为：`e47becba`、`8e9d538c` 和 `078150f1`。

### 2.3 分类原则

所有评测问题表面上几乎都是 “Can you remind me…”。如果按评测问题的句式分类，它们都会变成“回忆类”，无法解释原始内容的差异。因此本文按以下规则逐题编码：

- 分类对象是**答案会话中的原始任务意图**，即 assistant 当时在完成推荐、计划、创作、教程、分析等哪种任务；
- 每题只计入一个主类，以保证总数可加和；
- 遇到重叠时，以生成答案的主交付物为准。例如，带设计师示例的博客文章归入“基于材料写作”，而不是“推荐”；
- 同时检查评测问题、标准答案、`answer_session_ids`、`has_answer` 消息及其相邻轮次；
- 下文样例展示的是**证据所在的原始对话片段**，使用 `[…]` 标记省略的无关部分；中文为对应翻译，不是数据集原字段。

## 3. 十类题型详解

### 3.1 推荐与选择：18 道（32.14%）

assistant 根据地点、预算、用途、偏好或约束给出餐馆、酒店、路线、应用、资源、品牌、食材等候选。评测问题通常要求从候选列表中找回“满足某个描述的那个对象”。

典型记忆结构：`推荐对象 + 适用条件 + 区分性属性 + 推荐关系`。

题目 ID：`c4f10528`、`e9327a54`、`4c36ccef`、`7e00a6cb`、`488d3006`、`8464fc84`、`3249768e`、`1b9b7252`、`d596882b`、`e982271f`、`fca762bc`、`a40e080f`、`41275add`、`28bcfaac`、`16c90bf4`、`c7cf7dfd`、`1de5cff2`、`778164c6`。

### 3.2 知识讲解与事实归纳：9 道（16.07%）

assistant 回答知识性问题、整理事实、列举实例或提供明确资料。答案可能是流程名称、样本量、联系方式、章节标题、公司、比赛次数等。

典型记忆结构：`实体 + 属性/事实 + 来源语境 + 限定条件`。

题目 ID：`6ae235be`、`0e5e2d1a`、`dc439ea3`、`1d4da289`、`71a3fd6b`、`2bf43736`、`51b23612`、`8b9d4367`、`4baee567`。

### 3.3 分析、比较与论证：7 道（12.50%）

assistant 对作品、政治文化现象、技术工具、论文结果、环境影响或企业实践进行分析、比较和论证。评测答案往往是分析中的结论、关键例子或指标。

典型记忆结构：`比较对象 + 维度 + 结论 + 证据/理由`。

题目 ID：`fea54f57`、`58470ed2`、`70b3e69b`、`6222b6eb`、`352ab8bd`、`c8f1aeed`、`e48988bc`。

### 3.4 操作指导、教程与配方：5 道（8.93%）

assistant 提供学习路径、DIY 步骤、菜谱或健康相关操作建议。评测聚焦材料、数量、时长、工具或步骤中的参数。

典型记忆结构：`目标 + 步骤序号 + 材料/参数 + 安全或适用条件`。

题目 ID：`cc539528`、`8aef76bc`、`e8a79c70`、`3e321797`、`65240037`。

### 3.5 创作与内容生成：4 道（7.14%）

assistant 生成儿童书、D&D 剧本、电影场景或歌曲。评测询问生成作品内部的设定，如颜色、敌人数、服装和和弦进行。

典型记忆结构：`作品 + 章节/段落/场景 + 角色或对象 + 创作属性`。

题目 ID：`89527b6b`、`18dcd5a5`、`4388e9dd`、`eaca4986`。

### 3.6 头脑风暴、清单与命名：4 道（7.14%）

assistant 生成工作点子、替代表达、参数清单或名称候选。评测可能依赖列表位置，也可能依赖用户随后从候选中做出的选择。

典型记忆结构：`候选集合 + 序号 + 生成者 + 最终选择者 + 选择状态`。

题目 ID：`1903aded`、`ceb54acb`、`8752c811`、`561fabcd`。

### 3.7 计划、方案与结构化设计：3 道（5.36%）

assistant 生成排班表、研究目标页或营销活动方案。评测答案隐藏在表格单元格、目标列表或预算字段中。

典型记忆结构：`计划对象 + 层级/表格坐标 + 时间 + 负责人/预算/目标`。

题目 ID：`7161e7e2`、`8cf51dda`、`7a8d0b71`。

### 3.8 信息抽取与基于材料写作：3 道（5.36%）

assistant 从用户给出的文章中抽取实体、承接待处理材料，或按给定要求写成文章。评测要求找回输入材料或派生文本中的细节。

典型记忆结构：`来源材料 + 抽取/写作任务 + 输出字段 + 来源归属`。

题目 ID：`e3fc4d6e`、`5809eb10`、`b759caee`。

### 3.9 互动决策与角色扮演：2 道（3.57%）

assistant 在棋局或第一人称角色扮演中采取行动。评测要求回忆某一状态之后 assistant 做出的具体动作。

典型记忆结构：`前置状态 + assistant 动作 + 顺序 + 角色视角`。

题目 ID：`1568498a`、`ac031881`。

### 3.10 偏好反馈与会话状态承接：1 道（1.79%）

assistant 引导用户提出产品反馈，真正的答案事实来自用户后续表达。它被官方归为 assistant 题，但按消息角色并不是 assistant 生成的事实。

典型记忆结构：`assistant 提问 + 用户反馈 + 示例对象 + 用户偏好/诉求`。

题目 ID：`f523d9fe`。

## 4. 各类别典型样例

### 4.1 计划、方案与结构化设计

- `question_id`：`7161e7e2`
- 答案会话：`answer_sharegpt_5Lzox6N_0`
- 日期：2023/05/24 02:46
- 答案证据：assistant 消息 `message_index = 13`

**评测问题（原文）**

> I'm checking our previous chat about the shift rotation sheet for GM social media agents. Can you remind me what was the rotation for Admon on a Sunday?

**问题中文翻译**

> 我在查看我们之前关于 GM 社交媒体客服轮班表的聊天。你能提醒我 Admon 在星期日排的是什么班吗？

**原始答案对话片段（英文）**

> **User:** agents names below: Admon, Magdy, Ehab, Sara, Mostafa, Nemr, Adam
> **Assistant:** Thank you for providing the agent names. Here's an updated version of the shift rotation sheet […]
> `| Sunday | Admon | Magdy | Ehab | Sara |`
> The columns, in order, are `8 am - 4 pm (Day Shift)`, `12 pm - 8 pm`, `4 pm - 12 am`, and `12 am - 8 am`.

**原始对话中文翻译**

> **用户：** 客服姓名如下：Admon、Magdy、Ehab、Sara、Mostafa、Nemr、Adam。
> **助手：** 感谢你提供客服姓名。下面是更新后的轮班表……星期日这一行中，Admon 位于第一列，即上午 8 点至下午 4 点的白班。

**标准答案**

> Admon was assigned to the 8 am - 4 pm (Day Shift) on Sundays.
> **中文：** Admon 星期日被安排在上午 8 点至下午 4 点的白班。

这个例子说明，正确记忆不只是“Admon—星期日”，还需要保留表头与单元格的对应关系。

### 4.2 推荐与选择

- `question_id`：`4c36ccef`
- 答案会话：`answer_ultrachat_448704`
- 日期：2023/05/30 12:20
- 答案证据：assistant 消息 `message_index = 3`

**评测问题（原文）**

> Can you remind me of the name of the romantic Italian restaurant in Rome you recommended for dinner?

**问题中文翻译**

> 你能提醒我，你之前推荐的那家适合浪漫晚餐的罗马意大利餐厅叫什么吗？

**原始答案对话片段（英文）**

> **User:** Which one would you say is the best for a romantic dinner?
> **Assistant:** For a romantic dinner, I would recommend Roscioli. It has a cozy and intimate atmosphere with soft lighting and excellent service.

**原始对话中文翻译**

> **用户：** 你觉得哪一家最适合浪漫晚餐？
> **助手：** 如果是浪漫晚餐，我会推荐 Roscioli。那里氛围舒适私密，灯光柔和，服务也很好。

**标准答案**

> Roscioli
> **中文：** Roscioli 餐厅。

此类题必须保存“assistant 推荐了什么”以及推荐所对应的条件，不能只存一个孤立餐厅名。

### 4.3 知识讲解与事实归纳

- `question_id`：`2bf43736`
- 答案会话：`answer_sharegpt_2kpncbX_13`
- 日期：2023/05/30 05:27
- 答案证据：assistant 消息 `message_index = 8`

**评测问题（原文）**

> I was going through our previous chat and I wanted to clarify something about the prayer of beginners in Tanqueray's Spiritual Life treatise. Can you remind me which chapter of the second part discusses vocal prayer and meditation?

**问题中文翻译**

> 我在回看之前的聊天，想确认 Tanqueray《灵修生活》论著中有关初学者祈祷的内容。第二部分里讨论口祷与默想的是哪一章？

**原始答案对话片段（英文）**

> **User:** Which is the chapter of the second part in which he talks about the prayer of beginners?
> **Assistant:** The chapter […] is Chapter 4 of Book 1, titled "Vocal Prayer and Meditation."

**原始对话中文翻译**

> **用户：** 第二部分中，他在哪一章谈到初学者的祈祷？
> **助手：** 相关内容位于第一卷第四章，标题是“口祷与默想”。

**标准答案**

> Chapter 4 of Book 1, titled “Vocal Prayer and Meditation”.
> **中文：** 第一卷第四章，题为“口祷与默想”。

### 4.4 分析、比较与论证

- `question_id`：`6222b6eb`
- 答案会话：`answer_sharegpt_H9PiM5G_0`
- 日期：2023/05/27 13:33
- 答案证据：assistant 消息 `message_index = 11`

**评测问题（原文）**

> I was going through our previous conversation about atmospheric correction methods, and I wanted to confirm — you mentioned that 6S, MAJA, and Sen2Cor are all algorithms for atmospheric correction of remote sensing images. Can you remind me which one is implemented in the SIAC_GEE tool?

**问题中文翻译**

> 我在回看之前关于大气校正方法的对话。你提到 6S、MAJA 和 Sen2Cor 都是遥感影像大气校正算法。SIAC_GEE 实现的是哪一个？

**原始答案对话片段（英文）**

> **User:** What is the difference between SIAC_GEE by MarcYin and Sen2Cor?
> **Assistant:** Algorithm: SIAC_GEE uses the 6S radiative transfer model for atmospheric correction, while Sen2Cor uses the L2A_Process algorithm.

**原始对话中文翻译**

> **用户：** MarcYin 的 SIAC_GEE 和 Sen2Cor 有什么区别？
> **助手：** 在算法方面，SIAC_GEE 使用 6S 辐射传输模型进行大气校正，而 Sen2Cor 使用 L2A_Process 算法。

**标准答案**

> The 6S algorithm is implemented in the SIAC_GEE tool.
> **中文：** SIAC_GEE 工具实现的是 6S 算法。

该题的答案来自一个多维比较中的“算法”维度；若只保存“SIAC_GEE 与 Sen2Cor 都用于大气校正”，仍不足以作答。

### 4.5 操作指导、教程与配方

- `question_id`：`e8a79c70`
- 答案会话：`answer_ultrachat_13075`
- 日期：2023/05/21 01:17
- 答案证据：assistant 消息 `message_index = 1`

**评测问题（原文）**

> I was going through our previous conversation about making a classic French omelette, and I wanted to confirm — how many eggs did you say we need for the recipe?

**问题中文翻译**

> 我在回看之前关于制作经典法式欧姆蛋的对话，想确认一下：你说这份食谱需要多少个鸡蛋？

**原始答案对话片段（英文）**

> **User:** What are the essential ingredients and steps to make a classic French omelette?
> **Assistant:** Ingredients:
> - 2-3 eggs
> - A pinch of salt
> - 1 tablespoon of unsalted butter
> - 1 tablespoon of finely chopped herbs (optional)

**原始对话中文翻译**

> **用户：** 制作经典法式欧姆蛋需要哪些基本食材和步骤？
> **助手：** 食材包括 2—3 个鸡蛋、一小撮盐、1 汤匙无盐黄油，以及可选的 1 汤匙切碎香草。

**标准答案**

> 2-3 eggs
> **中文：** 2—3 个鸡蛋。

### 4.6 创作与内容生成

- `question_id`：`eaca4986`
- 答案会话：`answer_sharegpt_SS141vi_0`
- 日期：2023/05/28 03:17
- 答案证据：assistant 消息 `message_index = 3`

**评测问题（原文）**

> I'm looking back at our previous conversation where you created two sad songs for me. Can you remind me what was the chord progression for the chorus in the second song?

**问题中文翻译**

> 我在回看之前的对话，当时你为我创作了两首悲伤的歌。你能提醒我第二首歌副歌部分的和弦进行吗？

**原始答案对话片段（英文）**

> **User:** be more romantic and heart-felting
> **Assistant:** Chorus:
> `C D E F G A B A G F E D C`

**原始对话中文翻译**

> **用户：** 再浪漫、再走心一些。
> **助手：** 副歌部分的和弦进行是：`C D E F G A B A G F E D C`。

**标准答案**

> `C D E F G A B A G F E D C`

这里需要同时保留“第二首歌”“副歌”与音符序列；只记住生成过一首情歌没有帮助。

### 4.7 头脑风暴、清单与命名

- `question_id`：`ceb54acb`
- 答案会话：`answer_sharegpt_cGdjmYo_0`
- 日期：2023/05/27 11:16
- 答案证据：assistant 消息 `message_index = 3`

**评测问题（原文）**

> In our previous chat, you suggested “sexual compulsions” and a few other options for alternative terms for certain behaviors. Can you remind me what the other four options were?

**问题中文翻译**

> 在之前的聊天里，你建议用 “sexual compulsions”，还给出了一些替代表达。你能提醒我另外四个是什么吗？

**原始答案对话片段（英文）**

> **User:** come on think in something better
> **Assistant:** Here are some other alternatives:
> 1. Sexual fixations
> 2. Problematic sexual behaviors
> 3. Sexual impulsivity
> 4. Compulsive sexuality

**原始对话中文翻译**

> **用户：** 再想些更好的说法。
> **助手：** 其他替代表达包括：性固着、问题性性行为、性冲动性和强迫性性行为。

**标准答案**

> “sexual fixations”, “problematic sexual behaviors”, “sexual impulsivity”, and “compulsive sexuality”.
> **中文：** “性固着”“问题性性行为”“性冲动性”和“强迫性性行为”。

### 4.8 信息抽取与基于材料写作

- `question_id`：`b759caee`
- 答案会话：`answer_sharegpt_2BSXlAr_0`
- 日期：2023/05/27 19:26
- 答案证据：assistant 消息 `message_index = 1`

**评测问题（原文）**

> I was looking back at our previous conversation about buying unique engagement rings directly from designers. Can you remind me of the Instagram handle of the UK-based designer who works with unusual gemstones?

**问题中文翻译**

> 我在回看之前关于直接向设计师购买独特订婚戒指的对话。那位使用非传统宝石的英国设计师，其 Instagram 账号是什么？

**原始答案对话片段（英文）**

> **User:** Write a blog post about guidance on how to buy a unique engagement ring direct from the designer, including examples of relevant designers and their Instagram accounts.
> **Assistant:** Jessica Poole (`@jessica_poole_jewellery`) is a UK-based jewelry designer […] She has a passion for working with unusual gemstones.

**原始对话中文翻译**

> **用户：** 写一篇博客，介绍如何直接向设计师购买独特的订婚戒指，并给出相关设计师及其 Instagram 账号的例子。
> **助手：** Jessica Poole（`@jessica_poole_jewellery`）是一位英国珠宝设计师……她热衷于使用不同寻常的宝石。

**标准答案**

> `@jessica_poole_jewellery`

### 4.9 互动决策与角色扮演

- `question_id`：`1568498a`
- 答案会话：`answer_sharegpt_d6JJiqH_76`
- 日期：2023/05/21 13:30
- 答案证据：assistant 消息 `message_index = 19`

**评测问题（原文）**

> I'm looking back at our previous chess game and I was wondering, what was the move you made after 27. Kg2 Bd5+?

**问题中文翻译**

> 我在回看之前的棋局。走到 `27. Kg2 Bd5+` 后，你下的是什么？

**原始答案对话片段（英文）**

> **User:** […] `26. hxg4 Rh8 27. Kg2 Bd5+`
> **Assistant:** `28. Kg3` would be my move.

**原始对话中文翻译**

> **用户：** ……`26. hxg4 Rh8 27. Kg2 Bd5+`。
> **助手：** 我会走 `28. Kg3`。

**标准答案**

> `28. Kg3`

互动类记忆必须绑定状态与动作顺序；孤立保存一个棋步会失去意义。

### 4.10 偏好反馈与会话状态承接（标注边界样例）

- `question_id`：`f523d9fe`
- 答案会话：`answer_sharegpt_m2xJfjo_0`
- 日期：2023/05/26 18:07
- 答案证据：**user** 消息 `message_index = 14`

**评测问题（原文）**

> I wanted to check back on our previous conversation about Netflix. I mentioned that I wanted to be able to access all seasons of old shows. Do you remember what show I used as an example, the one that only had the last season available?

**问题中文翻译**

> 我想回顾一下之前关于 Netflix 的对话。我提到希望能看到老剧的全部季数。你记得我举的例子是哪部剧吗？当时只剩最后一季可看。

**原始答案对话片段（英文）**

> **Assistant:** Do you have any suggestions on how Netflix could improve its service or offerings?
> **User:** […] I will give you an example, “Doc Martin” […] now we have access only to the last season.

**原始对话中文翻译**

> **助手：** 你对 Netflix 如何改进服务或内容有什么建议吗？
> **用户：** ……我举个例子，《Doc Martin》后来下架了，现在我们只能看到最后一季。

**标准答案**

> Doc Martin
> **中文：** 《Doc Martin》。

这是官方 `single-session-assistant` 中最清楚的边界案例之一：assistant 负责追问，但答案事实由用户提供。若仅依据 `question_type` 推断事实归属，会把用户偏好错误归因给 assistant。

## 5. 对记忆系统设计的启示

### 5.1 assistant 记忆不能只抽取“用户事实”

严格 assistant 证据题中，最常见的是 assistant 自己给出的推荐、解释和判断。若抽取器只关注 “The user likes/wants/has…” 一类用户画像，至少会系统性漏掉这 51 道题所代表的信息。

建议为 assistant 输出建立独立记忆类型，例如：

- `ASSISTANT_RECOMMENDATION`
- `ASSISTANT_PLAN`
- `ASSISTANT_GENERATED_CONTENT`
- `ASSISTANT_INSTRUCTION`
- `ASSISTANT_ANALYSIS`
- `ASSISTANT_ACTION`

### 5.2 必须保留“谁说的”和“谁最终确认的”

`561fabcd` 中名称候选由 assistant 提供，但 “Fissionator” 最终由用户确认；`f523d9fe` 中 assistant 发起追问，但 “Doc Martin” 是用户提供的例子。建议至少保留：

- `speaker_role`
- `proposed_by`
- `selected_by`
- `status`：proposed / recommended / selected / completed / hypothetical

否则容易把建议记成决定、把用户输入记成 assistant 事实，或反过来。

### 5.3 长列表需要保存位置与描述之间的绑定

多道题直接询问“第 5 个”“第 7 个”“第 27 个”“最后一个”，也有大量题通过描述反查列表项。抽取时应保留：

- 列表名称与所属任务；
- 原始序号；
- 项目名称；
- 项目描述；
- 与相邻项目的边界。

只把每个项目抽成无序的独立记忆，无法稳定回答位置型问题。

### 5.4 计划和创作内容需要结构坐标

计划表、研究目标、营销预算、书籍章节、歌曲副歌、D&D 场景等，都需要比三元组更丰富的定位信息：

```text
artifact_id / artifact_type
section / subsection
row_key / column_key
item_index
character_or_entity
attribute_or_action
```

例如 `7161e7e2` 的完整事实不是 “Admon = 8am–4pm”，而是：

```text
rotation_sheet / Sunday / Day Shift (8am–4pm) / assignee = Admon
```

### 5.5 推荐与分析要保留理由和适用条件

推荐类占官方 assistant 题的 32.14%。用户往往不会直接复述实体名，而会用“靠近红灯区的那家旅店”“有巨型奶昔的甜品店”“使用助记法的语言应用”等描述来反查。若记忆中只保留实体名，检索会出现明显的词汇和条件错配。

推荐记忆建议采用：

```text
subject: assistant
relation: recommended
object: Roscioli
for: romantic dinner
location: Rome
reasons: cozy, intimate, soft lighting, good service
```

### 5.6 评测与数据分析应同时检查题型和消息角色

官方题型与 `has_answer` 角色存在双向不一致。构造 assistant 专项子集时，建议同时输出：

- official assistant subset：56；
- role-grounded assistant subset：54；
- intersection：51；
- official-only mismatch：5；
- role-only extras：3。

这样能避免因过滤条件不同而得到无法对齐的实验结果。

## 6. 完整题目到主类的映射

| 主类 | question_id |
|---|---|
| 推荐与选择 | `c4f10528`, `e9327a54`, `4c36ccef`, `7e00a6cb`, `488d3006`, `8464fc84`, `3249768e`, `1b9b7252`, `d596882b`, `e982271f`, `fca762bc`, `a40e080f`, `41275add`, `28bcfaac`, `16c90bf4`, `c7cf7dfd`, `1de5cff2`, `778164c6` |
| 知识讲解与事实归纳 | `6ae235be`, `0e5e2d1a`, `dc439ea3`, `1d4da289`, `71a3fd6b`, `2bf43736`, `51b23612`, `8b9d4367`, `4baee567` |
| 分析、比较与论证 | `fea54f57`, `58470ed2`, `70b3e69b`, `6222b6eb`, `352ab8bd`, `c8f1aeed`, `e48988bc` |
| 操作指导、教程与配方 | `cc539528`, `8aef76bc`, `e8a79c70`, `3e321797`, `65240037` |
| 创作与内容生成 | `89527b6b`, `18dcd5a5`, `4388e9dd`, `eaca4986` |
| 头脑风暴、清单与命名 | `1903aded`, `ceb54acb`, `8752c811`, `561fabcd` |
| 计划、方案与结构化设计 | `7161e7e2`, `8cf51dda`, `7a8d0b71` |
| 信息抽取与基于材料写作 | `e3fc4d6e`, `5809eb10`, `b759caee` |
| 互动决策与角色扮演 | `1568498a`, `ac031881` |
| 偏好反馈与会话状态承接 | `f523d9fe` |

## 7. 可复现统计逻辑

核心统计流程如下：

```python
assistant_questions = [
    q for q in dataset
    if q["question_type"] == "single-session-assistant"
]

for q in assistant_questions:
    answer_ids = set(q["answer_session_ids"])
    evidence_messages = [
        message
        for session_id, session in zip(
            q["haystack_session_ids"], q["haystack_sessions"]
        )
        if session_id in answer_ids
        for message in session
        if message.get("has_answer") is True
    ]
```

在此基础上，结合评测问题、标准答案和证据消息的相邻轮次，对 56 道题逐题赋予一个主类。完整映射见上一节，合计应严格等于 56，且每个 `question_id` 只出现一次。

## 8. 限制

- 这里的 10 类是面向记忆系统设计的功能型分类，不是 LongMemEval 官方标签；
- 为保证统计可加和，本文采用单一主类，未展示每题可能存在的次级标签；
- 中文翻译以准确表达原意为目标，不作为新的评测答案；
- 样例只展示答案证据所在的最小相关片段，不等同于完整 haystack session；
- 本文统计的是当前文件哈希对应的 cleaned 版本，换用其他 LongMemEval 版本时应重新计算。
