# 第一章 · Agent 与 Harness 重点

- 记录日期：2026-11-08（Asia/Taipei）
- 来源：[用户本次原始输入](../2026-11-08-input.raw.txt)
- 对应正文：[第一章](../../../book/chapter1.md)
- 正文版本：`dbc046eb896ac4e39aa19c7774c8bf49583b89a6`
- 内容类型：我的学习摘记，已整理，未经用户确认

## 重点

- **AI Agent = LLM + Harness**。我的关注点包括 Harness 中的 tools、context，以及 constrain、verify、correctness：既要让 Agent 能做事，也要约束行为并验证结果。
- **工具的有无和接口设计会明显影响效果**。不仅要问“有没有这个工具”，也要关注工具接口是否适合任务和模型使用。
- **Agent 的自由度需要设计**：工具要细化到什么程度、是否继续 loop 由谁决定、哪些行为依赖 context 引导、哪些约束用 hard code 实现，都值得关注。
- **关注被模型内化的 Harness 补丁**：原始速记为 `Harness "reasoning o1" "json format" GONE`。结合第 10 题，我关注的是：模型推理和格式遵循能力增强后，部分显式推理提示、格式修复等补丁的作用可能减弱。

## 待确认

- `reasoning o1` 具体指哪一类推理补丁，尚未明确；整理版保留原始速记，避免把它扩写成确定结论。
- `GONE` 在此暂按“部分补丁可能被内化、收益降低”理解，不表示整个 Harness 或结果校验不再需要。这个理解有待用户确认。

## 关联思考

- [第 1 题：根据瓶颈选择模型、上下文或工具](../answers/2026-11-08-q01-capability-bottleneck.answer.md)
- [第 3 题：自主模型与确定性约束如何共存](../answers/2026-11-08-q03-harness-and-autonomy.answer.md)
- [第 10 题：可能随模型能力演进的工程手段](../answers/2026-11-08-q10-evolving-engineering-techniques.answer.md)
