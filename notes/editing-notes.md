# 整理记录

这里集中保存整理说明、转写解释、原稿与正文来源，以及 Agent 补充。

## 第一章

- 记录日期：2026-11-08（Asia/Taipei）
- 正文：[第一章](../book/chapter1.md#思考题)
- 正文版本：`dbc046eb896ac4e39aa19c7774c8bf49583b89a6`
- 完整输入：[原始转写](chapter-01/raw/2026-11-08-input.raw.txt)

### 第 1 题 · 能力选择与任务瓶颈

- 原稿：[2026-11-08-q01-capability-bottleneck.raw.txt](chapter-01/raw/2026-11-08-q01-capability-bottleneck.raw.txt)

**整理说明**

保留了“个人助手优先上下文”和“不限定条件时偏向工具”两层判断，没有替用户合并成单一选项。

### 第 2 题 · 压缩、外置与子 Agent 隔离

- 原稿：[2026-11-08-q02-context-compression.raw.txt](chapter-01/raw/2026-11-08-q02-context-compression.raw.txt)

**整理说明**

按上下文将“店里”作为“之类”等口述杂音处理，保留关键节点的意思；将“外资按需检索”整理为“外置，需要时检索”；将“一口用子agent”整理为“用子 Agent”。

**Agent 补充**

这里要区分累计缓存读取量与轨迹长度：原题讨论前者的近似二次增长。书中参考提纲也强调按阈值批量压缩、外置结果和子 Agent 隔离，避免每轮重建缓存。见[第一章参考答案第 2 题](../book/reference-answers.md#第一章-ai-agent-入门)。

### 第 3 题 · 自主模型与确定性约束如何共存

- 原稿：[2026-11-08-q03-harness-and-autonomy.raw.txt](chapter-01/raw/2026-11-08-q03-harness-and-autonomy.raw.txt)

**整理说明**

把探索性任务与固定流程分开组织；原稿“彻底杜绝其他可能性”在此具体表达为限定执行顺序、排除其他路径。

**Agent 补充**

“排除其他路径”的保证取决于约束是否覆盖全部实际执行入口，不能由模型口头承诺流程顺序来保证。正文强调执行前、上下文之外的验证机制，见[第一章护栏部分](../book/chapter1.md)。

### 第 4 题 · 循环诱因、重复检测与人工接管

- 原稿：[2026-11-08-q04-loop-termination.raw.txt](chapter-01/raw/2026-11-08-q04-loop-termination.raw.txt)

**整理说明**

按重复调用检测的上下文，将“after we have a dash”理解为“达到设定阈值后”；将“human loop”统一为 human-in-the-loop。

**其他记录**

具体阈值留待实现时按任务设定。

### 第 5 题 · 从感知、行动、策略观察 DeepSeek

- 原稿：[2026-11-08-q05-deepseek-product-observation.raw.txt](chapter-01/raw/2026-11-08-q05-deepseek-product-observation.raw.txt)

**整理说明**

全部表述保留为个人使用观察；将有关沙盒、开源和免费的解释明确标为推测，未写成已确认的因果关系。

**Agent 补充**

此处沿用你对产品功能的观察，没有核实产品当前能力或内部架构。原稿把 thinking 放在行动维度，整理版保留该使用体验；做架构分析时，还可把“如何推理和选工具”与“实际可执行哪些动作”区分开。对照依据为[第一章的产品比较与护栏讨论](../book/chapter1.md)。

### 第 6 题 · 航班客服采用混合模式

- 原稿：[2026-11-08-q06-hybrid-booking-workflow.raw.txt](chapter-01/raw/2026-11-08-q06-hybrid-booking-workflow.raw.txt)

**整理说明**

结合客服输入风险语境，将“drill breaking”理解为 jailbreak；将“转化成 rules”理解为把用户输入映射到既有规则和工具可处理的形式；将“返还款”整理为“退款”。

### 第 7 题 · 根据工具和参数进行动态风险评级

- 原稿：[2026-11-08-q07-parameter-aware-risk.raw.txt](chapter-01/raw/2026-11-08-q07-parameter-aware-risk.raw.txt)

**整理说明**

结合动态风险评级的上下文，将“We sell the classifier”理解为 risk classifier；将分类器与规则检查组织为“辅助筛查 + 确定性最终检查”，保留原稿对规则判断的偏好。

**Agent 补充**

结构化字段与正则检查可以成为确定性规则的一部分，但不能据此保证全面防住提示注入。正文还要求高风险复核在上下文之外、动作生效之前完成，见[第一章护栏部分](../book/chapter1.md)。

### 第 8 题 · 受限动作空间适用的场景

- 原稿：[2026-11-08-q08-restricted-action-space.raw.txt](chapter-01/raw/2026-11-08-q08-restricted-action-space.raw.txt)

**整理说明**

将英文短答整理为中文，保留原有简短程度；没有替用户扩写案例。

**其他记录**

当前答案只给出适用场景，尚未展开具体案例与原因。

### 第 9 题 · 用户暂未响应时如何移交控制

- 原稿：[2026-11-08-q09-human-handoff.raw.txt](chapter-01/raw/2026-11-08-q09-human-handoff.raw.txt)

**整理说明**

将对产品的观察与自己的设计原则分开；整理假设下可继续的低风险工作，以及缺少确认时应暂停的高风险工作。

**其他记录**

“一两分钟”仅保留为个人观察中的例子，不是统一的产品等待时限。

### 第 10 题 · 可能随模型能力演进的工程手段

- 原稿：[2026-11-08-q10-evolving-engineering-techniques.raw.txt](chapter-01/raw/2026-11-08-q10-evolving-engineering-techniques.raw.txt)

**整理说明**

将“React”规范为此处语境下的“ReAct”；将口述“严谨的约束，啊，采样”整理为“严格的约束采样”；保留参考答案启发与个人首先想到的例子的来源区别。未保留与回答论点无关的口述年份。

**Agent 补充**

格式补丁收益降低，并不表示高风险操作可以省略确定性格式校验；外部知识库也不意味着必然消失。题目所附的[第一章参考答案第 10 题](../book/reference-answers.md#第一章-ai-agent-入门)保留了这些适用边界。

### 重点摘记

结合第 10 题，将 `reasoning o1` 理解为推理能力增强后逐渐内化的显式推理提示，将 `json format` 理解为输出格式修复；`GONE` 表达这些能力补丁可能被内化、收益降低。这一解释针对具体补丁，Harness 的约束与验证仍是本次笔记的关注点。
