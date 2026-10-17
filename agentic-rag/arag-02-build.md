# Build（Lessons 5–7）：设计 AI Research Assistant + 实现工具 + 组装

> 这一章是“做一个完整项目”。顺序：先写 blueprint（蓝图），再逐个实现工具（每个独立可测），最后组装进 main.py。读法：把三个工具（local RAG、arXiv、definition）当成三个“能力”，看 ReActAgent 怎么靠 description 选它们。

## 1. 设计：Blueprint 先于 Code（Lesson 5）

**Blueprint before coding（先写蓝图再写代码）**：多部件系统先讲清，再动手。好处：clarity（清晰）、needs up front（提前点名需求：API、数据源、挑战）、roadmap（路线图）。类比：house-blueprint（房子蓝图）。

**Brief（任务书）**：AI research assistant（研究助理），结合 curated local library（精选本地文库）和 up-to-date online academic / reference source（最新在线学术/参考源）。

三个需求：
- **Deep-dive analysis（深度分析）**：关于用户提供论文的详细问题。
- **Horizon scanning（前沿扫描）**：发现新的/相关的研究。
- **Concept clarification（概念澄清）**：快速定义。

映射到三个工具：
- **Tool 1 local RAG**：私有 PDF，VectorStoreIndex、OllamaEmbedding，data/。
- **Tool 2 arXiv search**：实时 arXiv.org，按 keyword / author，免费、无需 key（arXiv library）。
- **Tool 3 definition**：简洁定义，免费 Wikipedia library。

**Hub-and-spoke（轮毂-辐条）**：中央 ReActAgent（OpenAI LLM）解读 query，planner 选工具，observation 返回，agent 综合或再调工具。

**Draft V1.0 system prompt（系统提示草稿）**：expert（专家）、accurate / methodical（准确/有条理）；列三个工具和各自何时用；分析 intent、检查 description、没有合适的就用 general knowledge（通用知识）回答；think step-by-step（逐步思考）并解释 tool-choice reasoning（工具选择理由）。

什么时候写 blueprint：多部件系统动手前。别跳过蓝图直接写代码。失败模式：工具拼起来才发现需求不清。Fix：先写 brief + 工具映射。

## 2. 实现工具：Modular、独立可测（Lesson 6）

**Modular、independently testable tool（模块化、独立可测的工具）**，先于组装。结构：/ai-research-assistant，下有 /data（paper1.pdf、paper2.pdf）、/tools（__init__.py、rag_tool.py、api_tools.py）、main.py、.env、requirements.txt。

依赖：llama-index、llama-index-llms-openai、llama-index-embeddings-ollama、python-dotenv、arxiv、wikipedia；.env 存 OPENAI_API_KEY；本地 Ollama + nomic-embed-text。

**Tool 1 local RAG（tools/rag_tool.py）**：get_rag_tool() factory（工厂）；Settings GPT-5 + Ollama；加载 ../data；VectorStoreIndex.from_documents；as_query_engine similarity_top_k=3；QueryEngineTool.from_defaults，名字 local_rag_tool；**description 最关键**；__main__ 块做 isolated testing（隔离测试，经 tool.call）。

**Tool 2 arXiv（tools/api_tools.py）**：FunctionTool 包装 arxiv_search_function(query, max_results=3)；arxiv.Search 按 relevance（相关性）排序；格式化成 Observation：title / authors / URL / 前 250 字符 summary；名字 arxiv_search_tool；隔离测试经 arxiv_tool.call。

**Tool 3 definition**：FunctionTool 包装 define_term_function(term)；wikipedia.summary(term, sentences=2, auto_suggest=False)；**PageError** 给 helpful not-found（有用的“找不到”）；**DisambiguationError** 给 actionable（可操作的），带最多 3 个选项（e.options[:3]），agent 可以澄清；generic catch-all（通用兜底）永不让 agent crash；名字 definition_tool；用“Transformer architecture”测试。

Tip：Wikipedia / 公开 API 可能 rate-limit（HTTP 429）；cache（缓存）definition 查询。

什么时候模块化：工具多于一个时。别把三个工具写进一个文件。失败模式：一个工具 crash 拖垮整个 agent。Fix：每个工具独立测试 + 兜底错误处理。

## 3. 组装 + 编排：main.py（Lesson 7）

**main.py** 是 orchestration / entry（编排/入口）：初始化 LLM + 工具、组装 ReActAgent、处理 query。

Setup：load_dotenv；OpenAI gpt-5；rag_tool=get_rag_tool()；all_tools=[rag_tool, arxiv_tool, definition_tool]（模块化编辑列表）；ReActAgent.from_tools verbose=True（打印 Thought-Act-Observe 日志）。Interactive loop（交互循环）：async main，while prompt 输入 != q：response=await agent.run(prompt)。

**读 verbose trace（详细追踪）**：OpenAI HTTP POST = thinking / synthesizing；Running step call_tool + tool log（Ollama embed POST）= Act；ToolCallResult = 下一循环的 Observation；字面 Thought / Action / Observation 标签可能不出现。

**Sanity check（合理性检查）**：问《Attention Is All You Need》结论 → 应走 local_rag_tool；问 recent Mixture of Experts paper → 应走 arxiv_search_tool；问“What is a hyperparameter?”→ 应走 definition_tool。

HTTP 429 在免费/共享层正常；库自动 retry；等一分钟；生产需要付费层、更好处理、queue（队列）。

**Tool selection（工具选择）**：LLM 拿到 query + history + description；semantic matching（语义匹配）+ structured output（结构化输出：可解析的 Thought / Action / Action Input，框架执行 named function）。失败：**wrong tool（选错工具，accuracy，最常见）**，来自 ambiguous description / query；**calling all tools（全调，低效）**—— 目标是 intelligent routing（智能路由），不是 brute force（蛮力）。

**Scalability（可扩展性）**：高质量 descriptive（描述性，不是 instructional 指令式）、distinct / specific（区分/具体）的 description；system prompt 行为规则（think step-by-step 为什么工具相关、怎么恢复 —— critic-guided，批评引导）；single-responsibility（单一职责）专科工具，避免 monolithic super-tool（单体超级工具）；几十/几百个工具时用 hierarchical routing / planning manager（层级路由/规划管理器）委派给 worker；expert move（专家动作）：在高质量 reasoning trace（推理轨迹）上 fine-tune LLM。

什么时候用 hierarchical routing：工具数到几十/几百时。别给三个工具就上层级路由。失败模式：description 含糊 → 选错工具。Fix：descriptive、distinct、specific 的 description + 行为规则。

## 这门课怎么教的（Module 2）

Design-before-code（先设计再写代码）、side-by-side（并排）、modular build（模块化构建）、assembly（组装）—— 蓝图、静态 vs agentic、工具隔离、main.py 编排。好懂处：promotion metaphor（提升隐喻：pipeline 变成 tool）、反复强调 tool description 驱动 reasoning、line-by-line walkthrough（逐行讲解）绑到单个项目、sanity-check 查询预测哪个工具该触发。最值得偷的：**一个项目横跨多集，单个变更做 before / after，把 trace 当故事讲（thought、action、observation）**。要避免：长段不间断代码讲解，不按函数切块、不说看什么；课程按 labeled line range（标注行范围）切块，每个工具组装前隔离测试。

## 自测 4 问（带答案要点）
1. Blueprint 里三个需求映射到哪三个工具？
   → 深度分析 → local RAG；前沿扫描 → arXiv；概念澄清 → definition。
2. 为什么每个工具要独立可测？Definition 工具的错误处理有什么讲究？
   → 一个工具 crash 不拖垮 agent；PageError 给有用 not-found，DisambiguationError 给最多 3 个可操作选项，通用兜底永不 crash。
3. 读 verbose trace 时，什么对应 think / act / observation？
   → OpenAI POST = think / synthesize；call_tool + tool log = act；ToolCallResult = observation。
4. 工具选择的两个失败模式？扩展到几十个工具时怎么办？
   → 选错工具（最常见，description 含糊）+ 全调（低效）；层级路由/规划管理器委派给 worker。
