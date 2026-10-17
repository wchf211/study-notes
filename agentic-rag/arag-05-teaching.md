# Teaching Lens：Agentic RAG 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖全课 12 课（Modules 1–4；Module 5 只有 Test Your Skills，按跳过规则未读）。

## 1. Module 1：Problem-First、Layer-by-Layer

Problem-first、layer-by-layer —— 先给熟悉的 pipeline，再用具体失败暴露局限，再加 agency，再 dissect agent，再比较推理策略。好懂处：analogy-heavy（开卷考试、图书管理员 vs 研究助理、电影导演、CPU vs OS）、concrete scenario（HR PTO、PPO vs HMO、Titan、Transformer / 作者）、worked trace、comparison table。Show Answer 预判困惑（brain vs planner、为什么不直接用 frontier LLM）。

最值得偷的：**先给 relatable failure，每个概念一个类比、反复用，展示完整 trace，最后用 trade-off 表收尾**。要避免：具体例子之前堆抽象术语；课程每个术语先在场景里落地。

## 2. Module 2：一个项目横跨多集

Design-before-code、side-by-side、modular build、assembly —— 蓝图、静态 vs agentic、工具隔离、main.py 编排。好懂处：promotion metaphor（pipeline 变成 tool）、反复强调 tool description 驱动 reasoning、line-by-line walkthrough 绑到单个项目、sanity-check 查询预测哪个工具该触发。

最值得偷的：**一个项目横跨多集，单个变更做 before / after，把 trace 当故事讲（thought、action、observation）**。要避免：长段不间断代码讲解，不按函数切块、不说看什么；课程按 labeled line range 切块，每个工具组装前隔离测试。

## 3. Module 3：定性到定量

Qualitative-to-quantitative —— 失败分类 + trace 方法、三招 fix（description、错误处理、系统提示）、live Test-Observe-Refine-Repeat 演练、golden dataset + RAGAs、研究前沿（QPP、Critic、分类）、自动化 Test-Refine-Measure-Repeat。好懂处：agent psychologist / detective 框架、每招 fix 的 before / after 对、单个棘手查询（Yann LeCun / Transformers）贯穿 failure-diagnosis-fix-success、具体分数 + NaN 解释。

最值得偷的：**在屏幕上从真实 trace 诊断，展示精确的 before / after 编辑，重跑同一查询让改善可见**。要避免：指标只讲抽象定义；课程把每个指标绑到“它会改变什么”，并警告 rate-limit / 时间成本。

## 4. Module 4：先局限再解法

Limitation-then-solution、operations、horizon —— 每课从上一阶段的局限开始（flat vector search 忽略结构；能跑的 prototype 不是 production；课程结束不是领域结束），再给下一层（graph、production 支柱、未来趋势）。好懂处：two-problem 结构 + 总结表（Dependency Graph）、four-pillar checklist（Production）、recap-plus-roadmap 收尾（Future Trends）；生动类比（flat world、pets vs cattle、control tower）和具体威胁/失败例子（stock ticker、餐厅预订、prompt-injection 辞职邮件、delete_all）。

最值得偷的：**把高级主题框成“接下来会坏什么”，每课只给两三个 named problem + 收尾表，最后给可行动的下一步**。要避免：只有 hype 的未来谈话；课程把每个远期想法映射回已经教过的部件（guardrail、brain / tool / planner / memory）。

## 5. 全课讲法总结

- **一个项目贯穿**：AI research assistant 从蓝图到工具到组装到调试到评估到生产，观众不用每课重新理解例子。
- **Trace 是核心教具**：Thought / Action / Observation 不是附带，是每课的证据。
- **Before / after 对**：每个 fix 都展示编辑前和编辑后，重跑同一查询。
- **类比反复用**：图书管理员、电影导演、建筑事务所、宠物 vs 牛群 —— 每个类比在多课出现，不是只用一次。
- **Trade-off 表收尾**：每组比较（推理策略、工具依赖、未来趋势）都用表收尾，观众带走的是判断。

## 自测 4 问（带答案要点）
1. Module 1 最值得偷的讲法？
   → 先给 relatable failure，每个概念一个类比、反复用，展示完整 trace，最后用 trade-off 表收尾。
2. Module 2 最值得偷的讲法？
   → 一个项目横跨多集，单个变更做 before / after，把 trace 当故事讲。
3. Module 3 最值得偷的讲法？
   → 在屏幕上从真实 trace 诊断，展示精确的 before / after 编辑，重跑同一查询让改善可见。
4. Module 4 最值得偷的讲法？
   → 把高级主题框成“接下来会坏什么”，每课只给两三个 named problem + 收尾表。
