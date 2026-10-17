# ADK Eureka Implementation（Module 4）：把 Eureka 变成可建的系统

> 这一章把 Module 3 的 Eureka 从“概念”变成“可建的 ADK 系统”。关键降维：用一个 2-D 网格导航任务（Gymnasium，有目标格 + 墙）代替 GPU 机器人，循环一样，迭代飞快。读法：先看整条 pipeline 跑通长什么样，再逐个阶段放大。

## 1. 整体设计：专科 Agent 的顺序 Pipeline，不是单个大 Prompt

Agent 是顺序 pipeline 里一串 named specialist sub-agent（命名专科子 agent），不是一个 monolithic prompt（单体大 prompt）：SetupAgent 准备状态 → RewardGenerator 提议候选奖励函数 → Evaluator 打分 → RewardSelector 选最好 → ReflectionAgent 写结构化 critique（批评）→ HumanFeedbackAgent 可选加人工编辑 → loop controller 决定迭代还是停。

Pipeline 保持 shared, inspectable state（共享、可检查的状态）+ log + live leaderboard（实时排行榜）+ artifact（产物）—— 奖励代码、指标、反思文本、最佳候选文件 —— 行为可审计、可复现。Demo 输出：系统几轮迭代就找到一个紧凑合理的奖励（朝目标进展 + 到达 bonus + 小步惩罚 + 撞墙惩罚），加一次人工编辑改变行为。

什么时候用这种拆法：需要 debuggable（可调试）、可扩展的 generate–evaluate–improve（生成-评估-改进）循环。别建成一个藏状态的单个大 prompt。例：网格导航代替机械手，迭代快、教的是同一循环。失败模式：单个 prompt 循环里坏奖励悄悄污染后续步骤。Fix：拆角色 + 状态/日志/产物显式 + 按 fitness 打分。

## 2. SetupAgent：Grounding 阶段，只干一件事

SetupAgent 只跑一次、在最前，干 grounding（接地）：加载并校验 task / env / model / loop 配置（任务定义、奖励形状、迭代上限、模型、baseline、停止阈值）、校验生成代码和环境的 contract（契约）、创建环境实例并读它的源码、加载已有的 prior reward（如果跑过）、创建输出目录、初始化 leaderboard 和 metrics 文件、跑一个短 baseline rollout（基线试跑）建参考分。

它有意 idempotent（幂等）：检测到已有 run，不 wipe（抹掉）历史，保留 audit trail（审计轨迹）；配置坏了就 fail fast（快速失败）报清晰 contract error，不让坏配置 cascade（级联）。核心思想：LLM 永远不猜环境长什么样；它收到的是源码、state variable、action / observation 形状、和一个被检查过的 contract。

只要生成代码必须匹配环境，就用 dedicated setup phase（专用搭建阶段）。别把 setup 和 generation 混在一起。例：先校验任务，再记 baseline，再把 grounded package（接地后的包）交给 generator。失败模式：generation 挂在变量名不匹配 / 缺字段上。Fix：先校验 contract，坏了就大声停。

## 3. 生成 + 评估：创意和裁判分开

Generator 和 Evaluator 是两个不同技能的 agent。**Generator**（LLM agent + structured-output function）：每轮从任务描述、环境源码、最新 reflection / feedback 写一个新鲜候选奖励函数，返回 title、reward code、rationale（理由）、可执行 evaluator spec，并且被要求不逐字复用上一个候选。**Evaluator**：用 PPO 在短 Gymnasium rollout 里跑每个候选，量 cumulative reward（累计奖励）、time to goal（到目标时间）、steps（步数）、success rate（成功率）；失败（语法错、坏名字、crash）的得 -inf，错误留给 reflection。

Pipeline 是 fault-tolerant（容错）且并行：多个候选一起训、一起评，一个失败不停循环。生成是创意的、评估必须客观可重复时，就分开。别让 generator 给自己打分。例：每轮 5 个候选，各训各评，最好者晋级。失败模式：奖励出错或 game metric（刷指标）。Fix：隔离执行、错误扣分、错误文本留给下一轮修正。

## 4. 选择 + 反思 + 人反馈：标量不够，要诊断

评估后，**Selector** 按 true fitness（normalized total reward，归一化总奖励，不是生成的奖励本身）排名，把最好者记进 leaderboard。**ReflectionAgent** 再对最好候选做文本反思：报告 fitness 和每分量奖励在开始/结束的值、哪个分量变了、哪个被忽略/主导，产出 targeted tuning advice（定向调优建议）—— 如某分量平了就加权重，某分量爆炸就加 cap，处理某个失败模式。

**Human feedback** 只在 checkpoint（检查点，反思之后、下一代之前）进场，被 append（追加），不直接执行；下一代 prompt = 任务 + 环境 + 上一最好 + 反思 + 人反馈。人输入最有价值的地方：风格、安全、fitness 漏掉的 edge case（边界情况）。

Scalar 分数 alone（光一个标量）说不清修哪时，就用这个。别只回一个数字，也别让人文本 bypass validation（绕过校验）。例：reflection 说 progress 项 plateau（平台期）了，下一轮奖励加权重；人加“别贴墙走”当新惩罚想法。失败模式：盲目变异改错项。Fix：per-component 诊断 + checkpoint 人输入。

## 5. Loop Control：退出逻辑要显式，别让 LLM 决定停不停

整个 pipeline 装在两个文件里：agent.py（entry point，入口）定义 SequentialAgent —— SetupAgent 一次，然后循环；reward_loop.py（loop factory，循环工厂）建重复的 LoopAgent，包含 Generator → Evaluator → Selector → Reflection（+ 可选 HumanFeedback），配 shared state object（迭代数、最佳候选、leaderboard、metadata）。LoopAgent 内部是顺序的，受 exit condition（退出条件）守卫；controller 看 iteration count（迭代数）和 patience counter（耐心计数器），每轮之后有 EscalationChecker：max iterations（最大迭代数）到、score target（分数目标）到（常带更严的 margin）、连续若干轮无改进、fatal error / timeout（致命错误/超时）时停，否则继续。Log + artifact（指标、反思文本、最佳代码、退出原因）让每次迭代可复现。

只要循环可能永远跑，就用显式退出逻辑。别靠 LLM 决定何时停。例：5 轮后停，或 2 轮无改进后停，先到为准。失败模式：无限循环，或噪声上过早停。Fix：硬 cap + patience + 目标阈值 + 记录退出原因。

## 这门课怎么教的（Module 4）

先给一个 concrete、可跑的 demo（小网格任务）让抽象 Eureka 循环可见，再按 pipeline 执行顺序走：setup → generate / evaluate → select / reflect / feedback → assemble / control。同一小例子贯穿，代码是小段带注释的 chunk，不是整段 dump，配 state flow 和 loop control 图。心智模型：专科 agent 当 pipeline 阶段，共享可检查状态。每课只隔离一个阶段、点名输入/输出，好跟。难处：ADK 专属原语（SequentialAgent、LoopAgent、EscalationChecker）来得快，容易和概念循环混淆。最值得偷的：**先 demo 整条循环端到端跑在 toy 任务上，再放大每个阶段 —— 学习者先锚定输出，再看代码**。要避免：先讲 loop factory，再让读者看一次完整迭代。

## 自测 4 问（带答案要点）
1. 为什么拆成专科 agent pipeline，而不是单个大 prompt？
   → 可调试、可扩展；状态/日志/产物显式；坏奖励不会悄悄污染后续。
2. SetupAgent 干什么？为什么 idempotent + fail fast？
   → grounding：校验配置/contract、读源码、记 baseline、建输出；不抹历史、保留审计；配置坏就大声停，不 cascade。
3. Generator 和 Evaluator 为什么分开？失败的候选怎么处理？
   → 生成创意、评估客观可重复；失败得 -inf，错误留给 reflection，不停循环。
4. 退出条件有哪几个？为什么不让 LLM 决定停？
   → max iterations / 目标到 / 若干轮无改进 / 致命错误/超时；LLM 停不准；要硬 cap + patience + 目标 + 记录退出原因。
