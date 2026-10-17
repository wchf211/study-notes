# Teaching Lens：这门课怎么组织讲解，值得偷什么

> 这篇是讲法审计，不是技术复读。注意：本课 digest 本身是 high-level 版（标题/开头/结构提炼，非逐段深读），以下讲法观察来自模块结构与每课 hook / 顺序，结论可信，逐段细节仍需深读确认。

## 1. 全课结构：一条 storyline + 每模块一个 hook

- **一条 storyline**：prompt → RAG → agent → fine-tune → eval → deploy → safety。观众始终知道自己在 pipeline 的哪一段。
- **每模块一个 hook**：每个模块都用一个具体场景开场（带延迟/成本上限的聊天功能、misbehaving 分类器、必须引用的客服助手、必须用工具的研究助手、模型包成 API 必须又稳又便宜、对抗用户）。没有一个模块是定义先行。
- **决策先行**：fine-tune 章先给决策树（prompt vs RAG vs tune），再讲方法；RAG 章先给分阶段排障，再讲修法。观众先学会“什么时候用”，再学“怎么用”。

## 2. 值得偷的讲法

1. **耦合预算方程**：总花费 \~ 调用数 × token，一行公式管住成本/延迟/上下文三个变量。讲 production LLM 时，这就是开场图。
2. **Hierarchy 图 = 安全边界**：system > user > tool，一张图讲清 prompt injection 为什么是安全问题，不是“模型不听话”。
3. **Failure triage by stage**：RAG 排障按 pipeline 阶段分（缺文档 / 坏 chunk / 检索错 / context 被忽略 / 幻觉填充），观众可以照着查。
4. **API 边界 + 预算/告警的生产心智模型**：Module 8 是全课对 YouTube 最有用的模块 —— 模型藏在稳定 API 后面，成本控制当产品功能，eval 进 CI/CD。
5. **Eval 当回归测试**：eval 集 = 测试集，prompt 改动 = 代码改动，回归就拦。这个类比一句话就能讲清。

## 3. 容易讲砸的坑（这门课自己踩过的）

1. **一课塞两个大概念**：structured outputs + function calling 挤进一课，CI/CD 课很压缩。讲视频时，一课一个 idea，宁可拆两期。
2. **顺序小错位**：hybrid search 在完整 pipeline 之前讲；framework 在 loop 讲透之前讲；幻觉放在 A/B 之后而不是和 grounding 放一起。讲的时候要先给全图，再讲零件。
3. **广度大于深度**：多模态章每个模态只有一个 intro，偏见课最抽象、缺具体 worked example。讲的时候要么给一个具体例子讲透，要么明确说“这期只给地图”。

## 4. 和另两门课的讲法对照

- **AI Product Engineering**：一个 running example + failure-driven。本课的 hook 密度接近它，但没有 running example 贯穿。
- **Master Agentic Design Patterns**：重复模板 + pattern-mapping table。本课的“每模块一个 hook + 决策先行”和它互补。
- **三门合起来**：running example（AIPE）+ pattern-mapping table（Patterns）+ 耦合预算方程 + API 边界心智模型（本课）= 生产 LLM 讲解的完整脚手架。

## 自测 4 问（带答案要点）
1. 全课 storyline 是什么？每模块怎么开场？
   → prompt → RAG → agent → fine-tune → eval → deploy → safety；每模块一个具体场景 hook。
2. 最值得偷的两个讲法？
   → 耦合预算方程；API 边界 + 预算/告警的生产心智模型。
3. 这门课最大的讲解坑是什么？
   → 一课塞两个大概念；零件在全图之前讲；广度大于深度时没给具体例子。
4. 本课 digest 的可信度边界是什么？
   → high-level 版（标题/开头/结构），非逐段深读；逐段细节需深读确认。
