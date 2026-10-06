# Ch19: Distributed Search — 提炼笔记（87–92，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 87. 三组件分解
- Crawler（发现+抓文档）/ Indexer（建索引）/ Searcher（答查询）——三者扩展和可用性需求不同，**分开设计、分开扩展**。
- 口径：答"设计搜索"先报规模压力，再点名三子系统。

## 88. 需求与估算
- 功能：crawl、index、按相关性 rank、快速返回。非功能：可用、可扩展、低延迟、成本。
- Building blocks：blob store（存原始文档 + 持久化索引，跨 region 复制保 durability）+ 索引/查询层。
- 估算锚点（YouTube 式）：150M DAU 启发式当 150M RPS（**非常激进，面试时要声明这是 heuristic 并区分峰均值**）；单机 64K RPS → 约 **2350 台**；每视频 300KB（200KB 文档 + 100KB 索引，按 1000 词/文档×100B/词条算），日增 6000 → **1.8GB/天**；1736 RPS 均值 → 入 1.39 Mb/s、出 55.56 Mb/s（text-only 结果假设，先声明）。

## 89. 倒排索引
- **Inverted index**：term → 含它的文档（带频次+位置）；**forward index** 是镜像（doc → 它的词）。
- 建索引流水线：抽文本 → 分词 → 计频次+记位置 → merge 入索引。
- docID key 三选：video ID（只查视频够用）/ URL（还要索引原始链接）/ UUID（最灵活，最贵）。
- 决策：**实时索引**（crawl 到可搜延迟低，但复杂）vs **离线索引**（按 batch 大小和间隔调延迟）；索引构建**不放查询关键路径**；索引可从持久文档库重建 → index 节点不需要文档库级别的 durability。

## 90. 设计：两种分片
- 组件：client、**search coordinator**（fan-out 到 search nodes 再聚合）、search nodes（持索引分区）、index nodes（离线建索引）、blob storage、ingestion pipeline、metadata store。两条链路分开画：**ingestion 流 vs query 流**。
- **Term-sharded vs document-sharded**（必考对比）：
  - 按 term 分：单次查询 fan-out 小，但热词 → 热点分片。
  - 按 doc 分：数据均匀，但每次查询打**所有**分片，网络+聚合成本高。
- 索引发布：**all-at-once**（一致切换，需同步机制）vs **rolling**（安全，过渡期版本混杂）；用 **versioned index snapshot + metadata + 缓存**协调上线。
- 数据分布：consistent hashing，replication factor **3**。

## 91. 扩展
- 三板斧：partitioning、replication、跨 AZ 冗余。Sharded partitioning = 分区 + 复制，主从 + **quorum 写（W+R>N）**。
- **同步 vs 异步复制**：同步强一致但写延迟高、副本挂则阻塞；异步快但 crash 可能丢。
- 分区越多并行度越高，但单查询 merge 成本和尾延迟也越高。**索引和搜索独立扩展**；热结果缓存；用 commodity + spot 实例（挂了只重建受影响文档）。
- 数字：search server 30 RPS，3 万 RPS → 1000 台；RF=3。

## 92. 评估
- 答题收尾模式：开场列的每个非功能需求，映射到一个具体机制（AZ、离线索引、分区、廉价硬件）。
- 本章接受的 trade-off：**用轻微索引过期，换永远可用+低延迟搜索**。

## 自测 4 问（带答案要点）
1. Inverted index vs forward index？
   → inverted：term→文档（含频次/位置），按词找文档；forward：doc→它的词，概念镜像。
2. Term-sharded vs document-sharded 的 trade-off？
   → term 分片单次查询 fan-out 小，但热词成热点分片；doc 分片数据均匀，但每次查询打所有分片，聚合成本高。
3. Quorum W+R>N 保证什么？举例？
   → 读写副本数之和大于总副本数，保证读到最新写；如 N=3 取 W=2、R=2。
4. 索引发布的两种策略？
   → all-at-once（一致切换，需同步机制）vs rolling（安全，过渡期版本混杂）；用 versioned snapshot + metadata + 缓存协调。
