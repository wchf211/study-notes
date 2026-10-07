# Newsfeed + Instagram — 提炼笔记（146–150，2026-10-06）


## 146. Newsfeed：生成 vs 发布
- 个性化排序聚合；拆两半：**feed generation**（聚合→过滤→排序）vs **feed publishing**（hydrate + 投递）。
- 非功能：数十亿用户、容错分区容忍、**可用性压过一致性**（PACELC，feed 旧一点可接受）、≤2 秒延迟。
- **排序是算力心脏**：候选集 → 谣言/clickbait 压制 → 特征打分（历史/赞/评论/点击，1–5 分）→ 重 ML 基建（大数据管线+GPU/TPU）。
- 估算：10 亿用户、5 亿 DAU，每人 \~300 好友+250 主页；5 亿×10 次/天 = 50 亿请求/天 ≈ **58K req/s**；用户元数据 50TB；文本 0.5PB；媒体约 112MB/人 × 5 亿 ≈ **56PB**；峰值约 **8000 台**。
- **图数据库存关系**（follow/like，schema 常变；property-graph，PG+JSON 也可实现）；其余关系型：User、Entity、Feed_item、Media。
- 链路：图库取 follow IDs → user cache hydrate → post cache 取最新/热门 → ranking 打分 → **<Post_ID, User_ID> 元组**进 newsfeed cache → 发 top N + 分页。热门媒体走 CDN。
- API：generateNewsfeed（内部异步）、getNewsfeed(user_id, count)。

## 147. Instagram 开场
- 工作流：上传、feed 消费、timeline 生成、搜索/发现、stories、直播。反复出现的主题：LB、限流、Kafka、NoSQL vs 关系型、分片复制。

## 148. 需求（20 亿用户 / 8 亿 DAU）
- 功能：传/下/看照片、按标题/用户名搜、关注、按关注生成 feed、**stories 24 小时可见**。
- 写：日 1 亿照片（1157/s）、5 亿 stories（5787/s）；**读写比 100:1** → 缓存优先、CDN 优先的架构理由。
- 存储：照片 2MB、story 1MB → 日增 **700PB**（课内算术）；带宽约 8GB/s；用户元数据 100TB；关注列表 36PB。
- 一致性可最终（用户容忍延迟），**durability 必须**（照片不能丢）。

## 149. 设计：从朴素迭代
- 基线：LB → app → RDBMS（元数据）+ blob（媒体）。上传：client→LB→app→blob 存二进制 + RDBMS 存元数据；看/搜：app 配元数据，从存储取二进制。
- 优化：**读写服务拆分**（读远多于写）→ 缓存 + **lazy loading**（滑到哪加载到哪）。
- **Cassandra 存照片元数据**：日千万级写、schema 灵活（加 hashtag 不用迁移）、写吞吐高；代价是放弃关系一致性/join。
- Schema：User、Photo（PhotoID/UserID/日期/路径/位置）、UserFollow。
- **按 hash(photoID) 分片**：分布均匀 + 直接定位。API：uploadPhoto/downloadPhoto，网关验 auth token。

## 150. 详细：fan-out 三选一（必考）
- **Pull**（fan-out on load）：打开 app 时现算，延迟高；读多写少时浪费（多数人从不发帖，白算）。
- **Push**（fan-out on write）：发帖时预计算推给粉丝，读极快；**明星发帖写放大**（一条推写数百万 timeline，多半给僵尸粉）。
- **Hybrid（正解）**：普通人 push、**明星 pull**，读时合并预推 + 现拉。
- Timeline 存 **KV**：key=userID，value=帖子链接列表；超大 value 再分片。
- Stories：timestamp 列 + **TTL/定时任务**自动删。
- CDN：读走 edge，写走 write server。病毒式点赞数用 **sharded counters**。

## 自测 4 问（带答案要点）
1. Pull / push / hybrid fan-out 怎么选？
   → pull 读时算延迟高，读多写少浪费；push 写时预计算读快，但明星写放大；hybrid 普通人 push、明星 pull，读时合并。
2. Instagram 照片元数据为什么用 Cassandra 不用 RDBMS？
   → 日千万级写、schema 灵活（加 hashtag 不迁移）、写吞吐高；代价是放弃关系一致性/join。
3. Newsfeed 里图数据库存什么？关系型存什么？
   → 图存关系（follow/like，schema 常变）；关系型存 User/Entity/Feed_item/Media。
4. Stories 24 小时过期怎么实现？
   → timestamp 列 + TTL/定时任务自动删。
