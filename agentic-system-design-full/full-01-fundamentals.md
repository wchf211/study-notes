# Fundamentals（Module 1）：Agent 地基

> 核心图景：agent = 感知 → 推理 → 行动 → 看结果 → 调整的循环（perceive–reason–act loop），而不是一次回答。这一章把零件、记忆、编排、安全、生产挑战一次性立起来，后面所有 case 都是这些零件的组合。

## 1. 什么算 agent

**Agent**：用 LLM 做推理、能感知环境、调工具、多步朝着目标工作的系统。**Bare model（裸模型）**：只回答 prompt，不规划、不行动、不验证。

三类：simple reflex agent（简单反射：固定 if-then 规则）、learning agent（学习型：从 reward / 经验改进）、LLM agent（语言模型推理 + 工具 + 记忆）。

什么时候用 agent：多步、需要实时数据/工具、含糊、要中途调整。什么时候别用：单次固定答案、确定性计算、完全可预测 workflow —— 脚本或单次模型调用更便宜可靠。例：假期规划助手查日历、搜航班酒店、航班满了改计划，而不是给一份静态行程。失败模式：把 chatbot 当 agent —— 流利回答，从不行动/验证。Fix：给工具、状态、反馈循环。

## 2. 三组件：Model + Tools + Instructions

**Model（模型）**：推理大脑。**Tools（工具）**：可调用的能力（搜索、计算器、API、DB），必须类型化描述：名字、用途、输入、输出 —— 模型只知道描述里写的东西。**Instructions（指令）**：角色、目标、约束、工作流规则。

选型：模型看能力 vs 成本/latency、context 大小、tool-calling 可靠性、structured output 需求；工具看可靠性、延迟、成本、安全；指令要具体、定允许的工具集、写明不确定时怎么办。别把 prompt 和工具逻辑硬编码搅在一起，也别给无边界的工具集。例：假期 planner 用中档模型 + 航班/酒店搜索工具 + “预算内、订之前先问”指令。失败模式：工具描述含糊 → 调错 / 格式错。Fix：紧 schema + 例子 + validation。

## 3. 交互与记忆：循环 + 三种记忆

循环：perceive（输入、对话历史、工具结果、传感器数据）→ retrieve 相关记忆 → reason / plan → act（调工具）→ store 结果。单次生成变成有状态、能改进的系统。

**三种记忆**：
- Short-term / working memory（短期/工作记忆）= 当前 context window，快但有限，session 结束就丢。
- Long-term memory（长期记忆）= 持久事实、偏好、过去轨迹，常存成摘要或 vector store（向量库）。
- External memory（外部记忆）= 实时知识库/文档，按需检索。

Long-term / external 的检索通常是 embedding（向量嵌入）+ 语义相似搜索，再把找到的内容 grounding（扎到证据上）给模型。外部记忆适合事实会变 / 超出 context 时；摘要让长期记忆保持紧凑。别把全部历史塞进每个 prompt，也别把记忆当永远正确 —— 它会过期、会错。例：假期 agent 记得 aisle-seat 偏好（长期记忆），查实时时刻表（检索）。失败模式：context 无界膨胀，agent 忘事 / 自相矛盾。Fix：摘要 + 保留/过期策略 + 校验检索到的事实。

## 4. 编排模式：谁干什么、按什么顺序、错了怎么办

基本循环：解读 goal → 选下一步 action / tool → 执行 → 观察 → 重复，直到完成，配显式停止规则。

- **ReAct**：推理 trace 和 action 交替，边做边自我修正。适合探索性单 agent 工作。
- **Plan-and-execute（先计划后执行）**：先写计划，再按步骤跑，某步失败改计划。适合长、可拆的工作，但启动慢。
- **Manager-worker（管理者-工人）**：coordinator 拆工作、分派给专科 worker、合并结果。适合并行、专科工作。
- **Decentralized handoff（去中心化交接）**：任一 agent 直接把控制权传给下一位专科，无中央老板。更灵活，但要清晰的交接规则。

简单线性任务别用多 agent pattern —— overhead 和协调失败超过收益。例：规划旅行：ReAct 边搜边改；manager-worker 让航班、酒店、预算三个 worker 并行。失败模式：循环永不结束 / 重复劳动。Fix：step cap、清晰退出条件、共享状态只有一个 owner。

## 5. Guardrails + Human Oversight：安全是分层，不是单个过滤器

没有检查的自主会造成：烧钱、错动作、泄露隐私、偏见输出。检查要铺在整个流程：input filtering（输入过滤：prompt injection、jailbreak）、intent classification（意图分类，分流高风险请求）、tool-use controls（工具控制：allow-list、参数校验、限流、least privilege 凭证）、output validation（输出校验：格式、事实、政策、毒性）、PII redaction（PII 脱敏）。

**三种 oversight（监督）**：
- Human-in-the-loop（人在环内）：关键动作前必须批准。
- Human-on-the-loop（人在环上）：人监控、可干预。
- Human-in-command（人指挥）：人定策略、担责。

给 reviewer 足够 context（agent 提议什么、为什么、批准后发生什么），否则 review 变 rubber-stamping（橡皮图章）。高成本 / 不可逆 / 受监管动作要严格人工批准；可逆低 stakes 用轻监控。别靠模型自我监督，也别加多到系统停摆的审批。例：订票 agent 只在显式批准后订，但可自由重搜。失败模式：prompt injection 骗 agent 外泄数据 / 做特权动作。Fix：独立于模型输出校验输入和工具参数 + 副作用要人工签收。

## 6. 生产挑战：上线前检查表

- Latency（延迟）：大模型 + 链式调用慢 → 小模型路由简单任务、缓存确定性结果、并行独立调用、长任务异步、优化模型/硬件。
- Hallucination（幻觉）：流利但假 → retrieval grounding、输出校验、关键决策 ensemble / voting（集成/投票）、暴露 confidence（置信度）、精确指令。
- Memory：context 漂移/过期 → 结构化 short/long/external 存储、读写权限、选择性保留、versioning / auditing（版本/审计）。
- Scalability（可扩展）：用户/agent 多了原型垮 → 模块化零件、多 agent 分布、load balancing（负载均衡）、精简 tool 调用。
- Integration（集成）：多 API/格式让系统脆 → 标准 tool schema、抽象层、retry / fallback、显式 agent 间协议。
- Security / privacy：injection、jailbreak、泄露 → 分层 input / output / PII / tool 控制、认证、安全部署。
- Evaluation（评估）：动态 agent 没有单一标准指标 → task-success 指标、simulation（仿真）、human-centred review（以人为本评审）、全量 tracing / logging（追踪/日志）。
- Human-oversight overhead（人工监督开销）：review 太多 bottleneck（瓶颈）→ 只升级高 stakes / 低置信 case，给 reviewer 简明摘要，从干预中学习。
- Fault tolerance（容错）：工具/agent 失败会 cascade（级联）→ graceful degradation（优雅降级）、retry with backoff（退避重试）、checkpointing（检查点）、escalation path（升级路径）、error isolation（错误隔离）。

上线前用检查表（latency、reliability、memory、security、oversight、observability、scalability）逐项过。别把“演示跑通一次”当成 production-ready。

## 这门课怎么教的（Module 1）

用具体对比开场（单次模型回答 vs 会行动的假期规划 agent），再层层加：是什么 → 三个零件 → 零件怎么交互/记忆 → 行为怎么编排 → 怎么保安全 → 生产会坏什么。假期规划 running example 贯穿，guardrails / 挑战章配新鲜小例子（客服 bot、金融 agent）。心智模型：perceive–reason–act loop + 三零件解剖图 + 对比表（agent 类型、记忆类型、编排模式）。每课末尾短 recap + 决策指导（何时用/不用），好跟。难处：两节架构课有重叠（组件先介绍一遍，又在交互循环里再介绍一遍）；编排模式密集连讲，没有一张决策树。最值得偷的：**同一个 running example 锚定每个抽象 pattern，再把同一任务用两个 pattern 各解一遍，trade-off 可见而不是断言**。要避免：三个相似 taxonomy 连着堆，不停下来在一个具体任务上比较。

## 自测 4 问（带答案要点）
1. Agent 和 bare model 的本质区别？三类 agent？
   → 循环（perceive–reason–act）+ 工具 + 状态 + 反馈；simple reflex / learning / LLM agent。
2. 三组件各是什么？工具描述为什么必须精确？
   → model / tools / instructions；模型只知道描述里写的东西，含糊就调错。
3. 三种记忆？为什么别把全部历史塞进每个 prompt？
   → short / long / external；context 会爆、会过期、会自相矛盾；摘要 + 保留策略 + 校验。
4. 三种 oversight？生产检查表哪几项？
   → in-the-loop / on-the-loop / in-command；latency / reliability / memory / security / oversight / observability / scalability。
