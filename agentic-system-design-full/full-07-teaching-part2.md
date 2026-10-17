# Teaching Lens Part 2：Modules 4 + 6 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 4 + 6（Part 2 已读部分；Module 5 只有 1 个 thought exercise，按跳过规则未读）。Modules 7–13 的讲法等 Part 3 再补。

## 1. Module 4：先 Demo 整条循环，再放大每个阶段

先给 concrete、可跑的 demo（小网格任务）让抽象 Eureka 循环可见，再按 pipeline 执行顺序走：setup → generate / evaluate → select / reflect / feedback → assemble / control。同一小例子贯穿，代码是小段带注释 chunk，不是整段 dump，配 state flow 和 loop control 图。

心智模型：专科 agent 当 pipeline 阶段，共享可检查状态。每课只隔离一个阶段、点名输入/输出。难处：ADK 专属原语（SequentialAgent、LoopAgent、EscalationChecker）来得快，容易和概念循环混淆。

最值得偷的：**先 demo 整条循环端到端跑在 toy 任务上，再放大每个阶段 —— 学习者先锚定输出，再看代码**。要避免：先讲 loop factory，再让读者看一次完整迭代。

## 2. Module 6：先给痛点，再按系统自己的两阶段流

先给 relatable 痛点 —— 盯着空白画布 —— 再按 ChainBuddy 自己的两阶段流：clarify → generate → 证明有效。Running example 是单个工具（ChainForge / ChainBuddy）贯穿，planner / worker / orchestrator 团队用人团队类比解释。心智模型：copilot 当 starter + editor，不是 autopilot。每课对应一个阶段，评估课把结果绑回架构选择。

最值得偷的：**用用户感知 + 独立专家质量双轨评估一个 assistant，再解释两者的差距 —— 把“它有效”的断言变成关于 overreliance 的设计课**。要避免：只给 usability 分数当成“有帮助”的证明；评估课一次引入几个指标时，要先排哪个指标回答哪个问题。

## 3. 两章共通的讲法

- **Toy task 先行**：网格导航、ChainForge  starter，都是“小任务承载大循环”。
- **按执行顺序讲**：pipeline 怎么跑，课就怎么排，观众不用跳。
- **输出可审计**：artifact / leaderboard / spec 回显，都是“让过程可见”的设计，讲的时候要特意展示。

## 自测 4 问（带答案要点）
1. Module 4 的讲解顺序？为什么先 demo 再放大？
   → demo 整循环 → setup → generate/evaluate → select/reflect/feedback → assemble/control；先锚定输出，再看代码。
2. Module 6 的讲解顺序？心智模型？
   → 痛点 → clarify → generate → 证明有效；copilot 当 starter + editor，不是 autopilot。
3. 两章共通的讲法？
   → toy task 先行；按执行顺序讲；输出可审计（artifact / leaderboard / spec 回显）。
4. 评估 assistant 时为什么要双轨？单轨会漏什么？
   → 用户感知 + 专家质量；单看满意度会漏 overreliance 风险和 perception–reality gap。
