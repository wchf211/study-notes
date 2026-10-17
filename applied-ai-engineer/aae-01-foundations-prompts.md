# Foundations + Prompts（Modules 1–3）

> 这门课的起点不是“怎么调模型”，而是“怎么把一个 LLM 功能当生产系统交付”。角色定位：prompt → RAG → agent → fine-tune → eval → deploy → safety，一条端到端 storyline。

## 1. LLM 直觉：为什么措辞、截断、长上下文都重要

**LLM（大语言模型）**：按 pretraining 学到的模式预测下一个 token，再经 fine-tuning / RLHF（人类反馈强化学习）塑形。**Context window（上下文窗口）**：一次能塞进模型的 token 上限；**tokenization（分词）**：文本被切成 token 再处理。

这两点决定了三件实事：措辞变了，输出可能变；历史塞满了，会被静默截断；长上下文不是免费的，每 token 都是成本。失败模式：把模型输出当成检索到的事实。Fix：把它当成“基于模式的生成”，关键处用检索 / 校验兜底。

## 2. 选型：不是看榜单，是看约束

选模型要比的维度：capability（能力）、cost（成本）、latency（延迟）、privacy（隐私）、control（可控程度）。Closed vs open、general vs specialized、hosted vs self-hosted，都按这五个维度打分。

失败模式：默认选最大的。Fix：按任务约束 + 实测选。榜单是起点，不是决策。

## 3. 成本 / 延迟预算：一个方程管住三件事

核心耦合关系：总延迟 / 总花费 \~ 调用次数 ×（输入 + 输出 token）。**p95（95 分位延迟）**：95% 的请求都快于这个值，是生产里比平均值更有用的延迟指标。

做法：给 input token、每 action 的调用数、p95 设预算；用 truncation（截断）/ summarization（摘要）/ retrieval（检索）控输入；用 streaming（流式输出）改善体感；用 output caps（输出上限）防失控。例：课程场景里的目标是首答 p95 < 1s + 每次交互花费上限。失败模式：把完整历史全塞进去，然后被静默截断。Fix：预算先行，截断要显式、可见。

## 4. 环境搭建：安全脚手架

API key 处理、env config（环境配置）、最小 client、第一次 test call、error / timeout 基础。失败模式：hardcode key。Fix：key 走环境变量 / secret 管理，绝不进代码仓库。

## 5. Prompt：System vs User 是安全边界

**System prompt（系统提示）**：持久的角色 / 规则。**User prompt（用户提示）**：本次任务。**Instruction hierarchy（指令层级）**：system > user > tool 内容，低层不能覆盖高层。

这是安全边界，不是风格问题：粘贴进来的文档、工具返回的文本，都是 untrusted（不可信输入），绝不能覆盖 system 规则。失败模式：文档里的 prompt injection（注入指令，如“忽略你的规则”）劫持行为。Fix：层级 + 输入检查，把 tool / 文档内容当数据，不当指令。

## 6. Few-shot 与 Chain-of-Thought：什么时候加，什么时候不加

**Few-shot（少样本）**：给几个 input→output 例子，让模型学格式 / 模式。**Chain-of-Thought / CoT（思维链）**：让模型先写中间步骤再给答案，帮助多步 / 分类任务。

用的时候：格式或推理不明确时。别用的时候：CoT 会加成本 / 延迟，或泄露推理过程。失败模式：few-shot 例子本身教错了 pattern。Fix：例子要覆盖正确 pattern，且定期检查例子是否还在教对的东西。

## 7. Structured Output + Function Calling：让下游敢解析

把输出约束成 schema / tool call，而不是让下游 regex（正则）去猜自由文本。**Function calling（函数调用）**：模型输出结构化调用，代码校验、执行、返回结果。

失败模式：假设 JSON 一定合法。Fix：validate（校验）+ 处理 refusal（拒绝）/ missing args（缺参数）。

## 8. Prompt 测试与版本管理：Prompt 当代码管

Prompt 要 version（版本化）、配 regression set（回归集）、改动要测试、记录哪次输出用的是哪个 prompt / 模型。失败模式：线上 prompt 改了，没做 before / after 对比。Fix：改 prompt = 改代码，走同样的测试流程。

## 这门课怎么教的（Modules 1–3）

Module 1 用生产岗位故事 hook，每阶段一句话 outcome。Module 2 用一个带延迟 / 成本上限的聊天功能 hook，顺序是机制 → 选型 → 预算 → 搭建，最值得偷的是耦合预算方程（总花费 \~ 调用数 × token）。Module 3 用一个 misbehaving 分类器 / 客服 bot hook，顺序是层级 → 例子/推理 → 结构 → 测试，最值得偷的是“层级图 = 安全边界”。小坑：structured outputs + function calling 挤进一课，setup 课在理论之后显得像挂件。

## 自测 4 问（带答案要点）
1. 为什么措辞、截断、长上下文都会影响 LLM 输出？
   → 预测 next token + context window + tokenization；历史满了会静默截断，长上下文按 token 收费。
2. 选模型比哪五个维度？为什么不默认选最大？
   → capability / cost / latency / privacy / control；按任务约束 + 实测选。
3. 成本 / 延迟预算方程是什么？控输入的三招？
   → 总量 \~ 调用数 ×（输入 + 输出 token）；truncation / summarization / retrieval + streaming + output caps。
4. Instruction hierarchy 为什么是安全边界？Few-shot / CoT 什么时候别用？
   → system > user > tool，低层不能覆盖高层，防 injection；CoT 加成本 / 延迟 / 泄露推理时别用。
