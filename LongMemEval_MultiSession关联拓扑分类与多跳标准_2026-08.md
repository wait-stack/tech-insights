# LongMemEval Multi-Session 关联拓扑分类与多跳标准

> 统计对象：LongMemEval-S cleaned 数据集中的官方 `multi-session` 题型
>
> 统计日期：2026-08-21
>
> 核心问题：这些题究竟主要是“同一话题的跨会话关联”，还是需要桥接变量的真正多跳推理？

## 1. 结论摘要

LongMemEval 的 500 道题中，官方标为 `multi-session` 的有 **133 道，占 26.6%**。逐题检查问题、标准答案、`answer_session_ids`、`has_answer` 消息及相邻上下文后，本文的主结论是：

> **`multi-session` 主要表示证据分散在多个会话，并不等同于 multi-hop。**

在 133 道题中：

| 关联拓扑 | 题数 | 占全部 133 题 | 占 121 道可回答题 |
|---|---:|---:|---:|
| 同话题直接聚合、计算或比较 | 90 | 67.67% | 74.38% |
| 同锚点跨会话组合 | 28 | 21.05% | 23.14% |
| 严格桥接多跳 | 3 | 2.26% | 2.48% |
| 信息不足对照题 | 12 | 9.02% | — |
| **合计** | **133** | **100%** | **121** |

排除 12 道信息不足对照题后，**118/121（97.52%）** 都可以通过“查询同时直接召回多个证据，再做集合或算术操作”解决；只有 **3/121（2.48%）** 符合本文的严格多跳定义。

因此，对这批数据最准确的描述不是“多跳题集合”，而是：

- 主体是**同一语义槽位的并行证据聚合**；
- 其次是**同一对象或事件的分散属性组合**；
- 真正需要先解出一个桥接变量、再据此寻找下一条证据的题非常少。

## 2. 数据口径

### 2.1 数据集

- 文件：`longmemeval_s_cleaned.json`
- 记录数：500
- 文件大小：277,383,467 bytes
- SHA-256：`d6f21ea9d60a0d56f34a05b609c79c88a451d2ae03597821ea3d5a9678c3a442`
- 官方 `multi-session` 题：133，道数占全量的 26.6%
- 其中 `_abs` 信息不足对照题：12
- 其余可回答题：121

### 2.2 答案会话数量

| 每题 `answer_session_ids` 数量 | 题数 | 占比 |
|---:|---:|---:|
| 2 | 84 | 63.16% |
| 3 | 26 | 19.55% |
| 4 | 17 | 12.78% |
| 5 | 6 | 4.51% |
| **合计** | **133** | **100%** |

会话数量只能说明证据分散度，不能判断是否多跳。一个需要相加的五会话问题，仍可能是五条彼此平行的直接证据。

### 2.3 证据角色与标注

- 124 道题的 `has_answer` 证据只在 user 消息中；
- 1 道题同时涉及 user 和 assistant 证据；
- 8 道题在其答案会话中没有 `has_answer == true` 消息，均属于 `_abs` 对照题；
- 典型 `_abs` 题会提供相关但不充分的信息，用来检查系统是否会错误补全答案。

本文按官方 `question_type == "multi-session"` 统计，不把消息角色作为排除条件。

## 3. 什么才算“严格多跳”

### 3.1 核心判据：检索依赖，而不是计算步数

本文使用下面的可执行定义：

> 如果查询能够独立、直接地指向每一条必需证据，那么即使之后需要计数、求和、求差或比较，也不算严格多跳。只有当第二条证据的定位或解释依赖第一步得到的中间变量时，才算严格桥接多跳。

可以把两种结构写成：

**并行直接关联**

```text
                 ┌─> 证据 A ─┐
查询 Q ──────────┼─> 证据 B ─┼─> 聚合 / 算术 / 比较 ─> 答案
                 └─> 证据 C ─┘
```

**严格桥接多跳**

```text
查询 Q ─> 证据 A ─> 桥接变量 X ─> 证据 B ─> 答案
```

关键测试是：**在不知道证据 A 的答案前，能否仅凭原始问题稳定地定位并理解证据 B？**

- 能：通常是同话题直接关联或同锚点组合；
- 不能：证据 A 的输出成为下一次检索条件，属于严格多跳。

### 3.2 四步标注决策树

1. **答案是否可由给定会话唯一推出？**不能则归入“信息不足对照题”。
2. **问题能否直接指向每个答案证据？**能则归入“同话题直接关联”。
3. **证据是否围绕同一显式对象、事件或度量，只是把起点/终点、原价/折扣价等操作数拆开？**是则归入“同锚点跨会话组合”。
4. **是否必须先从一条证据解出实体、日期、事件或状态，再用它定位/解释另一条证据？**是则归入“严格桥接多跳”。

### 3.3 三个常见误判

**误判一：有两个会话，所以是两跳。**

不是。会话数量描述数据分布，跳数描述证据依赖关系。

**误判二：需要两次算术，所以是多跳。**

不是。例如分别回忆 MCU 和 Star Wars 的观看时长再相加，两个数字都被问题直接点名，属于并行召回加聚合。

**误判三：两个会话表面话题不同，所以是多跳。**

也不一定。判断依据不是聊天主题名称，而是问题与证据的语义路径。一次聊健身、一次聊游泳池，只要问题直接询问“参加过哪些竞技运动”，两条经历仍是同一语义槽位的直接实例。

## 4. 分类体系与统计

### 4.1 A 类：同话题直接关联，共 90 道（67.67%）

每条证据都直接填充问题要求的同一个槽位。证据之间是并行关系，不存在“先用 A 找到 B”的依赖。

| 子类 | 定义 | 题数 | 占全部题 |
|---|---|---:|---:|
| A1 枚举、去重、计数与状态筛选 | 收集多个同类事件或实体，去重、过滤后作答 | 36 | 27.07% |
| A2 数值聚合 | 对问题直接点名的多个数值求和、均值或总量 | 45 | 33.83% |
| A3 并行比较与选择 | 对多个并列对象计算变化量或比较属性 | 9 | 6.77% |
| **A 类合计** |  | **90** | **67.67%** |

典型结构：

- “我竞技性地参加过几种运动？”→ 分别召回游泳、网球 → 去重计数；
- “看完 MCU 和 Star Wars 共用了几周？”→ 2 周 + 1.5 周；
- “哪个平台涨粉最多？”→ Twitter、TikTok、Facebook 的变化量并行比较。

### 4.2 B 类：同锚点跨会话组合，共 28 道（21.05%）

多条证据围绕同一个明确锚点，但属性或状态被拆在不同会话中。常见形式包括：

- 订单日期 + 到货日期；
- 原价 + 售价；
- 起始状态 + 结束状态；
- 总任职时长 + 上一职位时长；
- 某次活动的时间 + 同一活动的另一属性。

这类题需要组合或算术推导，但问题本身通常已给出共同对象，能够直接召回各操作数，所以不按严格多跳计算。

### 4.3 C 类：严格桥接多跳，共 3 道（2.26%）

这三题存在真正的链式依赖：

| question_id | 桥接路径 | 标准答案 |
|---|---|---|
| `dd2973ad` | 医生预约在周四 → 前一天是周三 → 周三入睡时间 | 2 AM |
| `51c32626` | 情感分析论文 → 投稿 ACL → ACL 的投稿日期 | February 1st |
| `a96c20ee` | 展示论文海报 → 第一次研究会议 → 该会议在 Harvard | Harvard University |

其中 `51c32626` 还存在标注假设：原文给出的是“论文投给 ACL”和“ACL 的 submission date 是 2 月 1 日”，并没有直接说用户本人恰好在截止日提交。数据集把会议投稿日期当成了实际提交日期。

### 4.4 D 类：信息不足对照题，共 12 道（9.02%）

这些题通常提供语义接近的干扰事实，却缺少完成计算的关键操作数。例如，`2311e44b_abs` 提供了《Sapiens》的阅读速度，也提供了另一本书《The Nightingale》的当前页和总页数，但没有《Sapiens》的当前页或总页数，因此无法计算剩余页数。

## 5. 边界案例：`92a0aa75` 为什么不计入严格多跳

- 问题：`How long have I been working in my current role?`
- 会话 1：用户从 Marketing Coordinator 晋升为 Senior Marketing Specialist，前一阶段用了 2 年 4 个月；
- 会话 2：用户在该公司总共工作了 3 年 9 个月；
- 推导：3 年 9 个月 − 2 年 4 个月 = 1 年 5 个月。

它可以画成一条“职业状态链”，因此是最接近多跳的边界案例。但两个操作数都属于同一任职时间轴，问题也直接要求该时间轴上的当前阶段时长；第二条证据不需要先知道某个新实体才能被定位。因此本文把它归入 **B 类同锚点组合**。

如果采用更宽松的定义，把“跨状态的差值推导”也算多跳，那么严格多跳题会从 3 道变为 4 道：占全部题 **3.01%**，占可回答题 **3.31%**。无论采用严格还是宽松口径，都不改变“绝大多数不是多跳”的结论。

## 6. 典型题原始对话、中文翻译与答案

以下片段只保留答案相关消息；中文为本文翻译，不是数据集原字段。

### 6.1 A1：并行枚举与计数——竞技运动

- `question_id`：`ef66a6e5`
- 答案会话：`answer_f7fd1029_2`、`answer_f7fd1029_1`

**评测问题（英文）**

> How many sports have I played competitively in the past?

**问题中文翻译**

> 我过去参加过几种竞技运动？

**原始答案对话 1（英文）**

> User: I'm looking to find a local pool that offers lap swimming hours. I used to swim competitively in college, and I'm looking to get back into it as a way to stay active and relieve stress.

**中文翻译**

> 用户：我想找一个提供游泳训练时段的本地泳池。我大学时参加过竞技游泳，现在想重新开始，把它作为保持活力和缓解压力的方式。

**原始答案对话 2（英文）**

> User: I'm actually thinking of incorporating some strength training into my routine as well. Can you recommend some exercises that would be beneficial for my tennis game, considering I used to play tennis competitively in high school?

**中文翻译**

> 用户：我也在考虑把力量训练加入日常安排。考虑到我高中时参加过竞技网球，你能推荐一些对网球有帮助的训练吗？

**答案**

> `two`，即 **两种：游泳和网球**。

**拓扑判断**

问题直接要求所有“竞技运动”实例，两条证据都直接命中该槽位；这是并行枚举，不是 `游泳 → 网球` 的链式推理。

### 6.2 A2：并行数值聚合——电影马拉松

- `question_id`：`e831120c`
- 答案会话：`answer_86c505e7_1`、`answer_86c505e7_2`

**评测问题（英文）**

> How many weeks did it take me to watch all the Marvel Cinematic Universe movies and the main Star Wars films?

**问题中文翻译**

> 我看完所有漫威电影宇宙电影和《星球大战》正传电影一共用了几周？

**原始答案对话 1（英文）**

> User: [...] I watched all 22 Marvel Cinematic Universe movies in two weeks.

**中文翻译**

> 用户：[…] 我用两周看完了全部 22 部漫威电影宇宙电影。

**原始答案对话 2（英文）**

> User: [...] I just finished a Star Wars marathon, watched all the main films in a week and a half [...]

**中文翻译**

> 用户：[…] 我刚完成一次《星球大战》马拉松，用一周半看完了所有正传电影 […]

**答案**

> `3.5 weeks`，即 **3.5 周**：2 + 1.5 = 3.5。

**拓扑判断**

问题同时明确点名 MCU 和 Star Wars；两个时长可被独立直接召回。虽然需要加法，但没有桥接检索。

### 6.3 A3：并行比较——社交平台涨粉

- `question_id`：`gpt4_5501fe77`
- 答案会话：`answer_203bf3fa_1`、`answer_203bf3fa_3`、`answer_203bf3fa_2`

**评测问题（英文）**

> Which social media platform did I gain the most followers on over the past month?

**问题中文翻译**

> 过去一个月，我在哪个社交媒体平台上增加的粉丝最多？

**原始答案对话（英文）**

> User: [...] my Twitter follower count has jumped from 420 to 540 over the past month [...]
>
> User: [...] TikTok, where I've gained around 200 followers over the past three weeks [...]
>
> User: [...] my Facebook follower count has remained steady at around 800 [...]

**中文翻译**

> 用户：[…] 我的 Twitter 粉丝数在过去一个月从 420 增加到 540 […]
>
> 用户：[…] TikTok 在过去三周增加了约 200 名粉丝 […]
>
> 用户：[…] 我的 Facebook 粉丝数一直稳定在约 800 […]

**答案**

> `TikTok`。Twitter 增加 120，TikTok 增加约 200，Facebook 增加 0。

**拓扑判断**

三个平台都是问题直接要求比较的并列对象；这是并行召回、标准化变化量后取最大值。

### 6.4 B：同锚点组合——快门遥控器配送时长

- `question_id`：`b3c15d39`
- 答案会话：`answer_05d808e6_1`、`answer_05d808e6_2`

**评测问题（英文）**

> How many days did it take for me to receive the new remote shutter release after I ordered it?

**问题中文翻译**

> 我订购新的快门遥控器后，过了多少天才收到？

**原始答案对话 1（英文）**

> User: [...] I also ordered a new remote shutter release online on February 5th [...]

**中文翻译**

> 用户：[…] 我还在 2 月 5 日在线订购了一个新的快门遥控器 […]

**原始答案对话 2（英文）**

> User: [...] I just got a new remote shutter release that arrived on February 10th [...]

**中文翻译**

> 用户：[…] 我刚收到一个新的快门遥控器，它在 2 月 10 日送达 […]

**答案**

> `5 days`；数据集也接受首尾日期都计入时的 `6 days`。

**拓扑判断**

订购日和到货日都由问题中的同一显式对象“new remote shutter release”直接约束。需要日期求差，但不需要先解出另一个检索实体。

### 6.5 C：严格时间桥接——预约前一天的入睡时间

- `question_id`：`dd2973ad`
- 答案会话：`answer_f9de4602_2`、`answer_f9de4602_1`

**评测问题（英文）**

> What time did I go to bed on the day before I had a doctor's appointment?

**问题中文翻译**

> 我在看医生预约前一天几点睡觉？

**原始答案对话 1（英文）**

> User: [...] I had a doctor's appointment at 10 AM last Thursday [...]

**中文翻译**

> 用户：[…] 我上周四上午 10 点有一个医生预约 […]

**原始答案对话 2（英文）**

> User: [...] I didn't get to bed until 2 AM last Wednesday, which made Thursday morning a struggle.

**中文翻译**

> 用户：[…] 我上周三直到凌晨 2 点才睡，因此周四早上很难熬。

**答案**

> `2 AM`，即 **凌晨 2 点**。

**拓扑判断**

必须先从预约证据解出“上周四”，再做日期偏移得到桥接变量“上周三”，最后才能把另一会话中的睡眠事实与问题对齐：

```text
医生预约 → 上周四 → 前一天/上周三 → 凌晨 2 点睡觉
```

### 6.6 C：严格事件桥接——论文海报所在大学

- `question_id`：`a96c20ee`
- 答案会话：`answer_ef84b994_1`、`answer_ef84b994_2`

**评测问题（英文）**

> At which university did I present a poster on my thesis research?

**问题中文翻译**

> 我在哪所大学展示了关于论文研究的海报？

**原始答案对话 1（英文）**

> User: [...] I actually just presented a poster on my thesis research on it at my first research conference over the summer.

**中文翻译**

> 用户：[…] 去年夏天，我刚在自己的第一次研究会议上展示了一张关于论文研究的海报。

**原始答案对话 2（英文）**

> User: [...] I've been to Harvard University to attednd my first research conference [...]

**中文翻译**

> 用户：[…] 我曾去哈佛大学参加自己的第一次研究会议 […]

**答案**

> `Harvard University`，即 **哈佛大学**。

**拓扑判断**

问题中的“论文海报”不能直接指向 Harvard；必须先把它解析为“第一次研究会议”，再通过另一会话找到会议地点：

```text
论文海报 → 第一次研究会议 → Harvard University
```

### 6.7 C：严格实体桥接——情感分析论文投稿日期

- `question_id`：`51c32626`
- 答案会话：`answer_58820c75_1`、`answer_58820c75_2`

**评测问题（英文）**

> When did I submit my research paper on sentiment analysis?

**问题中文翻译**

> 我什么时候提交了关于情感分析的研究论文？

**原始答案对话 1（英文）**

> User: [...] I even worked on a research paper on sentiment analysis, which I submitted to ACL.

**中文翻译**

> 用户：[…] 我还做过一篇关于情感分析的研究论文，并把它投给了 ACL。

**原始答案对话 2（英文）**

> User: I'm reviewing for ACL, and their submission date was February 1st. [...]

**中文翻译**

> 用户：我正在为 ACL 做审稿，它的投稿日期是 2 月 1 日。[…]

**标准答案**

> `February 1st`，即 **2 月 1 日**。

**拓扑判断与标注风险**

推理路径是 `情感分析论文 → ACL → 2 月 1 日`，属于实体桥接多跳。但“ACL 的 submission date”更像会议日期或截止日期，不严格等于用户实际提交论文的日期；因此这题同时是一个标注语义偏强的案例。

### 6.8 D：信息不足——《Sapiens》剩余页数

- `question_id`：`2311e44b_abs`
- 答案会话：`answer_bf633415_abs_1`、`answer_bf633415_abs_2`

**评测问题（英文）**

> How many pages do I have left to read in 'Sapiens'?

**问题中文翻译**

> 《Sapiens》我还剩多少页没读？

**原始相关对话（英文）**

> User: I'm currently on page 250 of 'The Nightingale' by Kristin Hannah [...]
>
> User: [...] I just got back to reading "The Nightingale" [...] it's a long one with 440 pages!
>
> User: [...] I've been reading "Sapiens" at a pace of 10-20 pages a week.

**中文翻译**

> 用户：我目前读到 Kristin Hannah 的《The Nightingale》第 250 页 […]
>
> 用户：[…] 我刚重新开始读《The Nightingale》[…] 这本书很长，有 440 页！
>
> 用户：[…] 我读《Sapiens》的速度是每周 10 到 20 页。

**答案**

> `The information provided is not enough.`，即 **信息不足，无法计算**。

**拓扑判断**

`440 - 250` 是《The Nightingale》的剩余页数，不能迁移到《Sapiens》；阅读速度也不能推出当前页或总页数。正确行为是识别缺失证据，而不是把相邻实体的数字拼成答案。

## 7. 全部 133 道题的分类映射

每道官方 `multi-session` 题恰好出现一次；映射已做去重和全集校验。

### 7.1 A1 枚举、去重、计数与状态筛选：36 道

`0a995998`、`6d550036`、`gpt4_59c863d7`、`3a704032`、`gpt4_f2262a51`、`c4a1ceb8`、`gpt4_a56e767c`、`46a3abf7`、`gpt4_2f8be40d`、`2e6d26dc`、`gpt4_15e38248`、`88432d0a`、`80ec1f4f`、`d23cf73b`、`gpt4_7fce9456`、`d682f1a2`、`2ce6a0f2`、`00ca467f`、`gpt4_31ff4165`、`9d25d4e0`、`60472f9c`、`gpt4_194be4b3`、`a9f6b44c`、`5a7937c8`、`gpt4_ab202e7f`、`1a8a66a6`、`bf659f65`、`81507db6`、`4f54b7c9`、`681a1674`、`ef66a6e5`、`5025383b`、`60159905`、`a08a253f`、`8e91e7d9`、`21d02d0d`。

### 7.2 A2 数值聚合：45 道

`b5ef892d`、`e831120c`、`gpt4_d84a3211`、`aae3761f`、`6cb6f249`、`36b9f61e`、`28dc39ac`、`7024f17c`、`gpt4_d12ceb0e`、`eeda8a6d`、`2788b940`、`129d1232`、`d851d5ba`、`gpt4_e05b82a6`、`gpt4_731e37d7`、`edced276`、`10d9b85a`、`e3038f8c`、`2b8f3739`、`c2ac3c61`、`gpt4_372c3eed`、`gpt4_2f91af09`、`d3ab962e`、`85fa3a3f`、`f35224e0`、`6456829e`、`60036106`、`4adc0475`、`4bc144e2`、`3fdac837`、`91b15a6e`、`720133ac`、`8979f9ec`、`1c549ce4`、`6c49646a`、`1192316e`、`67e0d0f2`、`ef9cf60a`、`bc149d6b`、`d6062bb9`、`a3332713`、`55241a1f`、`f0e564bc`、`8cf4d046`、`37f165cf`。

### 7.3 A3 并行比较与选择：9 道

`gpt4_5501fe77`、`gpt4_2ba83207`、`2318644b`、`1f2b8d4f`、`7405e8b1`、`3c1045c8`、`a1cc6108`、`27016adc`、`157a136e`。

### 7.4 B 同锚点跨会话组合：28 道

`b3c15d39`、`60bf93ed`、`2311e44b`、`cc06de0d`、`a11281a2`、`9aaed6a3`、`e6041065`、`d905b33f`、`a4996e51`、`e25c3b8d`、`9ee3ecd6`、`77eafa52`、`0100672e`、`92a0aa75`、`3fe836c9`、`0ea62687`、`bb7c3b45`、`ba358f49`、`61f8c8f8`、`73d42213`、`099778bb`、`09ba9854`、`c18a7dc8`、`078150f1`、`a346bb18`、`87f22b4a`、`e56a43b9`、`efc3f7c2`。

### 7.5 C 严格桥接多跳：3 道

`dd2973ad`、`51c32626`、`a96c20ee`。

### 7.6 D 信息不足对照题：12 道

`88432d0a_abs`、`80ec1f4f_abs`、`eeda8a6d_abs`、`60bf93ed_abs`、`edced276_abs`、`gpt4_372c3eed_abs`、`2311e44b_abs`、`6456829e_abs`、`e5ba910e_abs`、`a96c20ee_abs`、`ba358f49_abs`、`09ba9854_abs`。

## 8. 对记忆系统设计的启示

### 8.1 先优化并行覆盖率，再优化复杂图推理

因为 74.38% 的可回答题属于同话题直接关联，系统首先需要保证：

- 能召回同一槽位下分散在多个会话的全部实例；
- 不因 top-k 被单一会话的相似片段占满而漏掉其他会话；
- 支持按会话、实体和时间做结果多样化；
- 聚合前能够去重，避免同一事实的重复表述被重复计数。

### 8.2 为同锚点组合保存“对象—属性—时间”结构

B 类题需要把同一对象的不同属性联合召回。建议记忆至少保留：

```text
anchor_entity / event
attribute_or_state
value
event_time
source_session
```

例如不要只保存“2 月 5 日订购”和“2 月 10 日到货”两个无主语日期；应保留它们都属于同一个 `remote shutter release` 订单。

### 8.3 仅在检测到桥接变量时启用多跳扩展

严格多跳只有 2.48% 的可回答题。对所有题默认展开知识图谱或多轮检索，可能提高延迟、成本和噪声。更合理的触发条件是问题包含：

- 相对时间：`the day before`、`after`、`when I first...`；
- 间接地点或组织：某作品所在的活动、某论文投稿的会议；
- 指代性事件：`that conference`、`the first event`；
- 第一跳结果能形成新的实体、时间或状态检索键。

### 8.4 把“证据不足”作为一等能力

12 道 `_abs` 题专门测试错误拼接。系统不仅要召回相似事实，还要检查：

- 计算所需操作数是否齐全；
- 数字是否属于同一实体；
- 日期是实际事件日期还是截止日期；
- 结论是否唯一，而不是依赖未经表达的常识假设。

## 9. 局限与复现说明

- 本文分类是对 LongMemEval 官方 `multi-session` 标签的二次人工拓扑编码，不是数据集自带字段；
- 每题只保留一个主标签，便于统计，现实中一题可能同时具有时间、算术和比较属性；
- “多跳”存在宽严口径差异，本文用“下一步检索依赖上一步输出”的严格定义，并单独报告边界案例；
- `51c32626` 的标准答案依赖“会议 submission date 等于个人实际提交日”的假设，统计保留官方答案，但明确标注风险；
- 8 道 `_abs` 题在答案会话中没有 `has_answer` 标记，本文结合问题、标准答案和相关会话人工判断；
- 原始对话样例中的 `[…]` 表示省略与答案无关的内容，中文翻译由本文补充。

## 10. 最终判断

如果只回答“同一个话题关联，还是多跳关联”：

> **LongMemEval 的 `multi-session` 题绝大多数是同话题或同锚点的跨会话关联，不是严格多跳。**

按可回答题计，约 **97.52%** 可以归入同话题直接关联或同锚点组合；严格桥接多跳约 **2.48%**。这个分布意味着，提高 LongMemEval multi-session 表现的第一优先级应是**跨会话召回覆盖、多样化、去重和正确聚合**，而不是一开始就把主要资源投入复杂的通用多跳图推理。
