# Ch22 + 方法论：Sharded Counters / RESHADED — 提炼笔记（101–103, 108，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 101. 单计数器为什么不行
- 单计数器 = 所有递增串行在排他锁后面，并发一上来**锁竞争非线性增长**，拿锁的开销超过更新本身。
- **Heavy hitters** 雪上加霜：极少数帖子（明星推文）吸走绝大多数并发写，形成热点，吞吐到顶、延迟起飞。
- 规模感：Twitter 约 6000 tweets/s、日 5 亿条、点赞数十亿级。
- 口径："为什么不用 DB 一行计数？"→ 锁竞争 + 热点，标准答案。

## 102. Sharded counter
- 单计数器换成 **N+1 个分片**，各在不同节点独立递增；写时选个分片（随机/hash），**读时 sum 所有分片**。
- Trade-off：写没竞争了，读变贵（要 fan-out 聚合）。
- API：createCounter(id, shards)、writeCounter（增/减）、readCounter。
- **分片数在创建时定**，按预期负载（粉丝数、帖子类型等启发式）预估。

## 103. 调参与边界
- 分片数：太少 → 竞争回来；太多 → 读放大（跨更多分片/节点/地域聚合）。热度是 bursty + 长尾 → **按监控反馈动态增减分片**。
- 路由三选：round-robin（简单， skewed 负载下崩）/ random（只有成本均匀时才均匀）/ metrics-based（专用节点看分片状态挑）。
- 读路径：**别每次都 sum**（延迟/吞吐扛不住）→ 定期聚合 + 缓存结果；聚合间隔越短越准 → 这就是**最终一致**，金融余额这种要精确的场景**不适用**——主动说出不适用的边界是加分项。
- Top-K/趋势扩展：按 hashtag 设计数器（按预期量定分片）+ 按 region 设；region 趋势如窗口内过 1 万触发；home timeline 按粉丝数+新鲜度排。
- 落地：Cassandra 存聚合后的 region 求和，Redis/Memcache 存映射（tweetId→counterIds），定期 snapshot 回 Cassandra 供读。

## 108. RESHADED（面试答题骨架）
**R**equirements → **E**stimation → **S**torage schema（可选）→ **H**igh-level design → **A**PI design → **D**etailed design → **E**valuation → **D**istinctive component/feature。
- 每道系统设计题都按这个顺序走：需求先行，估算驱动选型，高层设计是草稿（可迭代），详细设计从高层的短板出发收敛。
- 最后一个 D 是点睛：**说出这道题的独特 twist**（Uber 的欺诈检测、Google Docs 的并发控制）——区分 good 和 great 的一句话。

## 自测 4 问（带答案要点）
1. 单计数器为什么撑不住？
   → 排他锁串行化，锁竞争非线性增长；heavy hitters 造成热点，吞吐到顶。
2. Sharded counter 的读写 trade-off？
   → 写分散到分片无竞争；读要 fan-out 求和变贵 → 定期聚合+缓存，换最终一致。
3. 分片数怎么定？太多太少各有什么问题？
   → 创建时按预期负载（粉丝数等）预估，运行中按监控反馈动态增减；太少竞争回来，太多读放大。
4. RESHADED 八个字母分别是什么？最后一个 D 为什么重要？
   → Requirements / Estimation / Storage schema(可选) / High-level / API / Detailed / Evaluation / Distinctive；最后 D 要点出题目的独特 twist，是 great 答案的标志。
