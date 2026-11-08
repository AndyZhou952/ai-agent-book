# 第一章 · 重点笔记

- **AI Agent = LLM + Harness**。我的关注点包括 Harness 中的 tools、context，以及 constrain、verify、correctness：既要让 Agent 能做事，也要约束行为并验证结果。
- **工具的有无和接口设计会明显影响效果**。不仅要问“有没有这个工具”，也要关注工具接口是否适合任务和模型使用。
- **Agent 的自由度需要设计**：工具要细化到什么程度、是否继续 loop 由谁决定、哪些行为依赖 context 引导、哪些约束用 hard code 实现，都值得关注。
- **关注被模型内化的 Harness 补丁**：模型推理和格式遵循能力增强后，部分显式推理提示、格式修复等补丁的作用可能减弱。
