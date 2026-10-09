# Discover + 数据工程（Ch3）：RAG 的生产地基

> RAG 是 LLMOps 里最便宜的可靠性胜利：答案有据可查，不用重训。但生产 RAG 是个数据 pipeline，LLM 只是挂在后面的附件。

## 1. RAG 架构

**RAG（Retrieval-Augmented Generation）**：查询时检索最相关的文档，塞进 prompt，让模型照着回答。解决：私有/时新知识、hallucination、还能引用来源。

管线：ingestion（连接器、解析、**chunking** 切块、embedding）→ 向量库 → 检索（query embedding、相似搜索、re-rank、取 top-k）→ prompt 组装 → LLM 生成 → 校验/后处理 → API/UX，全程包 observability。

关键 trade-off：
- **Chunk size**：太大稀释相关性，太小丢上下文；
- 取几块（top-k）：多塞 context 烧 token，还可能干扰模型；
- 选哪个 LLM。
- 保持检索层**以文本为主**——可调试性（debug 时能看懂）。

## 2. Scoping：先谈钱和约束，再谈架构

从用户和"答对了能干嘛"出发，翻译成三约束：
- **Latency budget**：无流式时 >~2s 用户就觉得坏了；
- **Cost budget**：单 query 多少钱；
- **Compliance**：数据驻留、PII 处理。

这三条几乎决定后面所有技术选型。**Build vs buy**（调 API vs 自托管）看量、数据敏感度、延迟。MVP = 主动砍掉"挂了赔不起、又测不明白"的用例。
最贵的错误：**"好"的定义没对齐，就开始搭管线**。先写下成功指标、三约束、数据清单（来源/格式/新鲜度/权限）。

## 3. 数据地基

生产 RAG = 数据 pipeline + LLM 附件：
- **Ingestion**：各源连接器、PDF/扫描件 OCR 解析、归一化；
- **Chunking**：固定窗口（如 ~500 tokens、~50 重叠不断义）、语义切分（按话题断）、递归切分（段落→句子）；
- **Metadata**：每块贴来源/页码/日期/权限——过滤（"只搜 2024 年文档"）、引用、访问控制全靠它；
- **数据质量**：去重、embedding 前 PII 脱敏、新鲜度——**检索质量更多取决于数据干净，而不是 embedding 模型多强**。

决策：全量重建索引 vs 增量更新（新鲜度 vs 成本复杂度）；数据集版本化（eval 可复现）。
起点：200–1000 token 块、10–20% 重叠，再按检索指标调。
**把语料当生产数据库管**：契约、质检、更新 SLA。脏数据/旧数据让答案静默变差，不报错。

## 4. Embedding、向量存储、检索评估

- **Embedding**：文本 → dense vector（一长串数字），语义变成距离计算（常用 cosine similarity）。
- **Vector database**：存向量、做近似最近邻搜索；**HNSW** 是常用索引引擎（图索引，拿一点召回换大量速度）。
- **Hybrid search**：dense 向量 + sparse 关键词（BM25）+ re-rank，常比单打独斗强——向量抓改写（"time off policy"），关键词抓精确词（"PTO-2024"）。
- 检索指标：**context precision**（检回的里几个相关）、**context recall**（相关的里检回几个）、**MRR**（第一个正确答案排第几的倒数均值）、**nDCG**（排序质量分）。
- 生成侧 headline 指标：**faithfulness**（答案是不是真被检索到的上下文支持的）。

铁律：**检索和生成各用各的 golden 数据集分开评**。很多时候换个 embedding/切块策略，比换大模型更划算。

## 自测 4 问（带答案要点）
1. RAG 管线六步？挂了先查哪？
   → ingestion→向量库→检索→组装→生成→校验；先查检索（喂给模型的东西）。
2. Scoping 的三约束是什么？为什么先谈它们？
   → latency/cost/compliance；决定后面几乎所有选型。
3. Chunking 的起点配置？metadata 干嘛的？
   → 200–1000 tokens、10–20% 重叠；过滤/引用/访问控制。
4. 检索评估四个指标？生成侧看什么？
   → precision/recall/MRR/nDCG；faithfulness。
