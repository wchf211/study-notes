# Agent 地基（Ch1）：什么算 agentic、架构、生产可信

> Agent ≠ 模型。模型是计算器（输入→输出）；Agent 是私人助理（理解意图、规划步骤、执行、看结果再调整）。区分两者的就是 perceive–reason–act 循环。

## 1. 什么让系统算 agentic

**AI agent**：感知环境、自己做决定、自主行动去达成目标的系统，跑 **perceive–reason–act 循环**（感知→思考→行动→看结果→重复；出自 Russell & Norvig 教科书的经典表述）。没有这个循环，系统就是一锤子买卖，条件一变就傻。

三能力，缺一不可（缺了会怎样）：
- **Perception（感知）**：从原始输入（文本/图像/音频/传感器/DB 行）提取可用意义。缺了=瞎，决策没有现实依据。
- **Reasoning & planning（推理规划）**：解读上下文、选动作、排步骤。LLM 提供语言推理。两种模式：reactive（来啥回啥，快而简单）vs deliberative（先规划多步再动手，慢而能处理复杂目标）。
- **Action execution（行动）**：回话、调 API、写 DB……动作必须 grounded 在推理上，不是乱动。

例：旅行 concierge 发现去柏林的航班取消（感知）→ 旅客今晚还得到（推理）→ 查备选航班+火车、订最优组合、通知酒店晚到（多步行动）。

五个定义特征：autonomy（不需人一直指挥）、goal-oriented（奔着目标不是孤立输出）、perception-feedback loop（持续观察调整）、continuity（跨 turn 记 context，多步推理才成立）、flexibility（context 变了改计划）。

**什么时候才值得造 agent**：agent 贵（复杂度/成本/不确定性），规则引擎能搞定就别上。三个信号：(1) 需要 contextual judgment——模糊输入、trade-off、依赖历史（如退款审批看客户历史+语气+退货理由）；(2) 规则集已经烂到维护不动（几十个分支还在长）；(3) workflow 跑在非结构化自然语言上（文档/邮件/对话）。决策规则：**只有问题真需要动态决策、演化 context、或对非结构化数据推理，才上 agent**。

## 2. 架构：model + tools + instructions + 控制循环

现代 agent 是三个模块化组件，被一个循环串起来：

**Model（模型）**：中央推理机（通常 LLM）。选型四轴 trade-off：任务复杂度（抽取/摘要小模型如 Mistral/Gemma 够；规划判断要大模型）、latency（实时交互要快模型）、cost（大模型按 query 烧钱；常见 pattern：小模型跑常规步，大模型只上难的）、context window（长对话/长文档要长上下文模型）。

**Tools（工具）**：让 agent 摸到推理之外的世界——API（天气/支付/消息）、DB（用户画像/交易）、搜索引擎（训练数据之外的鲜活事实）、计算器（模型算不准的精确算术）、文件系统。例：查天气调天气 API，不信模型脑子里过期的知识。**Tool use** = agent 自己决定在任务中途调外部函数/服务。

**Instructions**：行为 spec——自然语言任务 prompt、system message（角色人设）、tricky 场景的 worked examples、约束（伦理边界/输出格式）、task policies。写好它叫 prompt engineering，是核心设计技能不是边角料。

**控制循环**（组件≠agent，循环才是）：Perceive（收输入）→ Recall（捞记忆）→ Reason（定下一步）→ Act（回话/调工具）→ Store（结果写回记忆）→ 重复。每轮都 sharpen context 和决策质量。

**Memory 三种**（没记忆的 agent 是失忆的）：short-term（当前 session，如两条消息前用户说了啥）、long-term（跨 session：偏好/设置/历史决策）、external（外部查：文档/知识库/API；通常语义检索——内容转 **vector embeddings**（文本语义的数字表示）存 **vector database**（按向量相似找最相关段落的存储），推理前先捞相关 context）。

**Orchestration patterns**（多步工作怎么编排）：
- 单 agent：tool-calling loop（要不要调工具→调→看结果→继续推理）；**ReAct**（Reason+Act 交替：显式推理一步、执行一步，证据到了就修正）；plan-and-execute（先出完整结构化计划再按序跑——序列可预测时更好）。
- 多 agent：manager–worker（中央拆任务派给专家）、decentralized handoff（agent 按 context 把控制权传给合适的 peer）。
- 多 agent 买模块化和专业分工，付通信同步 overhead。框架（LangChain/AutoGen/CrewAI）把这些 pattern 打包，省得重造 plumbing。

单 vs 多决策：范围窄、工具少、决策集中→单；需要真·不同专长、workflow 能干净切模块、要 oversight/仲裁/角色分离、scale 要分布→多。

## 3. Trust、可靠性、生产设计

LLM agent 的失败是传统软件没有的，生产设计默认不确定性，外面建 containment。

**七类不信任来源**：nondeterministic outputs（同样输入不同输出）、prompt sensitivity（改一个词行为大变）、**hallucinations**（流利但编的——模型不知道自己不知道）、limited context（长 session 前面忘了）、emergent behavior（多 agent 踫出没人设计的交互，难预测难 debug）、安全暴露——**prompt injection**（输入里藏"忽略你的规则"劫持指令）、**jailbreaks**（绕过安全拒绝的技巧）、**data exfiltration**（敏感数据经 tool/输出外泄）、opacity（推理链难检查，故障难解释）。

**六设计原则**：alignment（行为对齐用户意图）、robustness（意外输入优雅降级不崩）、transparency（可解释可追溯）、safety（无有害副作用）、controllability（人能介入/覆盖/叫停）、privacy/security。**Human-in-the-loop**（高风险步骤人先审再执行）是 controllability 的标准件。

**为安全失败而设计**（心态转变：默认会挂，bound blast radius）：graceful degradation（ shedding 功能不崩——答一半不答零）、rollback（回 last known-good；coding agent  revert 坏 commit 并告警而不是把 damage 推下去）、fallback to human（置信度低/赌注大转人工）、error surfacing（告诉用户挂了——静默失败会复利）。

**生产 checklist**：目标约束先写死（agent 允不允许干这个）；**guardrails**（运行时检查：输出校验/PII 脱敏/策略拦截）；**action sandboxing**（动作在隔离环境执行，错了摸不到真系统）；validation layers（输入/tool 输出/最终答案的 schema 完整性检查）；全量 logging/monitoring（每个决策和 tool 调用可追溯——看不见就没法 debug）；真实+对抗场景测试；gradual rollout（先窄后宽）；feedback loops。上面还有 governance：合规（GDPR 类）+ auditability（干了啥为啥干的完整记录）。

**可靠性度量**：task success rate、error rate、latency、user satisfaction——上线前定义。**LLM-as-judge**（另一个模型按 rubric 打分——没 ground truth 时有用，但 judge 自己有 bias）+ 高风险人工评。

## 自测 4 问（带答案要点）
1. Agent 和 model 的本质区别？什么情况下不该造 agent？
   → 循环（perceive–reason–act）+ autonomy/memory/goals/actions；规则引擎能搞定时别上。
2. 三组件 + 循环各是什么？ReAct vs plan-and-execute？
   → model/tools/instructions；循环六步。ReAct 交替推理执行（证据驱动），plan-and-execute 先计划后执行（序列可预测）。
3. 三种 memory？external memory 怎么工作？
   → short/long/external；语义检索：embedding→向量库→推理前捞。
4. 生产 checklist 哪几件？"为安全失败设计"四机制？
   → 目标约束/guardrails/沙箱/validation/logging/测试/灰度/反馈+治理；降级/回滚/转人工/报错不静默。
