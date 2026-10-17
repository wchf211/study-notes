# Teaching Lens：Modules 1–3 怎么组织讲解，值得偷什么

> 这篇是讲法审计，不是技术复读。覆盖 Modules 1–3（Part 1 已读部分）；Modules 4–13 的讲法等 Part 2 再补。

## 1. 全课结构：地基 → 两个深 case → 工程化 → 更多 case → 收尾

- **Module 1**：地基（定义 → 零件 → 记忆 → 编排 → 安全 → 生产）。
- **Modules 2–3**：两个深 case，先给最难的两个系统（MACRS、Eureka），每个都讲透机制 + 评估 + 失败。
- **Modules 4–13（未读，见 Part 1 续读点）**：ADK 工程化、ChainBuddy、WebVoyager、MuLan、OpenClaw、收尾、附录。顺序是“先懂原理，再看工程化，再看更多领域”。

## 2. 值得偷的讲法

1. **失败对比开场**：Module 2 先给旧推荐器 vs 单个 LLM 各自怎么失败，再引出拆分；Module 3 先给手写奖励多痛苦，再给 Eureka。观众先看到痛，再看到解法。
2. **同一段对话 / 同一个任务讲两遍**：MACRS 把同一段对话用单 agent 和团队各走一遍，标注哪个 act 改变了结果。讲多 agent 时，这比五张架构图管用。
3. **Ablation 当证明**：Eureka 每讲一个机制，都给去掉它的后果（无 reflection 约掉 28.6%）。“去掉它会坏什么”比“它多厉害”更有说服力。
4. **按系统自己的 pipeline 顺序讲**：MACRS 按问题 → 架构/act → 反思 → 证据；Eureka 按 generate → evolve → reflect → 加人 → 证明。观众跟着系统的执行顺序走，不用跳。

## 3. 容易讲砸的坑（这门课自己踩过的）

1. **先介绍五个 agent 名，再给端到端 turn**：MACRS 的五六个角色名第一遍容易混。讲的时候要先给一段完整 turn，再点名角色。
2. **Fitness 和 reward 混淆**：Eureka 里这两个概念必须反复区分，否则观众会把代理指标当真实目标。讲的时候每次提到 reward，都要问一句“用哪个 fitness 验的”。
3. **先给最终提升百分比，再讲分数定义**：52% 提升在讲清 human-normalized score 之前出现，观众不知道 52% 是相对什么。讲的时候先定义指标，再给数字。
4. **架构课重叠**：Module 1 两节架构课（组件、交互）有重叠。讲的时候要合并成一节，或明确说“这节是上一节的动态版”。

## 4. 和 Crash Course / Patterns 的讲法对照

- **Crash Course**：框架 spine + verifier/critic motif。完整版 Module 2–3 是它的加深版：同一个 motif（reflection / critic）在真实系统里讲透。
- **Master Agentic Design Patterns**：重复模板 + pattern-mapping table。完整版的 MACRS / Eureka 都可以配一张 pattern-mapping table（哪个 pattern 在本系统里负责什么）。
- **三门合起来**：framework spine（Crash Course）+ pattern-mapping table（Patterns）+ 失败对比开场 + ablation 证明（本课）= 讲 agent 系统的完整脚手架。

## 自测 4 问（带答案要点）
1. Modules 1–3 的讲解顺序是什么？
   → 地基 → MACRS 深 case → Eureka 深 case；先懂原理，再看工程化 / 更多领域。
2. 最值得偷的两个讲法？
   → 失败对比开场；ablation 当证明（去掉它会坏什么）。
3. MACRS 最容易讲砸的点？怎么避？
   → 先介绍五个 agent 名再给 turn；改成先给一段完整 turn，再点名角色。
4. Eureka 最容易讲砸的点？怎么避？
   → fitness 和 reward 混淆；每次提到 reward 都问“用哪个 fitness 验的”。
