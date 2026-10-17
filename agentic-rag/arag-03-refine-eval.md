# Refine + Evaluate（Lessons 8–9）：从定性调试到定量评估

> 这一章是“从能跑到可靠”。顺序：先定性（读 trace、诊断失败、修），再定量（golden dataset + RAGAs 打分）。读法：Lesson 8 的 trace 方法是手艺，Lesson 9 的评估框架是尺子 —— 手艺告诉你修哪，尺子告诉你修好了没。

## 1. 调试：Agent Psychologist / Detective（Lesson 8）

**核心认识**：Agent 是 probabilistic（概率性），不是第一次就完美；目标是 diagnose-refine-iterate（诊断-修正-迭代）。

**四种失败模式**，分 cognitive（认知）vs execution（执行）：
- **Incorrect tool selection（选错工具）**：如用 local_rag_tool 答一般生物学问题。
- **Malformed tool input（工具输入格式错）**：如给 definition_tool 一整句。
- **Flawed reasoning（推理缺陷）**：illogical Thought（不合逻辑的思考）、误解、loop（循环）、放弃。
- **Tool execution error（工具执行错误）**：工具代码 crash，如 API 失败。

**Trace analysis（三步）**：(1) 读 Thought，(2) 检查 Action + Action Input，(3) 看 Observation。像 agent psychologist / detective（心理学家/侦探）。

**Scenario（场景）**：“What are four levels of queries in RAG and Beyond?” Agent 盯着“What are...”当定义，调 definition_tool {'term': 'four levels of queries'}，拿到 Wikipedia not-found，放弃。诊断：flawed reasoning → incorrect tool selection；工具本身工作正常。

**Toolkit（三招）**：
1. **Refine tool description（修正工具描述）**：generic → 提到 RAG、eRAG、L1-L4。Explicit keyword（显式关键词）eRAG 和 query level（L1-L4）给 LLM 更强、更清晰的信号；specificity（具体性）降歧义；vague description（含糊描述）可能让 assistant 试 general web search。三个 RAG 工具（A evaluation / eRAG、B data processing / indexing、C multi-agent）时，specific description 让 agent 对“How does eRAG work?”立刻选 A。
2. **Fortify tool logic / robust error handling（加固工具逻辑/健壮错误处理）**：helpful error（有用错误），不 crash；改进 define_term_function 的 DisambiguationError，返回 agent 能用来澄清的选项。
3. **Guide behavior with system prompt（系统提示引导行为）**：constitution（宪法）；工具出错或信息不足时不放弃；re-evaluate（重新评估）、试另一个工具或用自己的知识回答。

**Loop（循环）**：Test-Observe-Refine-Repeat（测试-观察-修正-重复）。

**Walkthrough（完整演练）**：棘手查询“Who is Yann LeCun and what did they write about Transformers?” 起初假设文档里的 Transformers 意味着 local RAG 适用，拿到 observation（文档没有 LeCun），放弃。Fix：把 RAG description 修正成提到 Vaswani 和 Salemi、RAG and Beyond、eRAG。重跑成功：definition_tool 查 LeCun → arxiv_search_tool 查他关于 Transformers 的论文 → 综合。结论：builder（建造者）变成 maintainer / refiner（维护者/修正者）；下一步是定量分析。

什么时候用 trace 方法：任何 agent 行为不对时。别只看最终答案。失败模式：只修最终答案，不修 trace。Fix：按 Thought → Action → Observation 诊断。

## 2. 评估：从“感觉对”到“量出来”（Lesson 9）

**为什么 BLEU / ROUGE 不够**：word-for-word overlap（逐词重叠）对单个预写正确答案。问题：multiple correct answer（多个正确答案，正确表述被扣分）；dynamic data（动态数据，如“找一篇 arXiv 最新论文”的正确答案每天变）；process matters（过程重要，重叠说不出推理/效率/trace）。

传统 IR 指标可能量不出 retrieved doc（检索文档）对 RAG 的用处，因为 LLM 消费信息和人不同；按对最终任务的影响判 retrieval，不只按 topical relevance（主题相关性）。

**框架**：component-level（部件级，单个部件）vs end-to-end（端到端，最终答案）。维度：
- **Task success（任务成功）**：pass / fail 或 F1 / exact match（EM，精确匹配）。
- **Efficiency / cost（效率/成本）**：latency、LLM 调用/token。
- **Reasoning quality / trace evaluation（推理质量/trace 评估）**：逻辑/高效吗？中间步骤、reasoning-retrieval iteration（推理-检索迭代）。

按阶段分失败：retrieval（不完整/不相关）vs generation（不准确/离题）。

**Golden dataset（黄金数据集）**：标准化测试，能从知识库回答；evaluation_dataset.jsonl / golden_dataset.jsonl，每行 JSON query + ground_truth，来自 RAG and Beyond survey（Zhao 等）和 Evaluating Retrieval Quality（Salemi 和 Zamani）。四条：
- **四级查询**：Explicit Fact L1（显式事实）、Implicit Fact L2（隐式事实）、Interpretable Rationale L3（可解释理由）、Hidden Rationale L4（隐藏理由）。
- **三种数据注入形式**：Context、Small Model、Fine-tuning。
- **eRAG**：每个文档单独被 LLM 利用，下游性能 per document 当 relevance label（相关性标签）。
- **eRAG 好处**：改善 runtime，比 end-to-end 少最多 50x GPU memory。

**RAGAs（Retrieval-Augmented Generation Assessment，检索增强生成评估）**：开源、LLM-graded（LLM 打分）；先独立评估 rag_research_paper_tool。四个指标：
- **Faithfulness（忠实度）**：答案 grounded（扎根）在 context；提取 statement（陈述），对照 context 验证。
- **Answer relevance（答案相关性）**：答案回应问题；LLM 从答案生成潜在问题，和原问题比相似度。
- **Context precision（上下文精确度）**：retrieved context 相关/切题；signal-to-noise（信噪比），有用信息排得高。
- **Context recall（上下文召回）**：所有必要的 ground-truth 信息都在 retrieved context 里。

evaluate.py（只读，没跑）：pip install ragas datasets；import evaluate + Faithfulness、AnswerRelevancy、ContextPrecision、ContextRecall；wrapper 让 LlamaIndex OpenAI LLM 和 OllamaEmbedding 兼容；import get_rag_tool。create_rag_eval_dataset 打开 jsonl 成 golden，初始化 tool，迭代，调 rag_tool，try/except 提取 recall / precision 需要的 context，追加 question / answer / contexts / ground_truth，可选 EVAL_LIMIT 切片，转成 Dataset。build_judge_llm：OpenAI max_tokens=256、temperature=0（更快、更便宜、更确定性），包进 LlamaIndexLLMWrapper。main 定义 metric（judge LLM），设 OllamaEmbedding 包，调 create_rag_eval_dataset，RunConfig max_workers=1（串行）+ max_retries（扛 429），evaluate(dataset, metrics, embeddings, run_config)。跑起来可能几分钟，可能撞 HTTP 429。

**示例输出**：faithfulness 0.8333、answer_relevancy nan、context_precision 0.7500、context_recall 1.0000。分数 0-1，1.0 完美；高 faithfulness = 不编造；高 relevance = 回答具体问题；高 precision / recall = retriever 找到正确信息、没有噪声。answer_relevancy 可能返回 NaN（相似度算不出：空/太短答案、缺/无效 context、embedding 失败）。

**Advanced（前沿）**：
- **QPP（Query Performance Prediction，查询性能预测）**，来自《Am I on the Right Track?》：不靠人工标签，分析 query + retrieved result 估 query 质量；agent 第一个生成 query 的预测质量和最终答案质量正相关；更好的 retriever 能用更少迭代给更高质量答案；评估中间自生成 query 能发现走偏/漂移。
- **RAG-Critic**：第二个专科 critic model（批评模型）给 fine-grained structured feedback（细粒度结构化反馈），不只分数；分层错误 Tier 1 Retrieval error、Tier 2 Insufficient / incomplete in retrieval、Tier 3 Insufficient evidence / data coverage；能解释为什么（如 context 不足导致 hallucination）并触发 self-correction（自纠）。
- **Taxonomy（分类）**：Agentic RAG survey 分 single-agent router、multi-agent、hierarchical、corrective、adaptive；评估要匹配类型 —— corrective：量自纠是否改善答案；adaptive：量初始复杂度分类是否准确/高效；multi-tool single agent：量路由是否正确。

**自动化评估（CI/CD 式）**：Test-Refine-Measure-Repeat（测试-修正-度量-重复）—— Test 对 evaluation_dataset.jsonl 或 HotpotQA / TriviaQA；Refine 从 trace 分析（description、prompt、错误处理）；Measure 重跑 RAGAs；Repeat 和 baseline（基线）比（up = 成功，down = 回归）。

结论：需要超越 text-matching（文本匹配）的框架；golden dataset 是基础；RAGAs 定量量部件；QPP / Critic 是细粒度自动化评估的未来。

## 这门课怎么教的（Module 3）

Qualitative-to-quantitative（定性到定量）—— 失败分类 + trace 方法、三招 fix（description、错误处理、系统提示）、live Test-Observe-Refine-Repeat 演练、golden dataset + RAGAs、研究前沿（QPP、Critic、分类）、自动化 Test-Refine-Measure-Repeat。好懂处：agent psychologist / detective 框架、每招 fix 的 before / after 对、单个棘手查询（Yann LeCun / Transformers）贯穿 failure-diagnosis-fix-success（失败-诊断-修正-成功）、具体分数 + NaN 解释。最值得偷的：**在屏幕上从真实 trace 诊断，展示精确的 before / after 编辑，重跑同一查询让改善可见**。要避免：指标只讲抽象定义；课程把每个指标绑到“它会改变什么”，并警告 rate-limit / 时间成本。

## 自测 4 问（带答案要点）
1. Trace 分析三步？四种失败模式？
   → 读 Thought → 查 Action + Action Input → 看 Observation；选错工具、输入格式错、推理缺陷、工具执行错误。
2. 三招 fix 各修什么？
   → Description：降歧义，让选工具准；错误处理：有用错误不 crash；系统提示：出错/信息不足时不放弃。
3. RAGAs 四个指标各量什么？
   → Faithfulness：答案扎根 context；Answer relevance：回应问题；Context precision：相关/切题、噪声少；Context recall：必要信息都在。
4. 为什么 BLEU / ROUGE 不够？自动化评估循环是什么？
   → 多正确答案被扣分、动态数据、过程不量；Test-Refine-Measure-Repeat，和 baseline 比。
