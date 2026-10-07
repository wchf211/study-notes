# Ch 14: Distributed Cache — 提炼笔记（57–62，2026-10-06）


## 57. 为什么需要分布式缓存
- RAM 比 SSD 快约 **100 倍**，且 CPU/内存与存储的速度差还在拉大；服务反复读同样的数据 → 内存缓存挡在贵存储前面是标准解法。
- 单机缓存是单点、内存有界、不能横向扩展 → 分布式：**consistent hashing** 分片（节点增减只 remap K/n 个 key），每分片 **1 primary + replicas**，挂了自动提拔。
- API 就两个：`put(key, value)`、`get(key)`。**hit rate 是衡量缓存的中心指标**。

## 58. 写策略 / 淘汰策略 / 内部结构
- **三种写策略**（必考）：
  - **write-through**：同时写缓存和 DB，强一致，写延迟高。
  - **write-back**：先写缓存，异步落 DB，写延迟最低，但可能丢数据、读到旧数据。
  - **write-around**：写直接进 DB、bypass 缓存，靠读 miss 回填——适合 write-once 数据，不污染缓存。
- 淘汰：LRU（默认选项）、LFU、MRU、MFU、FIFO；TTL 过期分主动扫描 / 被动检查。
- 内存内部：hash table（O(1) 查找）+ 双向链表（维护 LRU 顺序）+ bloom filter（低成本判 key 是否存在）。
- 部署二选一：独立缓存服务器（独立扩展、"cache as a service"）vs 跟应用同机（便宜、随应用扩展，但资源争抢、一挂全挂）。
- **Cache client**：跑在每个应用服务器里的库，知道所有缓存节点地址，所有客户端用同一套 server list + hash 算法做路由。

## 59. 高层设计
- 先分清：**KV store 是持久存储层，distributed cache 是 ephemeral 的性能层**——面试必问的区别。
- 需求：低延迟、横向扩展、高可用（缓存挂了流量打到 DB 会级联雪崩）、同一 key 读到最新数据、跑在廉价 commodity 硬件上。
- 写策略决定一致性/延迟 trade-off，没有银弹；**TTL 调得好直接降低 miss 率**（如社交 feed 用 LRU + 短 TTL）。

## 60. 详细设计
- **服务发现三档**：本地配置文件（简单，更新靠手）< 集中式配置（仍靠手+外部健康检查）< **专用 configuration service**（自动感知节点增减、通知客户端，最贵但最稳）。
- 可用性：每分片 1 primary + 2 replicas；节点距离近时写用同步复制。replica 既做 failover 也分担读（热分片）。
- Consistent hashing 的坑：**负载不均 → 用 virtual nodes 修**。
- 请求链路：client → LB → 应用服务器（内嵌 cache client）→ consistent hash 选分片 → primary/replicas；config service 保证各客户端视图一致。

## 61. 评估：EAT 公式
- **EAT = hit_ratio × hit_time + miss_ratio × miss_time**（背下来，会算）。
- 例：LRU 95% 命中 → 0.95×5ms + 0.05×30ms = **6.25ms**；MFU 90% 命中 → 7.5ms。淘汰算法选得好不好，直接体现在平均延迟上——上线前按真实 workload 实测 hit rate。
- 单 DC vs 多 DC：多 DC 可用性更高，但一致性代价上升（CAP/PACELC）；**重加入的节点同步完成前不许对外服务**。

## 62. Memcached vs Redis（对照表要会背）
- **Memcached**（2003）：shared-nothing、只存 string，路由/hash 全在客户端；多线程；无内置持久化/分片管理/复制（靠第三方）。Facebook 2013 数据：28TB 内存 / 800+ 台，挡在 MySQL 和 Web 层之间，**95% 命中率**，5000 万请求只剩 250 万打到 DB。
- **Redis**：数据结构 store（list/set/zset/hash/bitmap/hyperloglog，可原地修改不用取回-反序列化-改-存回）；持久化（AOF 日志 / RDB 快照）；Sentinel 自动 failover；Cluster 自动分片（每分片 1 primary + replicas，控制面与数据面分离）；**复制是异步的，不保证强一致**；pipelining 批量发请求，吞吐可提 \~5 倍。
- 选型口径：**简单读多、要多线程、想自己掌控 → Memcached；要数据结构/持久化/托管复制集群 → Redis**。

## 自测 4 问（带答案要点）
1. 三种 write policy 的 trade-off？
   → write-through 强一致但写慢；write-back 写最快但可能丢数据/读旧；write-around 写 bypass 缓存，适合 write-once 数据防污染。
2. EAT 公式？95% 命中、hit 5ms、miss 30ms 时 EAT 是多少？
   → EAT = hit_ratio×hit_time + miss_ratio×miss_time = 0.95×5 + 0.05×30 = 6.25ms。
3. Consistent hashing 解决什么问题？它有什么坑，怎么修？
   → 节点增减只 remap K/n 个 key；坑是负载不均，用 virtual nodes 修。
4. Memcached vs Redis 怎么选？
   → 简单读多、要多线程、开发者想自己控 → Memcached；要丰富数据结构/持久化/托管复制集群 → Redis。
