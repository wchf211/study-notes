# RAG + Agents + Fine-tuning（Modules 4–6）

> 这一章的决策主线：先问“失败在哪个阶段”，再决定用哪个手段。RAG、agent、fine-tune 不是三级台阶，而是三把不同的扳手。

## 1. RAG：Embeddings → 向量库 → Chunking → Pipeline → 排障

**Embeddings（向量嵌入）**：把文本映射成向量，语义相近的文本在向量空间里离得近。**Cosine similarity（余弦相似度）**：衡量两个向量方向多接近，常用来排候选。**Vector database（向量数据库）**：给向量建索引，做快速近似最近邻搜索 + metadata 过滤。

**Chunking（切块）**：把文档切成自包含的小块。**Hybrid search（混合检索）**：keyword（关键词）+ vector（向量）一起搜，再 rerank（重排）。调 chunk 大小 / overlap（重叠）要按文档类型来。失败模式：答案被切到两个 chunk 的边界上，两边都不全。Fix：chunk 自包含 + hybrid + rerank 当默认，不要纯向量。

**End-to-end RAG pipeline**：ingest（摄入）→ chunk → embed → index → retrieve → grounded generation（基于证据生成）→ cite / 说“没找到”。关键设计：retrieval 和 generation 分开，各自能独立测试。**No-evidence fallback（无证据回退）**：检索不到时，明确说“没找到”，而不是编一个答案 —— 这是设计出来的状态，不是失败。

**分阶段排障**：缺文档？chunk 坏了？检索错了？context 被忽略？还是 hallucinated filler（幻觉填充）？先定位阶段，再修该阶段。失败模式：检索是 bug，却去 retrain / fine-tune。Fix：修阶段 + 加 eval + fallback。

向量库选型看：scale（规模）、latency、freshness（新鲜度）、ops burden（运维负担）。失败模式：源文档变了，index 还是旧的。Fix：index freshness 当成运维指标。

## 2. Agents：Tool → 框架 → 循环/记忆 → 组装

**Function calling / tool use**：模型发出结构化调用，代码 validate（校验）、authorize（授权）、execute（执行），再把结果返回。工具要清晰 schema、timeout、least privilege（最小权限）。失败模式：模型调 destructive tool（破坏性工具）没人拦。Fix：校验 + 授权 + 最小权限，工具调用不是“模型说了算”。

**Agent frameworks（框架）**：框架管 loop、tool routing、memory、error handling。Hand-rolled loop（手写循环）更易 debug，框架更易扩展。选型：control vs speed。失败模式：单工具路径还没跑通，就先上了框架。Fix：先让最简单的 loop 工作，再决定要不要框架。

**Reasoning / planning / memory**：ReAct 式循环 reason → act → observe；plan-then-execute（先计划后执行）；short-term working memory（短期工作记忆）+ summarized / persistent memory（摘要 / 持久记忆）。Cap steps and cost（步数和花费封顶）。失败模式：unbounded loop（无限循环）、stale memory（过期记忆）。Fix：硬上限 + 记忆刷新策略。

**组装一个 working agent**：以 research-style agent 为例（search + summarize + answer），配工具、检查、error handling。重点不是新理论，而是端到端拼起来。失败模式：跳过 tool 结果的 verification（校验）。Fix：工具结果也要验，不能裸信。

## 3. Fine-tuning：先决策，再动手

**决策树（prompt vs RAG vs tune）**：prompting = 最便宜最快，调行为；RAG = 新鲜 / 外部事实；fine-tuning = 稳定的风格 / 格式 / 领域行为，且数据稳定。先试 prompting / RAG。

失败模式：fine-tune 去教“会变的事实”。Fix：会变的知识走 RAG，稳定的行为才考虑微调。

**PEFT（参数高效微调）**：LoRA / QLoRA 在 frozen（冻结）的 base model 上训小 adapter（适配器），比 full fine-tuning 省内存 / 算力。适合风格 / 格式 / 领域适配、中等硬件。失败模式：数据太少或 rank 不对，导致 underfit / overfit（欠拟合 / 过拟合）。Fix：数据量 + rank 要匹配任务。

**Dataset（数据集）**：干净的 prompt→response 对，边界清晰、覆盖均衡、held-out test split（留出测试集）、无 PII（个人身份信息）/ 泄露的测试数据。Quality beats volume（质量胜过数量）。失败模式：噪声 / 重复例子教坏行为。Fix：清洗 + 去重 + 覆盖检查。

**评估微调**：base vs tuned 在 held-out 任务上比 quality、regression（回归）、format compliance（格式合规）、safety（安全），不只看 training loss（训练损失）。失败模式：只看训练指标就上线。Fix：回归 + 安全门。

## 这门课怎么教的（Modules 4–6）

Module 4 用“必须引用来源的客服 / 研究助手” hook，顺序是 vectors → store → chunk/search → pipeline → failures，最值得偷的是按 pipeline 阶段做 failure triage。小坑：hybrid search 在完整 pipeline 之前就讲了。Module 5 用“必须用工具的研究助手” hook，顺序是 tool call → framework → loop/memory → build，最值得偷的是带 step cap 的 loop 图。小坑：framework 课在 loop 完全讲透之前就出现了。Module 6 用“prompting / RAG 都不够用了” hook，顺序是决策 → 高效方法 → 数据 → 评估，最值得偷的是决策树先行、方法后行。

## 自测 4 问（带答案要点）
1. RAG pipeline 六步？为什么 retrieval 和 generation 要分开？
   → ingest → chunk → embed → index → retrieve → grounded generation → cite / 没找到；分开才能各自测试。
2. RAG 排障怎么分阶段？最常见的误修是什么？
   → 缺文档 / 坏 chunk / 检索错 / context 被忽略 / 幻觉填充；检索的 bug 别去 fine-tune。
3. Prompt vs RAG vs fine-tune 的决策规则？
   → prompting 调行为最便宜；RAG 给新鲜 / 外部事实；fine-tune 给稳定风格 / 格式 / 领域行为；先试前两个。
4. Tool 调用里模型和代码的分工？为什么 single-tool 路径先跑通再上框架？
   → 模型发结构化调用，代码校验 / 授权 / 执行；框架掩盖 bug，先让最简单 loop 工作。
