# LongMemEval Temporal-Reasoning 题型分类与时间推理标准

> 统计对象：LongMemEval-S cleaned 数据集中的官方 `temporal-reasoning` 题型
>
> 统计日期：2026-08-21
>
> 核心问题：Temporal 题具体在做哪些时间操作，哪些是计算、排序、时间反查或信息不足？

## 1. 结论摘要

LongMemEval 的 500 道题中，官方标为 `temporal-reasoning` 的有 **133 道，占 26.6%**。逐题检查问题、标准答案、`question_date`、`answer_session_ids`、会话日期和 `has_answer` 消息后，本文将其编码为 6 个互斥主类、10 个细分类别。

| 主类 | 题数 | 占全部 133 题 | 占 127 道可回答题 |
|---|---:|---:|---:|
| T1 查询时点回溯 | 21 | 15.79% | 16.54% |
| T2 事件间隔、持续时间与年龄 | 41 | 30.83% | 32.28% |
| T3 先后排序与最近性 | 39 | 29.32% | 30.71% |
| T4 按时间或序位反查事实 | 24 | 18.05% | 18.90% |
| T5 时间窗口计数与众数 | 2 | 1.50% | 1.57% |
| T6 信息不足对照题 | 6 | 4.51% | — |
| **合计** | **133** | **100%** | **127** |

主要发现：

- 只有 **62/127（48.82%）** 的可回答题以时间差、持续时间或年龄数值为主要答案；
- 另外 **65/127（51.18%）** 主要做事件排序、时间条件选择或窗口聚合，并不以“算出几天”为目标；
- **20/133（15.04%）** 只标注了 1 个答案会话，所以 `temporal-reasoning` 不等于跨会话题；
- 33 道题的标准答案明确同时接受“普通日期差”和“包含最后一天”的计数口径；
- 至少有一个明显的标准答案文本错误：`f0853d11` 的两事件相差 14 天，包含首尾应为 15 天，但答案写成了 8 天。

因此，这类题的核心不是单一的日期减法，而是一个完整的时间处理链：

```text
时间表达识别 → 事件时间归一化 → 实体/事件对齐 → 时间操作 → 完整性检查 → 答案
```

## 2. 数据口径

### 2.1 数据集

- 文件：`longmemeval_s_cleaned.json`
- 记录数：500
- 文件大小：277,383,467 bytes
- SHA-256：`d6f21ea9d60a0d56f34a05b609c79c88a451d2ae03597821ea3d5a9678c3a442`
- 官方 `temporal-reasoning` 题：133
- `_abs` 信息不足对照题：6
- 可回答题：127

本文按官方 `question_type == "temporal-reasoning"` 统计。分类对象是**回答问题所需的主要时间操作**，而不是答案表面的词性。

### 2.2 答案会话数量

| 每题 `answer_session_ids` 数量 | 题数 | 占比 |
|---:|---:|---:|
| 1 | 20 | 15.04% |
| 2 | 88 | 66.17% |
| 3 | 15 | 11.28% |
| 4 | 2 | 1.50% |
| 5 | 5 | 3.76% |
| 6 | 3 | 2.26% |
| **合计** | **133** | **100%** |

132 道题的答案证据标在 user 消息中；`gpt4_93159ced_abs` 的答案会话没有 `has_answer == true` 消息。

### 2.3 三种必须区分的时间

Temporal 题不能只保存一个笼统的 `date`。至少需要区分：

1. **会话/观察时间 `observation_time`**：数据集中的 `haystack_dates`，用于解释消息里的 `today`、`last Tuesday` 等；
2. **事件时间 `event_time`**：用户实际参加活动、购买物品、开始或结束任务的时间；
3. **查询时间 `query_time`**：`question_date`，用于回答“多少天前”“几个月前”。

例如某消息在 4 月 6 日说“我今天参加了礼拜”，问题在 4 月 10 日问“多少天前”，正确路径是：

```text
observation_time 2023-04-06
  + “today”
  → event_time 2023-04-06
  → query_time 2023-04-10 与 event_time 求差
  → 4 days ago
```

## 3. 分类标准

### 3.1 主标签决策树

每题只保留一个主标签，按以下顺序判断：

1. **关键事件或时间是否缺失？**缺失且无法唯一作答，归入 T6 信息不足。
2. **问题是否要求在时间窗口内计数或求出现次数最多的对象？**是则归入 T5。
3. **问题是否要求比较候选事件的先后、输出完整顺序或选择最近者？**是则归入 T3。
4. **时间条件是否主要用于定位某个事件，再回答人物、地点、物品、活动、日期或时刻？**是则归入 T4。
5. **答案是否是事件相对 `question_date` 已过去多久？**是则归入 T1。
6. **答案是否是两个事件的间隔、一个状态的持续时间、年龄或多个时段之和？**是则归入 T2。

### 3.2 两个容易混淆的边界

**“Which happened first?” 与 “What was the first issue?”**

- 如果问题列出候选并要求比较哪个先发生，归 T3；
- 如果“first”只是时间条件，用来定位一个事件并返回它的属性，归 T4。

**“How many days ago?” 与 “How many days between?”**

- 事件到 `question_date` 的距离归 T1；
- 两个历史事件之间的距离归 T2。

## 4. 六类题型详解

### 4.1 T1 查询时点回溯：21 道（15.79%）

问题询问某事件距 `question_date` 已过去多少天、周或月。常见表达包括：

- `How many days ago...`
- `How many weeks ago...`
- `How many months have passed since...`

这类题中有 16 道只需要一个答案会话。它们的挑战主要不是跨会话召回，而是正确保存会话时间，并将 `today`、`last Thursday` 等相对表达归一化。

### 4.2 T2 事件间隔、持续时间与年龄：41 道（30.83%）

| 子类 | 定义 | 题数 | 占全部题 |
|---|---|---:|---:|
| T2a 两个点事件的时间差 | 两个购买、活动、开始/结束事件之间求差 | 27 | 20.30% |
| T2b 状态持续、任职时长或年龄 | 从状态起点到观察点，或出生/当前年龄到迁移事件 | 12 | 9.02% |
| T2c 多个独立时段累计 | 多本书、多个活动的持续时间求和 | 2 | 1.50% |
| **T2 合计** |  | **41** | **30.83%** |

T2a 和 T2b 都可能使用减法，但语义不同：前者比较两个点事件，后者回答一个状态保持了多久。T2c 则必须先分别计算或召回每个区间，再求总和。

### 4.3 T3 先后排序与最近性：39 道（29.32%）

| 子类 | 定义 | 题数 | 占全部题 |
|---|---|---:|---:|
| T3a 两两先后、第一或最近选择 | 从两个或多个候选中选更早/更晚/最近者 | 30 | 22.56% |
| T3b 三个及以上事件的完整排序 | 输出一条完整时间序列 | 9 | 6.77% |
| **T3 合计** |  | **39** | **29.32%** |

这类题通常不需要输出时间差，只需要把事件映射到可比较时间后排序。系统必须避免把“会话在召回结果中的先后”误当成“事件发生先后”。

### 4.4 T4 按时间或序位反查事实：24 道（18.05%）

| 子类 | 定义 | 题数 | 占全部题 |
|---|---|---:|---:|
| T4a 相对/绝对时间锚点反查 | 给定 `last Tuesday`、`two weeks ago`、Valentine's Day 等，反查人物、地点或事件 | 21 | 15.79% |
| T4b 时间规则或序位约束反查 | 根据“第一次”“每周二/周四”等规则计算并返回属性、日期或时刻 | 3 | 2.26% |
| **T4 合计** |  | **24** | **18.05%** |

T4 与普通事实问答的区别是：同一实体可能有多次相似事件，时间条件是消歧和选择答案的关键，而不是附属元数据。

### 4.5 T5 时间窗口计数与众数：2 道（1.50%）

这类题先按时间边界筛选事件，再做集合聚合：

- `a3838d2b`：统计 `Run for the Cure` 之前参加过多少场慈善活动；
- `gpt4_9a159967`：统计 3 月和 4 月乘坐次数最多的航空公司。

它要求事件级去重和次数展开。例如“一次往返、每程两段”应转换成四次航段，不能只把整段文本计为一次。

### 4.6 T6 信息不足对照题：6 道（4.51%）

6 道 `_abs` 题都用相邻实体制造干扰：

- 提到修围栏，但没有购买奶牛；
- 提到 NovaTech 的工作经历，但没有在 Google 任职；
- 提到 San Francisco 的 Airbnb，但问题问 Sacramento；
- 提到 iPhone，但问题问 iPad；
- 提到 Ferrari 和 Japanese Zero 模型，但没有 Porsche；
- 提到 Alex 成为父母，但没有 Tom 的信息。

正确行为是检查问题要求的每个实体和时间操作数是否存在，而不是把其他实体的日期迁移过来。

## 5. 典型题原始对话、中文翻译与答案

以下片段只保留与答案相关的原始消息；`[…]` 表示省略无关内容，中文为本文翻译，不是数据集原字段。

### 5.1 T1：查询时点回溯——Maundy Thursday 礼拜

- `question_id`：`gpt4_b5700ca9`
- `question_date`：2023/04/10 10:28
- 答案会话：`answer_a17423e7_1`，2023/04/06 22:22

**评测问题（英文）**

> How many days ago did I attend the Maundy Thursday service at the Episcopal Church?

**问题中文翻译**

> 我多少天前在圣公会教堂参加了濯足星期四礼拜？

**原始答案对话（英文）**

> User: [...] I'm glad I got to attend the Maundy Thursday service at the Episcopal Church today, it was a beautiful and moving experience.

**中文翻译**

> 用户：[…] 我很高兴今天在圣公会教堂参加了濯足星期四礼拜，那是一次美好而感人的体验。

**答案与计算**

> `4 days`。消息中的 `today` 由会话日期归一化为 4 月 6 日；问题日期是 4 月 10 日，相差 4 天。

### 5.2 T2a：两个点事件求差——两次博物馆活动

- `question_id`：`gpt4_59149c77`
- 答案会话：`answer_d00ba6d0_1`、`answer_d00ba6d0_2`

**评测问题（英文）**

> How many days passed between my visit to the Museum of Modern Art (MoMA) and the 'Ancient Civilizations' exhibit at the Metropolitan Museum of Art?

**问题中文翻译**

> 我参观现代艺术博物馆和参加大都会艺术博物馆“古代文明”展之间相隔多少天？

**原始答案对话 1（英文）**

> User: I just got back from a guided tour at the Museum of Modern Art focused on 20th-century modern art movements [...]

**中文翻译**

> 用户：我刚参加完现代艺术博物馆的一次导览，主题是二十世纪现代艺术运动 […]

会话日期：2023/01/08。

**原始答案对话 2（英文）**

> User: [...] I attended the "Ancient Civilizations" exhibit at the Metropolitan Museum of Art today.

**中文翻译**

> 用户：[…] 我今天参加了大都会艺术博物馆的“古代文明”展览。

会话日期：2023/01/15。

**答案与计算**

> `7 days`；若把最后一天计入，数据集也接受 `8 days`。

### 5.3 T2b：年龄推导——移居美国时的年龄

- `question_id`：`d01c6aa8`
- 答案会话：`answer_991d55e5_1`、`answer_991d55e5_2`

**评测问题（英文）**

> How old was I when I moved to the United States?

**问题中文翻译**

> 我移居美国时多少岁？

**原始答案对话 1（英文）**

> User: I'm 32-year-old male, and I'm trying to get a better understanding of the green card application process. [...]

**中文翻译**

> 用户：我是一名 32 岁男性，正在进一步了解绿卡申请流程。[…]

**原始答案对话 2（英文）**

> User: I've been living in the United States for the past five years on a work visa [...]

**中文翻译**

> 用户：过去五年里，我一直持工作签证居住在美国 […]

**答案与计算**

> `27`，即 32 − 5 = **27 岁**。

### 5.4 T2c：多个时段累计——三本书

- `question_id`：`gpt4_a1b77f9c`
- 答案会话：6 个

**评测问题（英文）**

> How many weeks in total do I spent on reading 'The Nightingale' and listening to 'Sapiens: A Brief History of Humankind' and 'The Power'?

**问题中文翻译**

> 我阅读《The Nightingale》以及收听《Sapiens》和《The Power》总共用了多少周？

**原始答案对话（英文）**

> User, 2022/01/01: I started reading 'The Nightingale' by Kristin Hannah today [...]
>
> User, 2022/01/15: I just finished reading "The Nightingale" by Kristin Hannah today [...]
>
> User, 2022/02/01: I just started listening to 'Sapiens: A Brief History of Humankind' [...] today [...]
>
> User, 2022/03/01: I just finished listening to 'Sapiens: A Brief History of Humankind' [...] today [...]
>
> User, 2022/03/06: I started listening to "The Power" by Naomi Alderman today [...]
>
> User, 2022/03/20: I just finished listening to 'The Power' by Naomi Alderman today [...]

**中文翻译**

> 用户分别在 1 月 1 日开始、1 月 15 日读完《The Nightingale》；2 月 1 日开始、3 月 1 日听完《Sapiens》；3 月 6 日开始、3 月 20 日听完《The Power》。

**答案与计算**

> 《The Nightingale》2 周 +《Sapiens》4 周 +《The Power》2 周 = **8 周**。

### 5.5 T3a：两事件先后比较——订婚派对与婚礼

- `question_id`：`gpt4_4929293a`
- 答案会话：`answer_add9b012_2`、`answer_add9b012_1`

**评测问题（英文）**

> Which event happened first, my cousin's wedding or Michael's engagement party?

**问题中文翻译**

> 我表亲的婚礼和 Michael 的订婚派对，哪一个先发生？

**原始答案对话 1（英文）**

> User, 2023/05/06: [...] I just came back from Michael's engagement party at a trendy rooftop bar today [...]

**中文翻译**

> 用户，2023/05/06：[…] 我今天刚参加完 Michael 在一家时尚屋顶酒吧举办的订婚派对 […]

**原始答案对话 2（英文）**

> User, 2023/06/15: [...] I just walked down the aisle as a bridesmaid at my cousin's wedding today [...]

**中文翻译**

> 用户，2023/06/15：[…] 我今天刚在表亲的婚礼上担任伴娘走过红毯 […]

**答案**

> `Michael's engagement party`，即 Michael 的订婚派对先发生。

### 5.6 T3b：多事件完整排序——三次送礼相关活动

- `question_id`：`gpt4_f49edff3`
- 答案会话：`answer_3e9fce53_1`、`answer_3e9fce53_2`、`answer_3e9fce53_3`

**评测问题（英文）**

> Which three events happened in the order from first to last: the day I helped my friend prepare the nursery, the day I helped my cousin pick out stuff for her baby shower, and the day I ordered a customized phone case for my friend's birthday?

**问题中文翻译**

> 请按先后顺序排列三件事：帮朋友布置育儿房、帮表亲挑选婴儿派对用品、为朋友生日订购定制手机壳。

**原始答案对话（英文）**

> User, 2023/02/05: [...] I just helped my friend prepare a nursery today [...]
>
> User, 2023/02/10: [...] I just helped my cousin pick out some stuff for her baby shower [...]
>
> User, 2023/02/20: [...] I just ordered a customized phone case for my friend's birthday today [...]

**中文翻译**

> 用户在 2 月 5 日帮朋友布置育儿房，2 月 10 日帮表亲挑选婴儿派对用品，2 月 20 日订购定制手机壳。

**答案**

> 育儿房 → 婴儿派对用品 → 定制手机壳。

### 5.7 T4a：相对时间反查人物——上周二午餐

- `question_id`：`gpt4_468eb064`
- `question_date`：2023/04/18，星期二
- 答案会话：`answer_9b09d95b_1`，2023/04/11，星期二

**评测问题（英文）**

> Who did I meet with during the lunch last Tuesday?

**问题中文翻译**

> 上周二午餐时我见了谁？

**原始答案对话（英文）**

> User: [...] I catch up with Emma, a freelance writer, over lunch today and she's now a potential collaborator for a project I'm working on.

**中文翻译**

> 用户：[…] 我今天午餐时和自由撰稿人 Emma 叙了旧，她现在可能成为我正在进行的项目的合作伙伴。

**答案**

> `Emma`。问题中的“上周二”对应 4 月 11 日，匹配该会话的 `today`。

### 5.8 T4b：周期规则计算——星期二和星期四的起床时间

- `question_id`：`gpt4_2c50253f`
- 答案会话：`answer_9af4e346_1`、`answer_9af4e346_2`

**评测问题（英文）**

> What time do I wake up on Tuesdays and Thursdays?

**问题中文翻译**

> 我星期二和星期四几点起床？

**原始答案对话 1（英文）**

> User: I've recently started waking up at 7:00 AM [...]

**中文翻译**

> 用户：我最近开始在早上 7 点起床 […]

**原始答案对话 2（英文）**

> User: On Tuesdays and Thursdays, I've also started waking up 15 minutes earlier to meditate and practice some yoga poses [...]

**中文翻译**

> 用户：星期二和星期四，我还会提前 15 分钟起床，用来冥想和练习瑜伽 […]

**答案与计算**

> `6:45 AM`，即 7:00 − 15 分钟。

### 5.9 T5：时间窗口众数——3 月和 4 月乘坐最多的航空公司

- `question_id`：`gpt4_9a159967`
- 答案会话：`answer_8a42fedf_1`、`answer_8a42fedf_3`、`answer_8a42fedf_2`

**评测问题（英文）**

> Which airline did I fly with the most in March and April?

**问题中文翻译**

> 3 月和 4 月，我乘坐次数最多的是哪家航空公司？

**原始答案对话（英文）**

> User: In March, I took a business trip to Chicago with United Airlines, flying from my hometown to Chicago on the 10th and returning on the 12th, with two flights each way.
>
> User: I'm planning a trip to San Francisco next month and I'm considering flying with Southwest Airlines. I've had a good experience with them before, like when I took a direct flight from my hometown to Las Vegas for a conference in March, from the 15th to the 18th.
>
> User: [...] I took at least 10 Uber rides during my week-long vacation to Hawaii with my family from the 20th to the 27th of April. We flew with American Airlines from our hometown to Honolulu, and then took a connecting flight to Maui.

**中文翻译**

> 用户：3 月乘 United 往返 Chicago，每程两段，共四个航段；3 月乘 Southwest 直飞 Las Vegas；4 月乘 American 到 Honolulu，再转机到 Maui。

**答案**

> `United Airlines`。需要先把复合行程展开成航段，再在 3—4 月窗口内计数。

### 5.10 T6：信息不足——Sacramento 的 Airbnb

- `question_id`：`982b5123_abs`
- 答案会话：`answer_ab603dd5_abs_1`、`answer_ab603dd5_abs_2`

**评测问题（英文）**

> When did I book the Airbnb in Sacramento?

**问题中文翻译**

> 我什么时候预订了 Sacramento 的 Airbnb？

**原始相关对话（英文）**

> User: [...] when I stayed in Haight-Ashbury for my best friend's wedding and had to book three months in advance.
>
> User: [...] I've been to SF before, exactly two months ago, for my best friend's wedding [...]

**中文翻译**

> 用户：[…] 我去 Haight-Ashbury 参加好友婚礼时，必须提前三个月预订。
>
> 用户：[…] 我两个月前曾去 San Francisco 参加好友婚礼 […]

**答案**

> `The information provided is not enough.`，即 **信息不足**。证据只涉及 San Francisco，没有 Sacramento。

## 6. 全部 133 道题的分类映射

每道官方 `temporal-reasoning` 题恰好出现一次；映射已做全集、唯一性和重复检查。

### 6.1 T1 查询时点回溯：21 道

`71017276`、`b46e15ed`、`0bc8ad92`、`af082822`、`gpt4_b5700ca9`、`9a707b81`、`gpt4_e072b769`、`gpt4_6dc9b45b`、`gpt4_8279ba02`、`gpt4_468eb063`、`8077ef71`、`bcbe585f`、`5e1b23de`、`gpt4_af6db32f`、`eac54adc`、`gpt4_7ddcf75f`、`gpt4_a2d1d1f6`、`gpt4_85da3956`、`gpt4_b0863698`、`gpt4_7bc6cf22`、`982b5123`。

### 6.2 T2a 两个点事件的时间差：27 道

`gpt4_59149c77`、`gpt4_fa19884c`、`gpt4_1d4ab0c9`、`0db4c65d`、`gpt4_1916e0ea`、`gpt4_7a0daae1`、`gpt4_1e4a8aeb`、`gpt4_4fc4f797`、`4dfccbf7`、`gpt4_61e13b3c`、`370a8ff4`、`gpt4_4ef30696`、`gpt4_8e165409`、`gpt4_74aed68e`、`gpt4_21adecb5`、`gpt4_e414231e`、`0bb5a684`、`08f4fc43`、`2c63a862`、`2a1811e2`、`bbf86515`、`f0853d11`、`c8090214`、`dcfa8644`、`a3045048`、`6613b389`、`8c18457d`。

### 6.3 T2b 状态持续、任职时长或年龄：12 道

`gpt4_1d80365e`、`2ebe6c90`、`6e984301`、`gpt4_93159ced`、`e4e14d04`、`c9f37c46`、`cc6d1ec1`、`d01c6aa8`、`993da5e2`、`gpt4_cd90e484`、`gpt4_4cd9eba1`、`b29f3365`。

### 6.4 T2c 多个独立时段累计：2 道

`gpt4_a1b77f9c`、`b9cfe692`。

### 6.5 T3a 两两先后、第一或最近选择：30 道

`gpt4_4929293a`、`gpt4_ec93e27f`、`gpt4_98f46fc6`、`gpt4_68e94287`、`gpt4_2487a7cb`、`gpt4_76048e76`、`gpt4_2312f94c`、`gpt4_385a5000`、`gpt4_0b2f1d21`、`gpt4_6ed717ea`、`gpt4_70e84552`、`gpt4_2d58bcd6`、`gpt4_65aabe59`、`gpt4_483dd43c`、`gpt4_b4a80587`、`gpt4_8c8961ae`、`gpt4_d9af6064`、`gpt4_7de946e7`、`gpt4_d31cdae3`、`gpt4_88806d6e`、`gpt4_93f6379c`、`gpt4_2f56ae70`、`gpt4_78cf46a3`、`gpt4_0a05b494`、`gpt4_1a1dc16d`、`gpt4_2f584639`、`gpt4_213fd887`、`gpt4_5438fa52`、`gpt4_c27434e8`、`gpt4_fe651585`。

### 6.6 T3b 三个及以上事件的完整排序：9 道

`gpt4_f49edff3`、`gpt4_7f6b06db`、`gpt4_18c2b244`、`gpt4_7abb270c`、`gpt4_45189cb4`、`gpt4_e061b84f`、`gpt4_d6585ce8`、`gpt4_f420262c`、`gpt4_7ca326fa`。

### 6.7 T4a 相对或绝对时间锚点反查：21 道

`2ebe6c92`、`gpt4_e061b84g`、`71017277`、`b46e15ee`、`gpt4_d6585ce9`、`gpt4_1e4a8aec`、`gpt4_f420262d`、`gpt4_59149c78`、`gpt4_e414231f`、`gpt4_4929293b`、`gpt4_468eb064`、`gpt4_fa19884d`、`9a707b82`、`eac54add`、`4dfccbf8`、`0bc8ad93`、`6e984302`、`gpt4_8279ba03`、`gpt4_b5700ca0`、`gpt4_68e94288`、`gpt4_5dcc0aab`。

### 6.8 T4b 时间规则或序位约束反查：3 道

`gpt4_2655b836`、`gpt4_4edbafa2`、`gpt4_2c50253f`。

### 6.9 T5 时间窗口计数与众数：2 道

`a3838d2b`、`gpt4_9a159967`。

### 6.10 T6 信息不足对照题：6 道

`gpt4_70e84552_abs`、`gpt4_93159ced_abs`、`982b5123_abs`、`c8090214_abs`、`gpt4_c27434e8_abs`、`gpt4_fe651585_abs`。

## 7. 标注与评测风险

### 7.1 日期差存在两种计数口径

36 道问题明确询问 `How many days...`，其中 33 道标准答案接受普通日期差和“包含最后一天”的替代答案。系统输出最好保留主答案和口径说明，而不是只输出一个没有解释的数字。

### 7.2 `f0853d11` 的替代答案明显错误

- `Walk for Hunger`：2023/02/21；
- `Coastal Cleanup`：2023/03/07；
- 普通日期差：14 天；
- 若首尾都计入：15 天；
- 数据集答案：`14 days. 8 days (including the last day) is also acceptable.`

这里的 `8 days` 与证据不一致，应视为答案文本缺陷，不能拿它校准日期算法。

### 7.3 会话日期不一定等于事件日期

当消息说 `today` 时，会话日期可作为事件日期；但消息也可能明确说 `on February 10th`、`last Friday` 或 `two months ago`。时间归一化应优先解析文本表达，再使用会话日期作为锚点，不能无条件把会话日期复制成事件日期。

### 7.4 排序依赖事件时间，不依赖记忆写入顺序

Haystack 的排列、检索排名、摄取时间和事件时间是不同维度。排序题只能比较归一化后的 `event_time`。

## 8. 对记忆系统设计的启示

### 8.1 时间字段应结构化保存

建议每条事件记忆至少保留：

```text
event_id
entity / action / state
time_expression_original
event_time_start
event_time_end
time_granularity
recurrence_rule
observation_time
source_session_id
normalization_confidence
```

只把日期拼进自然语言 memory text，会使相对时间、时间范围和周期规则难以稳定计算。

### 8.2 查询前先生成时间约束

对于 `last Tuesday`、`two weeks ago`、`in March and April` 等问题，应先把它们相对于 `question_date` 归一化成查询区间，再用该区间参与召回和重排。

### 8.3 聚合前做实体与事件对齐

日期计算必须建立在正确实体上。`Sacramento` 不能复用 `San Francisco` 的 Airbnb 日期，`iPad` 不能复用 `iPhone` 的购买日期；同一事件的重复描述也不能被重复计数。

### 8.4 按操作类型路由答案器

可以将 temporal query 路由到不同确定性算子：

| 操作 | 推荐执行方式 |
|---|---|
| `QUERY_MINUS_EVENT` | `question_date - event_time` |
| `EVENT_MINUS_EVENT` | 两个归一化事件日期求差 |
| `DURATION` | 状态起止点或年龄差 |
| `SORT` | 按 `event_time` 排序并返回实体 |
| `TIME_FILTER` | 先按区间筛选，再返回事件属性 |
| `COUNT / MODE` | 时间过滤后去重、展开次数并聚合 |

LLM 负责解析问题与证据，日期求差、排序和计数尽量交给确定性代码，可以降低单位换算和包含边界错误。

### 8.5 回答前执行完整性检查

至少验证：

- 每个被比较的实体是否都有事件时间；
- 持续时间是否有起点和终点；
- 时间窗口是否明确；
- 相对表达是否有可用锚点；
- 计数对象是否已经去重；
- 答案是否依赖错误实体或未来信息。

## 9. 局限与复现说明

- 本文分类是对官方 `temporal-reasoning` 标签的二次人工编码，不是数据集自带字段；
- 每题只保留一个主标签，部分题同时包含排序、过滤和算术，本文按最终主要操作分类；
- `answer_session_ids` 是标注证据范围，不等同于实际系统最少需要召回的会话数；
- 相对日期以 `question_date`、`haystack_dates` 和消息内时间表达联合解释；
- 原始对话样例中的 `[…]` 表示省略与答案无关的内容，中文翻译由本文补充；
- 本文保留官方参考答案，同时单独标记已发现的答案歧义或错误。

## 10. 最终判断

LongMemEval 的 Temporal 题可以概括为：

> **约三分之一是事件间隔或持续时间，约三分之一是先后排序，剩余主要是查询时点回溯和按时间反查事实。**

它测试的不是单一“时间算术”，而是时间表达归一化、事件身份对齐、区间计算、排序、过滤、聚合和证据不足识别的组合能力。对记忆系统而言，最关键的基础能力是把**事件发生时间**与**会话观察时间**、**查询时间**明确分离。
