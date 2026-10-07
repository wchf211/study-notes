# AI 时代 I：重塑与 ChatGPT 系统 — 提炼笔记（180–184，2026-10-06）


## 180. AI 如何重塑系统设计
- AI 从产品功能变成 **computing substrate**；现代系统要会"推理、记忆、检索"（LLM 推理、对话记忆、知识库检索）。
- 新增 boxes：GPU/TPU 推理集群（与 app tier 分开）、**vector DB**、**RAG 管线**（ingest→chunk→embed→index→retrieve→rerank）、prompt/上下文编排、流式响应基建（SSE/WebSocket）、tool calling、eval 管线（golden dataset）、模型管理（版本/canary/回滚）、AI 安全栈（guardrail、moderation、PII 脱敏、jailbreak 检测）、反馈闭环。
- 被打破的四个经典假设：
  - **无状态** → 要会话/聊天历史存储；
  - **精确缓存** → **语义缓存**（同义不同词命中同一条；调 embedding + 相似度阈值 vs 误用/过期复用）；
  - **延迟预算** → **time-to-first-token** 成核心指标（buffering、backpressure、排队）；
  - **成本** → 按 token 计费，每个设计决策都是成本决策。
- 成本抓手：prompt caching、模型路由/cascade、语义缓存、distillation、小微调模型、spot GPU、按队列 autoscaling。
- 口径：新题型（chatbot、AI code review、RAG QA、embedding 推荐、tool-calling agent）；评分看 **quality/latency/cost/safety 四维 trade-off**；永远谈 eval 和 failure mode。

## 181. 角色与工具
- Role → 面试焦点：ML system designer（训练平台）、**AI platform engineer**（多租户 LLM 网关：路由/限流/成本/failover）、agents engineer（agent loop：planner/memory/tool/sandbox）、GenAI full-stack（chat UI/会话/流式/重试）、applied AI（RAG bot：检索质量/prompt/guardrail/eval）、infra（GPU 集群 gang 调度/bin packing/spot；推理 batching/量化/KV-cache）、data（管线/embedding/向量入库/eval 数据集）、safety、SRE（TTFT SLO）、security（prompt injection）。
- 警告：**别只报工具名**，要说清它在管线里的位置（retriever→reranker→orchestrator→model→guardrail→cache→eval），否则不如不说。
- "prompt engineering 能替代系统设计吗？"→ 不能：prompt 只是一个组件，面试考的是整个系统。

## 182. AI 设计题三桶
先定桶，再分配深度：
1. **AI 增强的传统系统**：LLM 不是中心（如语义搜索、embedding 推荐）。深挖：embedding、vector DB、ANN、keyword+vector 混合检索、rerank、索引新鲜度、成本/延迟。
2. **LLM 为中心**：chatbot/RAG QA/客服 agent。深挖：prompt 设计、context window、RAG、对话状态、流式、guardrail、eval、按 token 成本、fallback。
3. **ML 平台/基建**：训练/ serving/eval 平台。深挖：数据管线、GPU 调度、分布式训练、registry、版本、canary/回滚、batch vs streaming 推理、drift 监控。
- 各桶瓶颈：检索质量 vs 延迟 / 质量-成本-安全三角 / 吞吐利用率 vs 可靠性。

## 183. ChatGPT：需求与估算
- 功能：多轮对话管理（上下文感知）、NLU（意图+实体）、个性化（偏好/历史）、反馈收集（评分→模型改进）。
- 估算（10 亿用户、1.5 亿 DAU、日 10 请求 → **15 亿请求/天**；请求 2KB、响应 5KB）：
  - 服务器：150M/64K ≈ **2343 台**——**只算 web/app tier，GPU 推理 tier 另算且贵得多**（面试官必问）。
  - 存储：1.5B × 2KB ≈ **3TB/天**。
  - 带宽：约 17361 RPS → 入 **277.8Mbps**、出 **694.4Mbps**（响应比请求长，egress 大）。
  - 模型：30 亿参数 FP16 ≈ **6GB**，必须驻 GPU 显存（常跨卡分片）做实时推理。

## 184. ChatGPT：详细设计
- 链路：prompt → **API 网关**（鉴权/限流/会话）→ LB → app → **NLU 预处理**（意图/实体/转 embedding）→ profile service（历史/偏好）→ **model servers**（+ vector DB 取语义上下文）→ **生成后 moderation** → 返回；反馈进离线重训。
- 关键决策：
  - **Moderation 放生成之后**：要审模型实际产出的东西，不只审 prompt。
  - **语义缓存 > LRU**：同义不同词命中同一条。
  - Token 计费：pub/sub 事件打点进支付系统。
  - **结构化反馈**（赞踩）放 RDBMS，**非结构化反馈**（评论/会话元数据）放 MongoDB——按访问模式选库。
  - ZooKeeper 协调 model servers（成员/配置/选主/failover）；Redis 缓存对话上下文；CDN 只放静态 UI，**生成内容 bypass CDN**。

## 自测 4 问（带答案要点）
1. AI 打破了哪四个经典假设？
   → 无状态（要会话存储）、精确缓存（→语义缓存）、延迟预算（→TTFT 为王）、成本（按 token 计费处处是成本决策）。
2. AI 设计题三桶怎么分？
   → AI 增强传统系统 / LLM 为中心 / ML 平台基建；先定桶再分配深度，别在错的层上过度设计。
3. Moderation 为什么放生成之后？
   → 要审模型实际产出的东西，不只审 prompt。
4. 15 亿请求/天、2KB 请求、5KB 响应，带宽多少？注意什么？
   → 约 17361 RPS → 入 277.8Mbps、出 694.4Mbps（响应比请求长）；app tier 和 GPU 推理 tier 分开算。
