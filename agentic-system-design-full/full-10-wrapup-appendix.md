# Wrap-up + Appendix（Modules 12–13）：收尾清单 + 参考架

> 附录不是“可选读物”，是全课最可复用的部分：glossary 查术语、cheatsheet 选 pattern、landscape 选框架。读法：先看收尾五原则，再把附录当工具书用。

## 1. Final Thoughts：五原则 + 上线前 Checklist（Module 12）

收尾回到核心决策框架：从任务和约束出发，不是模型 —— 按工作选架构（单 vs 多 agent、循环 pattern、记忆、guardrail），再选工具和模型。五原则反复出现：

1. **Architecture matters more than the raw LLM（架构比裸模型重要）**。
2. **Iterate with observation and feedback（用观察和反馈迭代）**：reflection、evaluation、human input。
3. **Design for errors and failure（为错误和失败设计）**：recovery、exit condition、graceful degradation（优雅降级）。
4. **Keep humans in the loop for consequential actions（有后果的动作留人在环）**。
5. **Evaluate rigorously（严格评估）**：task success + human judgment，不只 fluency（流利度）。

信息是 agentic design 是 engineering（工程）—— trade-off、measurement、iteration —— 不是 prompt magic（prompt 魔法）。任何 agent 上线前用这个 checklist。失败模式：追更大模型来修架构问题。Fix：先修 loop、memory、grounding、evaluation。

## 2. Glossary：术语是设计工具，不是定义（Appendix 1）

**Glossary（词汇表）**：核心术语参考 —— agent、LLM agent、tool、memory 类型、planning、orchestration pattern、reflection、grounding、guardrail、human-in-the-loop / on-the-loop / in-command、evaluation 概念 —— 按可查的方式组织。课文还含一个 expanded answer（展开答案），把词汇应用到一个 worked example（设计一个 assistant），强化“术语是设计决策的工具，不只是定义”。

用法：术语模糊时查。别把词汇表当正课读 —— 它是工具书。

## 3. Cheatsheet：按任务形状选 Pattern（Appendix 2）

**Cheatsheet（速查表）**：一页式地图：核心组件（model、tools、instructions、memory）、架构（single-agent、multi-agent、planner-executor）、workflow pattern（prompt chaining、routing、parallelization、evaluator-optimizer、ReAct、plan-and-execute、manager-worker、handoff），配 when-to-use（何时用）指导和课程例子（MACRS、Eureka、ChainBuddy、WebVoyager）。

用法：按任务形状选 pattern。Expanded answer 把速查表应用到一个 worked design question（设计题）。别在没想清任务形状时翻速查表 —— 先定任务，再选 pattern。

## 4. Landscape Guide：选框架 + 预见生产风险（Appendix 3）

**Framework（框架）**：
- LangChain / LangGraph：broad integration（广泛集成）、stateful graph（有状态图）、steep learning curve（陡学习曲线）。
- AutoGen：conversable multi-agent debate（可对话多 agent 辩论），单 agent 不太适合。
- CrewAI：role-based crew（基于角色的团队），business process（业务流程）易上手，低层控制较少。
- Google ADK：structured（结构化）、Google Cloud-native（Google 云原生）、ecosystem lock-in（生态锁定）。
- 另有 OpenAI Agents SDK、Semantic Kernel。

**选型问题**：single vs multi-agent、integration 需求、团队经验、cloud 目标、对循环的控制程度。**应用**：coding assistant、customer experience automation（客户体验自动化）、multimodal web agent、automated data pipeline（自动数据管道）、scientific research（科学研究，Eureka 当例子）。**工程挑战**：reliability / hallucination、latency / cost、observability / debugging、safety / minimal footprint（最小权限工具）。

Worked answer 把 freight-quotation workflow（货运报价工作流）映射到能力/pattern（perception、ReAct + tools、human-in-the-loop routing），推荐 CrewAI（role-based business process fit）。用法：选框架 + 预见生产风险。别只按流行度选框架 —— 按选型问题逐项对。

## 这门课怎么教的（Modules 12–13）

Module 12 是短 recap，把每个原则映射回课程 case（Eureka reflection、WebVoyager recovery、ChainBuddy editable output）。最值得偷的：**用已见过的例子绑定的可复用 checklist 收尾，不讲新理论**。

Module 13 当参考架，不是新章节 —— glossary 查术语、cheatsheet 选 pattern、landscape 选框架。每个都以一个 worked application question（Show Answer 展开）结尾，强迫读者用参考资料。最值得偷的：**用一个“选 pattern / 框架”的 worked 练习收尾，复用前面 case**。要避免：把附录当可选 —— 它是真实设计工作里最可复用的部分。

## 自测 4 问（带答案要点）
1. 收尾五原则？
   → 架构 > 裸模型；观察+反馈迭代；为错误/失败设计；有后果动作留人在环；严格评估（task success + 人评，不只流利）。
2. Glossary / Cheatsheet / Landscape 各怎么用？
   → Glossary 查术语；Cheatsheet 按任务形状选 pattern；Landscape 选框架 + 预见生产风险。
3. 选框架问哪五个问题？
   → single vs multi-agent、integration 需求、团队经验、cloud 目标、对循环的控制程度。
4. 为什么附录不是可选读物？
   → 真实设计工作里最可复用；每个都以 worked 练习结尾，强迫用参考资料做决策。
