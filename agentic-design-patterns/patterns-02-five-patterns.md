# Five Patterns（Module 2）：五个可组合的组织方式

> 记住一张对比表：chaining 是“按固定顺序串起来”；routing 是“先分类再分派”；parallelization 是“独立子任务同时跑再合并”；orchestrator-worker 是“中央协调员动态拆派”；evaluator-optimizer 是“生成 → 评判 → 再改”的反馈环。前四个管“怎么做”，最后一个是包在任意一个外面的“怎么变好”。

## 1. Prompt Chaining：固定顺序，一步接一步

**Prompt chaining**：把工作拆成固定序列，每一步的输出喂给下一步（draft → critique → revise）。**Error propagation（错误传播）**：早期步骤的错会一路放大到最后，这是 chaining 的头号敌人。

适合：能拆成清晰中间产物、有明确 checkpoint 的任务。例：家教先列大纲 → 再写讲解 → 再出练习题。**不适合**：需要分支、动态重规划的任务，或早期错误会 cascade 的场景。

失败模式：中间产物坏了，后面全跟着坏。Fix：步骤之间加 validation，中间产物不合格就 early exit，不要硬往下传。

和其它 pattern 的关系：最简单的顺序 pattern；对比 parallel（并行）和 orchestrator-worker（动态协调）。

## 2. Routing：先分类，再交给专科

**Routing**：一个分类器 / 分派器把每个请求送到最合适的专科路径、模型、或工具。**Specialist（专科）**：只处理一类请求的组件，通常比通用组件在该类上更准、更便宜。

适合：请求类型混杂，专科明显优于通用。例：客服工单分流到 billing / technical / account 三条线。**不适合**：任务很均一，或分类本身不可靠 —— 分类错了，后面全错。

失败模式：模糊输入被 misroute。Fix：类别定义清晰 + fallback / default 路径 + confidence 检查（不确定就走通用路径或问人）。

和其它 pattern 的关系：路由决策本身通常用 structured output；和 parallelization 搭配可以先分流再 fan-out。

## 3. Parallelization：独立的就同时跑，再合并

**Parallelization**：把独立子任务同时执行，再合并结果。两种常见形态：(a) 把一个任务切成几段并行做（sectioning）；(b) 对同一任务跑多个尝试，再挑 / 合并最好的（sampling / voting）。

适合：子任务互相独立，或多样尝试能提升质量。例：同时调研几个子主题，再合成一份报告。**不适合**：步骤互相依赖，或额外调用的成本 / latency 超过收益。

失败模式：各部分结果不一致、冲突。Fix：明确的 merge / synthesis 步骤 + 各路用一致的指令。合并不是“堆在一起”，而是一个有 owner 的合成步骤。

和其它 pattern 的关系：chaining 的对偶（顺序 vs 并行）；orchestrator-worker 里常包含并行。

## 4. Orchestrator-Worker：中央协调员动态拆派

**Orchestrator-worker**：一个中央 coordinator 把 goal 拆成子任务，分派给专科 worker，收结果、组装，过程中根据新结果调整分工。**Coordinator bottleneck（协调员瓶颈）**：coordinator 成为单点慢 / 错的来源，是这个 pattern 的头号风险。

适合：复杂、会演化的 goal，子任务类型多样。例：research 系统的 planner 派 search、analysis、summarization 三个 worker。**不适合**：简单或完全可预测的工作 —— coordinator 是纯 overhead。

失败模式：coordinator 瓶颈，或子任务边界不清（两个 worker 做重了 / 漏了）。Fix：子任务小而边界清晰 + 明确的 handoff 格式（worker 交回什么、什么格式）。

和其它 pattern 的关系：它是 chaining + routing + parallelization 的泛化 —— 固定顺序、分类分派、并行，都可以是 coordinator 手里的手段。

## 5. Evaluator-Optimizer：生成 → 评判 → 再改，直到过关

**Evaluator-optimizer**：generator 出一稿 → evaluator 按标准打分 / 给 critique → generator 再改，循环直到通过或达到上限。**Evaluator bias（评判者偏见）**：evaluator 自身的盲点会被放大成“优化方向”，是这个 pattern 的头号风险。

适合：质量敏感的输出（代码、写作、计划），且迭代真的能变好。例：写代码 → 跑测试 / 评审反馈 → 再改。**不适合**：低 stakes 或时间紧的输出，或 evaluator 并不比 generator 高明 —— 那迭代只是在原地打转。

失败模式：无限循环，或 evaluator 偏见被当成真理。Fix：max iterations（硬上限）+ 具体可检查的 criteria（不是“写得更好点”）+ 关键处用 human / ground-truth 校验。

关键纠正：**Evaluator-optimizer 不是和其它四个平级的 pattern，而是一层可以包在任意 pattern 外面的反馈环**。把它当成列表里第五个并列项，读者会误归档。讲的时候要明确说“这是 layer，不是 peer”。

## Pattern 选择速查

| 任务形状 | 首选 | 别用 |
|---|---|---|
| 固定顺序、中间产物清晰 | Chaining | 需要分支 / 动态重规划时 |
| 请求混杂、专科明显更强 | Routing | 分类本身不可靠时 |
| 子任务独立、或多样尝试更好 | Parallelization | 步骤互相依赖时 |
| 复杂、演化、多样子任务 | Orchestrator-worker | 简单 / 完全可预测时 |
| 质量敏感、迭代能变好 | Evaluator-optimizer（包在外面） | 低 stakes / evaluator 不比 generator 强时 |

## 这门课怎么教的（Module 2）

每课模板高度一致：白话定义 → 类比 / 图 → 一步步怎么做 → 代码走读 → 好处 / 局限 → when-to-use 清单。每课换一个新领域（家教、客服、研究），生动但比较的负担在读者。最值得偷的：**what-it-is / when-to-use / failure-mode 三件套**，观众可以把五个 pattern 并排比较。最该避免的坑：把 evaluator-optimizer 当并列项讲 —— 要讲成 layer。

## 自测 4 问（带答案要点）
1. 五个 pattern 各用一句话说清？Evaluator-optimizer 为什么是 layer 不是 peer？
   → chaining 顺序串；routing 分类分派；parallel 并行合并；orchestrator-worker 中央动态拆派；evaluator-optimizer 生成→评判→再改；它可以包在任意 pattern 外面。
2. Chaining 的头号敌人是什么？Fix 是什么？
   → error propagation；步骤间 validation + early exit。
3. Routing 什么时候别用？Parallelization 什么时候别用？
   → 分类不可靠时别 routing；步骤依赖 / 额外调用成本超过收益时别 parallel。
4. Orchestrator-worker 的头号风险？Evaluator-optimizer 的头号风险？
   → coordinator bottleneck + 子任务边界不清；evaluator bias + 无限循环；Fix：小而清晰的子任务 + handoff 格式；max iterations + 具体 criteria + human / ground-truth 校验。
