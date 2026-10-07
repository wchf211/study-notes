# AI 时代 II：数据基建、LLM 客服 bot、AI 代码助手 — 提炼笔记（186–192，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 186. AI/ML 数据基建：需求
- 六大挑战：**training-serving skew**（训练和线上用的特征不一致，预测退化）、特征难复用、缺可复现性（版本化）、PB 管线+GPU 训练扩展瓶颈、**data drift**（生产数据漂离训练分布）、特征计算延迟拖累实时推理。
- 范围：批量+流式采集 → 清洗/归一/特征工程 → raw 与 processed 双版本不可变存储 → 离线训练+在线推理双路 serving → 质量/血缘/模型-数据交互监控。
- 估算：1 万事件/s × 1.5KB = **1.5TB/天** raw；变换 2.5x + 特征 0.5x → **6TB/天** processed → **2.2PB/年**；节点：ingestion 6、processing 15、serving 30；内部 storage↔processing 流量 multi-Gbps；按峰值（非均值）配容量。
- 口径：开场先列六挑战，尤其 skew；强调 **"version everything"**。

## 187. 五层架构（白板可画）
- Ingestion → Raw Storage → Processing → Processed Storage → Serving。
- Ingestion：流式 Kafka（解耦生产/消费，要 dedup/buffering）+ 批量 API connector 与 **CDC**（抓数据库变更）。
- Raw：blob lake（S3），**immutable + schema-on-read**——管线失败了直接重跑。
- Processing：Airflow ETL；**quality service 把脏批次拦进 quarantine zone**（不进生产）；**lineage** 记录每个特征的源事件+管线 run（debug 和 GDPR）。
- Processed：Parquet lake（重训）+ warehouse（BigQuery 分析）+ metadata catalog（发现）。
- **Feature store dual publishers**：同一份变换写 offline（训练）+ online（serving）——**结构性解决 training-serving skew**；在线读 Redis，ms 级。

## 188. 细节：exactly-once 与去重
- Kafka 按 **entity_id 分区**：保证单实体事件有序 + 消费者并行；consumer offset 保容错。
- CDC 去重：message ID 去重 + sink 幂等（按 entity_id + timestamp **UPSERT**）；批量 connector 带鉴权/分页/退避+jitter/检查点/先验后写。
- Processing DAG：extract→validate→transform→publish + quality gates；**exactly-once = checkpointing**；去重键 (event_id, source_timestamp)；迟到数据用 **watermark + allowed-lateness**；质量门拦坏批次进 quarantine。
- 发布：offline Parquet（按 feature/version/date 分区）、Redis online upsert、catalog 更新、drift 指标上报。

## 189. LLM 客服 bot（RAG）：需求与估算
- RAG 链：KB 预处理（clean→chunk→embed→index 进 vector DB）→ 每查询检索相关文档 → 增强 prompt → LLM 生成。"garbage in, garbage out" 适用于喂模型的 KB。
- 功能：自由意图理解、**grounded 回答**（引用 KB 而非模型参数记忆）、**低置信转人工**、附件/截图、多轮、反馈收集。
- 非功能：<5s 响应、99.9%、数亿用户、**LLM 之前先 PII masking/redaction**、监控面板。
- 估算：1 亿用户 × 日 2 条 × 100 token = **200 亿 token/天** → 约 **1 万张常驻 GPU**；存储 \~3TB/天 → 10 年 30–35PB。
- 决策：managed LLM API vs 自建（成本/控制/延迟）；检索质量 vs 索引新鲜度；会话状态缓存降延迟。

## 190. RAG bot：两条 workflow
- KB ingestion：事件驱动异步——KB 变更 → 清洗/格式化 → chunk → embed → 进 vector DB（批量 embed 便宜但 stale，per-event 新鲜但贵）。
- 请求：intent（分类器/NER/情感）→ 取会话上下文 → **hybrid retrieval**（vector + keyword/fuzzy，语义召回+确定性）→ 组 enriched prompt → LLM → adequacy check → 回答或**转人工**。
- **反馈闭环是学习引擎**：人工接管/修正成为训练信号。Admin 面：analytics、人工 handoff、feedback loop。

## 191. AI 代码助手：延迟游戏
- 定义性约束：**TTFT <300ms**——开发者等提示词会断 flow。
- 延迟抓手：**SSE 流式**（首 token 快到）、缓存重复 prompt、客户端 debounce **150–200ms**、vLLM batch 推理；**telemetry 全异步**（pub-sub，永不 block IDE）。
- 估算：100 万开发者 × 日 50 次补全 = 5000 万次/天；峰值 \~6000 RPS；GPU 7500/50 = **150 台**（+25% buffer ≈ 190）；telemetry 4TB/天；带宽 1.8GB/s。

## 192. 代码助手：架构细节
- API：getCompletion（POST、SSE 流式；带 session_id/file_content/cursor/language/open_files）/ submitFeedback（**fire-and-forget**，IDE 不阻塞）/ explainCode（SSE；instruction-tuned prompt）/ getStatus（健康检查，客户端降级用）。
- 推理链：context 聚合 → vector DB 取 repo 相关代码片段（RAG 式）→ 缓存查（prompt-hash Redis，10–20% 命中；**5% 命中率提升省数千 GPU-小时/月**；TTL 平衡 stale）→ vLLM batch + KV-cache → SSE 流式 → 写缓存。
- Telemetry：IDE 事件 → Kafka → columnar store（ClickHouse/BigQuery，4TB/天）→ 聚合（acceptance rate、延迟分布、按语言）→ 微调/eval 管线。
- **优雅降级**：推理失败或超时 → 不返回 suggestion（不报错），cached 响应部分兜底——IDE 必须保持响应。
- 成本：INT8 量化、prompt 缓存、按区域流量 autoscale；model registry 在 blob（版本/量化/路径/部署状态，安全回滚）。
- 收尾口径：**requirements-compliance 表**（每个功能/非功能需求 → 组件 → 如何满足）。监控：acceptance rate、suggestion 与最终代码的 edit distance、TTFT、按语言性能；警惕 review/QA 瓶颈（AI 产出快过人工 review）。

## 自测 4 问（带答案要点）
1. training-serving skew 怎么解？
   → dual publishers：同一份变换逻辑写 offline + online feature store，线上线下共享特征定义。
2. RAG bot 为什么 retrieval 质量比模型大小重要？
   → 垃圾 KB → 幻觉；hybrid retrieval（vector + keyword）；PII redaction 必须在进 LLM 之前。
3. 代码助手的延迟游戏怎么打？
   → TTFT<300ms：SSE 流式首 token、debounce 150–200ms、prompt 缓存、vLLM batch；telemetry 全异步不 block IDE。
4. 流式管线怎么保证 exactly-once？
   → 幂等 sink（UPSERT entity_id + timestamp）+ checkpointing；CDC 按 message ID 去重；迟到数据 watermark + allowed-lateness。
