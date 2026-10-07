# RAG 系统设计（Ch11）：最高频的 GenAI 实战题

> RAG 是"LLM 知识是静态的"这个痛点的标准解。面试考的是整条管线，不是"调个向量库"。

## 1. 四个组件 + 需求

**数据索引 → 检索器 → 增强 → 生成器**。功能需求：理解意图、摄取/索引、 relevant 检索、基于检索生成；非功能：低延迟、可扩展、高可用、**数据新鲜度（不用全量重建索引）**、内容安全、容错。

## 2. Chunking：杠杆最高的决定

直觉：LLM 一次能看的上下文有限，文档必须切片。**切太碎**：检索准但丢全局上下文、调用次数多；**切太大**：上下文全但塞进噪音、还贵。策略：
- **Fixed-length**：简单粗暴，按固定长度切，可能把一句话拦腰斩断。
- **Semantic**：按段落/语义边界切，保留结构。
- **Overlapping**：相邻块重叠一点，边界信息不丢。
- 面试口径：chunking 策略是 RAG 里**杠杆最高**的决定，先讲它。

## 3. Embeddings：一致性是铁律

- **索引时和查询时必须用同一个 embedding 模型**（甚至 BERT vs RoBERTa 混用都会掉检索质量）。
- **Dense vs sparse**：dense（embedding）懂语义但贵、不可解释；sparse（BM25/TF-IDF）精确匹配、便宜、可解释。**Hybrid 两者结合**是生产默认。
- 向量库：Pinecone（托管）、Milvus（开源）、Weaviate、Faiss。
- 检索：cosine / 欧氏 / 点积；hybrid = dense + sparse。
- **对抗威胁**：keyword stuffing、embedding collision、contextual poisoning → 异常检测防御。

## 4. 系统设计（100M DAU 量级）

- 估算：3B 模型 6GB；检索语料 10TB；**交互数据是重头**：100KB/查询 → 100 TB/天 → 3 PB/月；112 GPU 服务器；ingress 185 Mbps / egress 926 Mbps。
- 三个子系统：
  1. **Query 理解与 embedding**：分词/纠错/同义词扩展 → SBERT/MiniLM 编码 → embedding 缓存。
  2. **知识管理**：摄取（PDF/DOCX/HTML → 语义切块）→ embedding → Faiss/Pinecone 索引（hybrid 检索）→ metadata 预过滤（日期/作者/来源，**拿召回换延迟**）。
  3. **生成与质检**：组装 context（stateless，Redis 暂存）→ LLM 生成（temperature/流式）→ **质检（幻觉检查、政策检查、用户反馈），生产环境质检不是可选项**。
- **降级模式**：子系统解耦，挂了可以跳过 metadata 过滤等环节继续服务。

## 自测 4 问（带答案要点）
1. Chunking 为什么是 RAG 杠杆最高的决定？三种策略的 trade-off？
   → 决定 LLM 看到什么；fixed 简单但断语义、semantic 保结构、overlapping 保边界。
2. Embedding 一致性为什么是铁律？
   → 索引和查询模型不一致，检索质量直接掉（连 BERT/RoBERTa 混用都不行）。
3. 100M DAU 下 RAG 的存储重头是什么？
   → 交互数据 100 TB/天（3 PB/月），不是模型也不是语料。
4. 生产 RAG 挂了怎么办？
   → 子系统解耦+降级模式（如跳过 metadata 过滤）；质检是必选项。
