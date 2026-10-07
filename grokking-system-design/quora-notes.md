# Quora — 提炼笔记（116–120，2026-10-06）


## 116. 开场
- Quora 优化的是**深度**（搜索引擎优化速度），代价是延迟（等人来答）。
- 好题目的原因：读写都重 + 社交图谱 + 排序 feed + 搜索。四步法：需求 → 初版 → 终版（含垂直分片）→ 评估（含可用性加固）。

## 117. 估算（1B 用户 / 3 亿 DAU）
- 功能：问答（文本/图/视频）、赞踩、评论、关键词搜、个性化 feed、答案按有用度排序。
- 假设：每 DAU 每天 1 问；15% 带图（250KB）、5% 带视频（5MB）、文本 100KB/问。
- 存储：30（文本）+ 11.25（图）+ 75（视频）= **116.25TB/天 → 42.43PB/年**。
- 服务器：峰值 3 亿 RPS ÷ 64K/台 ≈ **4700 台**。带宽：入 11 / 出 215.3 / 共 **226.3 Gbps**。
- 一致性分档：发帖/评论可最终一致（用户容忍延迟的地方就放松）。

## 118. 初版：读写分离 + polyglot
- API：createQuestion、addAnswer、addComment、upvote、addTopic、followTopic、getQuestion(s)、searchQuestionsByKeyword、getUserFeed。
- 数据模型：User/Question/Answer/Comment/Topic；Topic↔Question 多对多（junction 表）、User↔Topic 多对多（follow）、Upvote **多态**（可投问题或答案）。
- 架构：LB 后 web 服务 + session store；按领域拆**专用服务**（question/topic/answer/comment/vote/user/search/feed/metadata）+ 推荐引擎 + ML ranker（topic 图谱 + 用户历史）。
- 关键链路：feed 用**事件驱动 pub/sub** 构建、**WebSocket** 推给 follower；搜索索引靠事件流实时刷新；媒体走 blob + CDN。
- 口径：**尽早把读（feed/搜索/排序）和写（发问/答）分开**；polyglot 理由——关系型存实体关系，blob 存媒体，缓存存热点。

## 119. 终版：按 workload 选引擎
- **InnoDB**：元数据+关系，ACID，主从复制 + MySQL Router。
- **MyRocks**（LSM-tree）：写密集型 workload。
- **HBase**（宽列）：feed 数据，稀疏宽表关系型不好放。
- 内容放分布式文件系统；媒体 blob + CDN。
- 缓存：Memcached，LRU + **TTL 失效** + **multiget 批量**省 round trip。
- 分区：**先垂直分片**（按服务拆表，简单）→ 增长后再 hash-based 水平分片。
- Feed 路径：**Kafka** 收事件 → ranker 预计算排序 → 推入 Cassandra/MyRocks 的 feed 存储；Kafka 把 ingestion 和排序解耦， spike 不打断 feed 新鲜度。
- ZooKeeper 管配置和服务发现。

## 120. 评估：一致性分档 + DR
- 一致性：**关键数据（问题、投票）同步复制强一致**；非关键（浏览计数）最终一致。
- 可用：无状态同构服务 + 自动扩缩 + 分片；datastore 隔离、shard 副本、CDN 当静态 fallback、ZK 协调、LB。
- 性能：MyRocks 低 P99 写延迟、multiget、Asynq（写不走 DB round trip）、Kafka 流、sharding 去热点。**垂直分片有天花板**：写吞吐成瓶颈就转水平分片。
- DR：每日异地备份 + S3 主备 zone 复制；S3 **11 个 9 durability** 但只有 **99.9% 可用性**。备份便宜但有损（窗口期丢数据）且慢（恢复数小时）→ 同步跨 region 复制补；polyglot 让各类数据独立恢复。
- 收尾口径：每个非功能需求映射到机制；DR 按 **RPO/RTO** 讲（备份窗口 vs 同步复制成本）。

## 自测 4 问（带答案要点）
1. Quora 为什么用 polyglot 存储？每种存什么？
   → 按 workload 选引擎：InnoDB 存元数据+关系（要 ACID）；MyRocks 存写密集型（LSM）；HBase 存 feed（稀疏宽表）；blob+CDN 存媒体。
2. 一致性怎么分档？
   → 关键数据（问题、投票）同步复制强一致；非关键（浏览计数）最终一致。
3. 垂直分片的天花板是什么？之后怎么办？
   → 按服务拆表简单但有上限；写吞吐成瓶颈后转 hash-based 水平分片。
4. 10 亿用户、3 亿 DAU，日增存储和服务器数？
   → 116.25TB/天 → 42.43PB/年；约 4700 台（64K RPS/台）。
