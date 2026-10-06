# Twitter — 提炼笔记（140–144，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 140. 开场
- 五步法：需求 → API/数据模型 → 组件+估算 → 瓶颈/trade-off → 改进/监控。
- 四个先抛出的挑战：极端读多、**fan-out 是硬骨头**（明星推文瞬间到数百万粉丝，thundering herd）、timeline 可最终一致、ID 要唯一且大致时间有序（Snowflake 式）。
- 规模：日 5 亿推、3.68 亿 MAU。

## 141. 需求：1:1000
- 功能：发推（文本+媒体）、删、赞踩、回复、搜索、**home timeline**（关注的人）vs **user timeline**（某人自己的）、关注/取关、转推。
- 非功能：HA、feed 读低延迟、**写读比 1:1000**（主动报这个数，后面所有决策都从它推出：缓存、fan-out-on-write、反范式化）、durability、最终一致可接受。
- Trade-off：feed 选 AP 不选 C。

## 142. 高层：API 先行
- 链路：DNS → LB → 应用服务器 → SQL/NoSQL，CDN 缓存热点。
- API：postTweet、like/dislike、replyTweet、searchTweet、viewHome_timeline、follow/unfollow、retweet/undoRetweet。
- 推文 ID 用 Snowflake 式（时间有序 → timeline 排序便宜）。
- 搜索分两档：**近 7 天实时** vs 全量归档；上限 3200 条（不含回复 800）；**token 分页**（next_token）处理无界 timeline。
- 口径：先摆 API 签名，逼自己把实体定死（用户/推文/关注/赞），数据模型自然展开。

## 143. 详细：polyglot persistence
- 没有单一数据库，按 workload 配：
  - HDFS/Google Cloud **300+ PB**：日志/事件/备份，LZO 压缩，BigQuery 经 Presto 查——冷数据上云、实时生产留本地（"partly cloudy"）。
  - **Manhattan**：自研分布式 KV（MySQL 到顶、Cassandra 不合身之后造的）。
  - 自研 blob store 存大媒体；MySQL/PG 留给真要强一致的（广告/计费）。
  - Kafka + Dataflow：日 **4000 亿**实时事件进 BigQuery/Bigtable。
  - **FlockDB**：图存关注/拉黑关系；**Lucene**：约 **1 万亿**记录、**100ms** 内返回（近一周 in-RAM 实时索引 + 磁盘归档索引）。
- 配套：Pelikan 统一缓存（twemcache/slimcache）、ZK 动态配置、Splunk 日志、Zipkin tracing、Scribe 聚合。
- 硬骨头：明星推文 spike；**sharded counters 放用户附近**（CDN-like）吸计数热点；top-k 热榜用本地聚合+全局聚合 over sliding window。

## 144. Client-side LB：deterministic aperture
- 中心化 LB 自己成瓶颈 → 路由逻辑下沉到客户端（Finagle 的 deterministic aperture）。
- 请求分发：**P2C**（Power of Two Choices）——随机抽两个实例发给较空的，"比纯随机指数级更均匀"。
- 会话分发四代演进：mesh（公平，不扩展）→ random aperture（扩展，不公平）→ 离散环 deterministic（弱公平）→ **连续环坐标**（可扩展+公平，共享后端时按重叠加权）。
- 核心 tension：**公平性 vs 连接成本的可扩展性**——面试官追问 LB 格子时的高级牌。

## 自测 4 问（带答案要点）
1. 1:1000 的读写比能推出什么设计决策？
   → 缓存、fan-out-on-write、feed 最终一致、反范式化，几乎所有下游决策都从它来。
2. Celebrity tweet 的 thundering herd 怎么解？
   → fan-out 是硬骨头；sharded counters 放用户附近（CDN-like）吸热点；top-k 用本地聚合+全局聚合 over sliding window。
3. Polyglot 里图关系存哪？热 key 存哪？媒体存哪？
   → FlockDB 存关注/拉黑；Pelikan 缓存热 key；自研 blob store 存大媒体。
4. Deterministic aperture 解决什么问题？P2C 是什么？
   → 中心化 LB 成瓶颈，路由下沉客户端；P2C 随机抽两个实例发给较空的，比纯随机指数级更均匀。
