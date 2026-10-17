# Foundations（Lessons 1–4）：从 RAG 到 Agentic RAG

> 这一章回答一个问题：标准 RAG 缺什么？答案：它是个固定 pipeline（流水线），不会规划、不会用工具、不会澄清。Agentic RAG 把 RAG 降级成一个工具，让 LLM 当推理引擎，规划并调用它。读法：先看标准 RAG 在哪三类问题上翻车，再看 agent 的四个部件，最后看推理策略的 trade-off（权衡）。

## 1. 从 RAG 到 Agentic RAG：固定 Pipeline vs 会规划的研究助理

**LLM 的局限**：frozen knowledge（冻结知识）、hallucination（幻觉）、没有 private / domain knowledge（私有/领域知识）。**RAG（检索增强生成）** 像 open-book exam（开卷考试）：先 retrieve（检索），再从 context（上下文）里回答。

两阶段：
- **Offline indexing（离线索引）**：load（加载）、chunk（切块）、embed（向量化）、store vectors（存向量）。
- **Online retrieval / generation（在线检索/生成）**：embed query（向量化问题）、similarity search（相似度搜索）、augment prompt（增强 prompt）、generate grounded answer（生成有依据的回答）。

例：HR PTO（带薪假）问题 —— 检索到的 policy chunk（政策片段）拼进 augmented prompt，回答有依据。

**标准 RAG 的三类翻车**：
- **Multi-step comparison（多步比较）**：如比较 PPO vs HMO，固定 pipeline 不会分开查、再并排综合。
- **Live tool（实时工具）**：如“当前股价 + PDF 财报”，RAG 没有实时工具。
- **Ambiguity（歧义）**：如三个都叫 Titan 的项目，RAG 不会澄清。

Fix 是 agency（能动性）：LLM 当 reasoning engine（推理引擎），规划并用工具，RAG 只是其中一个工具。类比：标准 RAG 是 fast librarian（快图书管理员）；agent 是 skilled research assistant（熟练研究助理），查图书馆、新闻、计算器，再综合。Agentic 例子：按 plan（计划）分别查 RAG，保留两个 context，并排综合。

什么时候用 Agentic RAG：问题需要多步、实时数据、或澄清。简单事实问题用标准 RAG 就够。失败模式：给固定 pipeline 一个它不会规划的问题。Fix：让 LLM 规划，把 RAG 当工具调。

## 2. Agent 的解剖：Brain + Tools + Planner + Memory

四个部件，用 film-director（电影导演）类比：
- **Brain（大脑，LLM）**：像 CPU / raw processing（原始处理）。理解 intent（意图）、decompose problem（拆解问题）、决定何时 retrieval（检索）、选工具、综合输出。
- **Tools（工具）**：训练数据之外的能力；RAG 是 primary（主要），外加 supplementary（补充）。类别：data retrieval（数据检索：RAG、SQL）、live information（实时信息：API 调用、天气）、computation（计算：code interpreter、financial model）。工具本质是 function call（函数调用）；name + description（名字 + 描述）对 planner 至关重要。
- **Planner（规划器）**：orchestration logic / prompt（编排逻辑/提示）；循环：analyze goal（分析目标）→ choose tool（选工具）→ execute（执行）→ observe（观察）→ 重复。ReAct 是其中一种实现。
- **Memory（记忆）**：short-term scratchpad（短期便签：对话 + thought-act-observe 历史）vs long-term knowledge base（长期知识库：vector store；原型用 in-memory index，生产用 ChromaDB / Weaviate / Pinecone）。

**Loop（循环）**：目标未达成时 —— think（想）→ planner 选工具 → execute → 用 thought / action / observation 更新 memory。

缺哪个会怎样：缺 brain，复杂任务处理不了；缺 tools，brain in a jar（罐中之脑）；缺 planner，chaotic / inefficient（混乱/低效）；缺 memory，多步任务做不了。

课程实现用 Groq API + Llama 3（速度/稳定性/可复现），提到 Llama-4 Scout MoE 选项（最长 131K token）但仍用 Llama 3。

## 3. Agentic 推理策略：ReAct、Reflection、Plan-and-Execute、Multi-Agent

**ReAct（Reason + Act，推理 + 行动）**：prompting strategy（提示策略），强制 LLM 行动前先大声思考。Master prompt（主提示）：persona / goal（角色/目标）、tool listing（工具清单：名字 + 描述）、formatting instruction（格式指令：Thought / Action / Action Input）、memory / scratchpad（之前循环）。

Trace 例子：“什么是 Transformer，谁是主要作者？” Cycle 1 用 definition_tool 查 Transformer；Cycle 2 用 local_rag_tool 查《Attention Is All You Need》作者（Vaswani、Shazeer、Parmar、Uszkoreit、Jones、Gomez、Kaiser）；synthesis（综合）合并两者。

好处：debuggability（可调试）、reduced hallucination（少幻觉）、adaptability（适应性）。现实里：ChatGPT Research mode 是同一循环；Gemini function calling = act，API 响应 = observation。

**Self-correction / reflection（自我纠正/反思，Reflexion 式）**：draft（草稿）→ 第二次 LLM 调用按 context 批评 → refine（修正）。例：草稿说 $50M 显示 strong growth（强劲增长），批评指出 context 没说 strong，修正成只说 $50M。

**Plan-and-execute（先规划再执行）**：先写完整 plan，顺序执行，只在必要时 replan（重新规划）。可预测 workflow 更适合；ReAct 更适合不可预测/研究型。Trade-off：adaptability vs efficiency。例：Q1 vs Q2 对比的 plan：RAG Q1 → RAG Q2 → calculator 算百分比变化 → 综合。

**Multi-agent（多 agent）**：specialist 团队胜过一个 generalist（通才）；lead architect / project manager（主架构师/项目经理）、research / surveying（研究/调查：RAG + API）、structural engineer / synthesizer（结构工程师/综合者）、compliance / QA（合规/质检）。把 retrieve-augment-generate 拆成角色。优势：specialization（专业化）、modularity（模块化）、sophistication（复杂能力）。

**对比表**：
- ReAct：dynamic / adaptable（动态/适应），简单任务上慢。
- Self-Reflection：high-accuracy / self-correction（高精度/自纠），extra cost（额外成本）。
- Plan-and-Execute：predictable / efficient（可预测/高效），rigid（僵硬）。
- Multi-Agent：complex / modular（复杂/模块化），coordination overhead（协调开销）。

为什么不用 frontier model（前沿模型）直接答？简单 trivia（冷知识）可以；但可扩展、可靠、可审计的系统需要 ReAct，尤其 private data、real-time data、complexity / reliability（小可验证步骤、易调试）。

## 4. Basic RAG vs Agentic RAG（LlamaIndex）：把 Pipeline 提升成 Tool

**LangChain**（LangGraph 做 stateful graph workflow，有状态图工作流）vs **LlamaIndex**（RAG 出身，强 ingestion / indexing / retrieval + agentic 层）。课程用 LlamaIndex，因为 data-centric（数据中心）。

关键概念：
- **Query engine（查询引擎）**：完整的、可运行的 RAG pipeline（VectorStoreIndex、query_engine.query）。
- **Tool（工具）**：给 agent 一个能力的 wrapper（包装）（QueryEngineTool；description 是关键信号）。

课程用 invented data（虚构数据）证明 retrieval：虚构 Project Cygnus，Innovate Inc. 的 AI analytics platform（分析平台），负责人 Dr. Aris Thorne，beta 目标 Q4 2025 面向企业金融机构；Python / Rust 微服务、distributed graph DB、Kafka、React PWA；两个文本文件在 data/。

**Static RAG（静态 RAG）**：load_dotenv、Settings.llm OpenAI gpt-5、Settings.embed_model OllamaEmbedding nomic-embed-text、SimpleDirectoryReader ./data、VectorStoreIndex.from_documents、as_query_engine similarity_top_k=1，查询“Who is lead of Project Cygnus?”

**Shift（转变）**：把 RAG pipeline 提升成工具；book-reading machine（读书机器）变成 hired research assistant（雇来的研究助理）的能力。Agentic 代码：QueryEngineTool + ReActAgent，project_cygnus_rag_tool，ReActAgent verbose=True，async agent.run 经 asyncio。

Trace：第一次 OpenAI 调用 = thought / tool choice；Ollama embed + RAG 工具内的 OpenAI = act；最后一次 OpenAI 调用 = 检查 observation（Dr. Aris Thorne）并给出最终答案。

对比：Static = monolithic（单体）、单查询、不灵活、难扩展；Agentic = ReActAgent 编排 Tool、conversational run（对话式运行）、灵活、易扩展。

## 这门课怎么教的（Module 1）

Problem-first、layer-by-layer（问题优先、逐层）：先给熟悉的 pipeline，再用具体失败暴露局限，再加 agency，再 dissect（解剖）agent，再比较推理策略。好懂处：analogy-heavy（类比密集：开卷考试、图书管理员 vs 研究助理、电影导演、CPU vs OS）、concrete scenario（具体场景：HR PTO、PPO vs HMO、Titan、Transformer / 作者）、worked trace（完整追踪）、comparison table（对比表）。Show Answer 预判困惑（brain vs planner、为什么不直接用 frontier LLM）。最值得偷的：**先给 relatable failure（可共鸣的失败），每个概念一个类比、反复用，展示完整 trace，最后用 trade-off 表收尾**。要避免：具体例子之前堆抽象术语；课程每个术语先在场景里落地。

## 自测 4 问（带答案要点）
1. 标准 RAG 在哪三类问题上翻车？Agentic RAG 怎么修？
   → 多步比较、实时工具、歧义；LLM 规划 + RAG 当工具调。
2. Agent 四个部件各是什么？缺哪个会怎样？
   → Brain / Tools / Planner / Memory；缺 brain 不能处理复杂任务，缺 tools 是罐中之脑，缺 planner 混乱低效，缺 memory 做不了多步任务。
3. ReAct / Reflection / Plan-and-Execute / Multi-Agent 各适合什么？
   → ReAct 不可预测/研究型；Reflection 高精度但额外成本；Plan-and-Execute 可预测 workflow；Multi-Agent 复杂但协调开销。
4. Query engine 和 tool 的区别？为什么 description 是关键信号？
   → Query engine 是完整 RAG pipeline；tool 是给 agent 能力的包装；planner 靠 description 选工具。
