# Case Studies + Wrap-up（Module 3–4）：Pattern 拼回真实系统

> 前两章是“零件 + 五种拼法”，这一章是“拼出来的两台整机”。读法：每看一个 case，先问“这里用了哪几个 pattern，各自负责什么，没了它会坏什么”。

## 1. Multi-Agent Research System：Pattern 组合的研究 pipeline

**组合**：orchestrator-worker（总规划）+ parallelization（并行搜索）+ evaluation（过滤 + 合成）。

流程：把一个宽泛的研究问题拆成子问题 → 并行搜索多个来源 → 丢掉弱来源、去重 → 由一个 synthesis owner 合成结构化报告。

适合：宽泛、多来源的研究问题。**不适合**：单事实查询 —— 杀鸡不用整条 pipeline。

失败模式：重复劳动、低质量来源、合成 incoherent（各段各说各的）。Fix：deduplication（去重）+ source checks（来源检查）+ 一个 synthesis owner（合成只能有一个负责人，不能三个 worker 各写一段直接拼）。

教学点：这是 Module 2 的 pattern 第一次“合体”—— 同一个系统里，orchestrator-worker 管分工，parallelization 管速度，evaluation 管质量。缺了任何一个，系统都会以一种可预测的方式变差。

## 2. Coding Agent System：有测试守门的代码 agent

**组合**：tools（读写文件、跑测试）+ structured output（改动描述）+ planning（改动计划）+ evaluator-optimizer（测试反馈 → 再改）。

流程：规划代码改动 → 小 diff 落地 → 跑测试 → 按失败反馈再改 → 再测，直到通过或触发 stop rule。

适合：范围清晰、有可验证测试的 coding 任务。**不适合**：需求含糊、或改动未经 review 就能进生产。

失败模式：破坏现有代码，或在同一个 failing test 上无限循环。Fix：small diffs（小步改动）+ test gates（测试是门，不是通过不能往下走）+ rollback / stop rules（改坏了能回退，卡住了能停）。

教学点：coding agent 是 evaluator-optimizer 最干净的例子 —— evaluator 不是一个“感觉”，而是测试套件，pass / fail 是客观的。这就是为什么 evaluator-optimizer 在这里特别好用：评判标准是 ground truth，不是另一个模型的品味。

## 3. Wrap-up：Build Ladder —— 简单 → 测 → 再加复杂

收尾给的不是新 pattern，而是一条**build ladder（搭建阶梯）**：

1. 先把 prompt 写好（最简单的系统）。
2. 加上 evaluation（定义 success metric，先能量出好坏）。
3. 只有当 measurement 证明不够时，才加复杂结构（多 agent、编排、反馈环）。

Pattern 是共享词汇，不是 rigid 的盒子 —— 真实系统里 pattern 经常 blend（一个 orchestrator 里既有 routing 又有 parallel）。先给系统起名字（“这里是 routing，那里是 evaluator”），团队才能讨论 trade-off。

挑战清单（上线前逐项检查）：fragile logic（逻辑脆）、nondeterministic behaviour（同样输入不同输出）、hard debugging / tracing（难定位哪一步坏了）、live-update difficulty（在线更新难）、coordinator bottlenecks（协调员瓶颈）。

失败模式（全课最大的）：over-engineering（上来就搭多 agent）或 shipping without evaluation（没测就上线）。Fix：先定义 success metric，再决定要不要加复杂。

## 这门课怎么教的（Module 3–4）

Module 3 是全课 pedagogy 最强的一段：两个端到端系统，每个都配一张 **pattern-mapping table（pattern → 在这个系统里负责什么 → 没了它会坏什么）**。同样的 pattern 在真实 pipeline 里再出现一次，印象就固化了。Module 4 用 build ladder 收尾，给读者一条“下次动手时照着走”的规则，而不是总结复读。注意一个导航坑：最后一课的 Next 按钮指向 mock interview，看起来像课程还没完 —— 读到 synthesis 就该停，mock 是跳过的练习。

## 自测 4 问（带答案要点）
1. Research system 用了哪几个 pattern，各自负责什么？
   → orchestrator-worker 分工 + parallelization 并行搜索 + evaluation 过滤合成；缺一不可。
2. Coding agent 里 evaluator 是什么？为什么它比一般 evaluator 可靠？
   → 测试套件；pass / fail 是客观 ground truth，不是模型品味。
3. Build ladder 三步？为什么 evaluation 在加复杂之前？
   → 好 prompt → 加 evaluation → measurement 证明不够才加复杂；没有 measurement，加复杂是盲目。
4. 全课最大的两个失败模式？Fix？
   → over-engineering + 没测就上线；先定义 success metric，再决定复杂度。
