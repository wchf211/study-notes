# ChainBuddy（Module 6）：给 LLM Pipeline 做 Copilot

> 这一章的主角不是一个“更强的 agent”，而是一个“帮人起步的 copilot”。核心设计决策：输出可编辑的起点，不是锁定的最终答案。读的时候注意评估课 —— 它用“用户感觉”和“专家评分”的差距，讲了一个关于 overreliance（过度依赖）的设计课。

（注：Module 5 只有 1 个 thought exercise“Design a Smart Parking Agent System”，按跳过规则未读，没有 teaching lesson。）

## 1. Blank Page Problem：瓶颈是起步，不是打字

**LLM pipeline**：一串 node（节点，如 prompt、retriever、evaluator）选好、配好、连好，输出能工作。**Blank page problem（空白页问题）**：空白画布不给起点，新手不知道从何建起。

现有工具（LangChain、Haystack、LlamaIndex、ChainForge）给 building block（积木），但仍假设用户知道要建什么。**ChainBuddy** 是 ChainForge 里的 copilot：把自然语言目标变成可编辑的 starter pipeline —— 先澄清目标，再生成完整 workflow。架构两阶段：chat assistant 收集需求 → multi-agent 系统生成 pipeline。核心设计目标：理解用户意图、降低知识门槛、提高速度和输出质量。

Copilot 在“起步和结构”是瓶颈时用，不在打字时用。别把它当全自动 builder 锁死最终答案。例：“总结这些会议纪要”变成 guided 对话，再给带 evaluation node 的 draft pipeline，用户可编辑。失败模式：给用户空白画布 + 一堆零件。Fix：给可编辑起点 + 结构化澄清。

## 2. 需求收集 Chat Assistant：先确认 Spec，再动手建

Assistant 不急着建，跑三阶段对话：understand the task（理解任务：高层目标、数据格式、受众）→ specify evaluation criteria（定评估标准：什么算好输出、要避免什么）→ generate a specification（生成规约：需求的结构化总结）。

它用 LLM 做 adaptive、targeted 提问 —— 澄清含糊术语、挖 hidden assumption（隐藏假设）、复用答案 —— 不是固定问卷，并且把 growing specification（不断生长的规约）展示给用户纠正。两个机制保它不跑偏：**intent inference（意图推断）**（从含糊输入推断底层任务）、**state machine（状态机）**（跟踪哪些需求 slot（槽）填了、缺了、确认了，并据此引导提问）。

用户目标 underspecified（欠指定）、后期改错代价大时，用这个 pattern。需求确认前别让 generator 动手。例：用户说“总结文档”；assistant 问哪些文档、给谁、多长、排除什么，再确认 spec。失败模式：凭含糊首 prompt 建错 pipeline。Fix：elicit（引出）、structure（结构化）、confirm（确认）需求，再生成。

## 3. 多 Agent 生成框架：Planner + Workers + Orchestrator

生成拆成 Planner 和并行 Workers，由 Orchestrator 协调。**Planner（meta-agent，元 agent）**：读 spec，写 high-level blueprint（高层蓝图）：需要哪些 node、什么顺序、怎么连，加 rationale（理由）。**Workers（专科 agent）**：各负责一种 node 类型（如 prompt engineering、retrieval、evaluation、code generation），把 blueprint 的各段变成具体、配置好的 node，并行跑。**Orchestrator（编排器）**：把 workers 的输出组装成完整、可执行的 ChainForge workflow，校验连接，交回给用户编辑。

分工像真实团队 —— architect（架构师）vs specialist（专科）vs integrator（集成者）—— 每个 agent 的活窄、可验证。输出是多组件类型的结构化 artifact（产物）时用。别让一个 agent 一步同时规划、实现、集成。例：planner 定“retriever → prompt → evaluator”；retrieval worker 配 retriever，prompt worker 写 prompt，evaluator worker 定检查；orchestrator 接线。失败模式：单个 agent 产出看起来合理、实则没接线或不一致的 pipeline。Fix：蓝图、逐 node 生成、组装/校验分开。

## 4. 评估：用户感觉 + 专家评分，双轨才可信

**Within-subjects study（被试内研究）**：12 个参与者每人用和不用 assistant 各做一次任务，每人当自己的对照。度量主客观结合：NASA-TLX（感知 workload，工作量）、system-usability 问题（信心）、blind expert rating（盲评专家打分，1–5，pipeline 质量）、recorded session + interview（录制 + 访谈，定性洞察）。

结果：感知到的 mental / physical demand 更低，信心更高，pipeline 更好 —— 用 assistant 的用户远更可能包含关键 Evaluator Node，这是 planner 蓝图的直接结果。访谈：解决 blank-page 问题（8/12）、帮 refine 想法（6/12）、超出预期（10/12）。Surprising finding（意外发现）是 perception–reality gap（感知-现实差距）：专家评 assisted 工作明显更好，但用户自己并没觉得成功多少 —— **Dunning–Kruger bias（达克效应）** 的例证，人会误判自己的质量。

这暗示两个风险：overreliance（无批判接受次优输出）、bias（无批判采纳 assistant 的 pattern）。所以 ChainBuddy 的关键设计选择：输出可编辑起点，不是锁定最终答案，人是最终仲裁者。评估 assistant 时用双轨（主观 + 客观）。别只信用户满意度。例：用户觉得“差不多”，但他的 pipeline 现在有了以前漏掉的 evaluation node。失败模式：发一个用户盲目信任的 assistant。Fix：输出可编辑、展示 reasoning、要求人工 review。

## 这门课怎么教的（Module 6）

先给 relatable 痛点 —— 盯着空白画布 —— 再按 ChainBuddy 自己的两阶段流：clarify → generate → 证明有效。Running example 是单个工具（ChainForge / ChainBuddy）贯穿，planner / worker / orchestrator 团队用人团队类比解释。心智模型：copilot 当 starter + editor，不是 autopilot。每课对应一个阶段，评估课把结果绑回架构选择，好跟。难处：评估课一次引入几个指标（TLX、SUS、专家评分、访谈），没先排哪个指标回答哪个问题。最值得偷的：**用用户感知 + 独立专家质量双轨评估一个 assistant，再解释两者的差距 —— 把“它有效”的断言变成关于 overreliance 的设计课**。要避免：只给 usability 分数当成“有帮助”的证明。

## 自测 4 问（带答案要点）
1. ChainBuddy 两阶段是什么？核心设计选择？
   → chat assistant 收集需求 → multi-agent 生成 pipeline；输出可编辑起点，不是锁定最终答案。
2. 需求收集三阶段 + 两个保不跑偏的机制？
   → 理解任务 → 定评估标准 → 生成规约；intent inference + state machine。
3. Planner / Workers / Orchestrator 各干什么？为什么不让一个 agent 全干？
   → Planner 写蓝图；Workers 各做一种 node；Orchestrator 组装 + 校验；单 agent 会产出看似合理、实则没接线/不一致的 pipeline。
4. 为什么评估要用双轨？Perception–reality gap 说明什么？
   → 用户感觉 + 专家评分；专家评更好但用户没觉得 → Dunning–Kruger；只信满意度会漏 overreliance 风险。
