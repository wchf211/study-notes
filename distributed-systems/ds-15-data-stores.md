# Distributed Data Stores（Module 17）

> 这一章是比较，不是百科。四种 store 各答同一批问题：谁给写排序？读看到什么？分区下会坏什么？读法：每种 store 先看数据模型 + 架构 → 操作 → guarantee → gotcha（坑）。最密的是 Cassandra 的 linearizability 分析和 Spanner 的 TrueTime / commit-wait 推理。

## 1. 开篇：没有普遍最好的 Store

要比较的 store：HBase / Bigtable、Cassandra、Spanner、FaunaDB。原则：不同 store 在数据模型、partitioning、replication、consistency 上做不同 trade-off —— 没有普遍最好的 store。

用这章学怎么按 workload + guarantee 评估 store。别按流行度选 store。例：时间序列用 wide-column store，事务用 relational store。失败模式：选了 store 才发现 consistency model 不合。Fix：先点名需要的 guarantee。

## 2. Bigtable / HBase 架构：排序 Key + LSM + Range Partition

**Bigtable / HBase**：sparse（稀疏）、sorted（排序）、multi-dimensional map（多维映射）：row 按 row key，group 进 column family（列族），每 cell（单元格）带 timestamped version（带时间戳版本）。物理布局：表按 key range 拆成 region / tablet（区域）；master 协调 region 分配，region server 服务 region，coordination service（HBase 里是 ZooKeeper）跟踪 membership。数据存在分布式文件系统的 LSM 文件（HFile / SSTable）里。

原则：排序 key + LSM + range partitioning = 可扩展的 scan（扫描）和写。Large、sparse、append-heavy 表 + key-based 访问时用。Ad-hoc join 或多行事务别用。例：按 URL 键的 web-crawl 表，列按 content / language。失败模式：sequential key 导致 hot region。Fix：key salting / 更好的 key 设计或 pre-splitting（预拆分）。

## 3. HBase 写 + 读：LSM 用读放大换快写

**写**：先 append 到 WAL（write-ahead log，预写日志）保 durability，再 insert 到 in-memory memstore（内存存储）；memstore 满了 flush（刷盘）成 immutable sorted file（HFile）到 disk；periodic compaction（定期合并）合并文件、丢过期版本。**读**：查 memstore + HFile，按 timestamp 合并版本，用 Bloom filter / block index 跳过无关文件。

原则：LSM 用 read amplification（读放大）换快顺序写。Write-heavy、key-based workload 用 HBase。别指望 in-place update（原地更新）—— 每次写都是新版本。例：一个 put 变成 WAL entry + memstore entry，后来变成文件。失败模式：太多小文件拖慢读（读放大）。Fix：compaction 调优 + Bloom filter。

## 4. HBase Guarantee：强，但窄 —— 一行、一 Region

**Single-row atomicity（单行原子性）**：一个 put 到一行是原子的（一行的所有列一起更新或都不更新）；没有多行事务。**Strong consistency per row（每行强一致）**：读看到该行最新 committed 写（单个 region server 排序）。**Durability** 经 WAL + 底层文件系统 replication。核心模型里没有 secondary index（二级索引）或 join。

原则：强但窄 —— 一行、一 region。Per-row 原子性 + 强读够用时用 HBase。别在上面搭多行事务还指望原子性。例：原子更新一个用户 profile 的几个列。失败模式：假设两行原子更新。Fix：设计单行 key，或用有多行事务的系统。

## 5. Cassandra 数据模型：为 Query 建模，不是为 Normalisation

**Cassandra**：wide-column：keyspace → table → row 按 partition key（哈希到 ring 上的 token），partition 内 clustering column（聚簇列）给行排序。没有 join；查询必须围绕 partition key 设计（query-first modelling，查询优先建模，denormalisation，反规范化，materialised view，物化视图）。

原则：为你的查询建模，不为 normalisation（规范化）。High-write、partition-key-based 访问、大规模时用 Cassandra。跨 partition 的 ad-hoc 查询别用。例：按 user_id 分区的表，帖子按 timestamp 聚簇。失败模式：按非 key 列查询扫描所有 partition。Fix：为该查询建专用表 / 物化视图。

## 6. Cassandra 集群通信：Gossip + Phi Failure Detector

**Peer-to-peer（点对点），无 master**：每个节点都能服务读/写。Membership（成员）和 failure detection（故障检测）用 gossip（节点定期和随机 peer 交换状态，epidemic 式传播成员信息）+ failure detector（phi accrual，phi 累积，从 heartbeat 到达模式估节点挂掉的可能性）。Seed node（种子节点）引导 gossip。

原则：gossip 去中心化、健壮，代价是 eventual convergence（最终收敛）。不想要单 coordinator 时用 gossip。别把 gossip 状态当 instant truth（即时真相）。例：新节点加入经 gossip 传播到整个 ring。失败模式：慢节点被误判死亡、过早移除。Fix：调 phi 阈值 + hinted handoff（提示移交）。

## 7. Cassandra Consistency Level：每操作可调，QUORUM ≠ Linearizable

Per-operation tunable consistency（一操作可调一致性）：ONE（一个 replica）、QUORUM（多数）、ALL（全部），外加 LOCAL_QUORUM、EACH_QUORUM、SERIAL / LOCAL_SERIAL（给 lightweight transaction）。强读规则：read quorum + write quorum > replication factor 得 overlap（重叠），（加 repair）得 fresh read。

原则：按 query 调 —— 关键读/写更强，可用/延迟优先的更弱。QUORUM / QUORUM 当默认强一点的设置；ONE 给 best-effort。别假设 QUORUM 单独 = linearizable（见下两课）。例：RF=3，QUORUM 写 + QUORUM 读至少重叠一个 replica。失败模式：ONE / ONE 在分区下服务 stale 数据。Fix：需要新鲜的操作提高 level。

## 8. Cassandra 的 Linearizability Violation：Quorum 重叠必要但不充分

哪怕 QUORUM 读 + 写，Cassandra 也不是 linearizable：读可能错过已完成写（读 quorum 没包含见过它的 replica，read repair 还没传播）；并发写可能被不同 replica 不同排序（last-write-wins 按 timestamp 可能复活已删数据 / 丢更新）；clock skew（时钟偏差）让“last”任意。

原则：quorum overlap 对 linearizability 必要但不充分 —— 还要单个 total order + 返回前 read repair。把 Cassandra 的 quorum 当“带重叠的最终一致”，不是线性化。别声称 QUORUM = linearizable。例：写 ack 自 replica A、B；后来的读自 B、C，C stale、B 的值还没 repair，返回旧数据。失败模式：靠 QUORUM 做 leader election / 唯一 ID。Fix：SERIAL / LWT 或线性化 store。

## 9. Cassandra 的 Linearizability Guarantee：LWT，只为少数操作付 Paxos 代价

需要 linearizability 的操作，Cassandra 给 lightweight transaction（LWT），SERIAL / LOCAL_SERIAL 一致性，经 Paxos-like 协议（propose → prepare / promise → propose / accept → commit）跨 replica。给单个 partition 的 linearizable compare-and-set（比较并设置）。

原则：只为少数需要 linearizability 的操作付 Paxos 代价。LWT 用在“insert if not exists”/ 条件更新。别每写都用 LWT —— 慢得多、contention（竞争）大。例：IF NOT EXISTS 抢唯一用户名。失败模式：竞争的 LWT 反复 abort / retry。Fix：竞争 key 保持少、操作保持短。

## 10. Cassandra 查询效率：Data Model 就是 Index

效率来自 query-first modelling：选 partition key 让每个查询只碰一个 partition；用 clustering order 免排序；secondary index 慎用（每个节点本地，会 scatter 查询）；用 materialised view / denormalised 表给 alternate access path（替代访问路径）；ALLOW FILTERING 只用于小数据。

原则：在 Cassandra 里，data model 就是 index。Denormalisation 买读速度。别事后加 index 应对任意查询。例：按 (user_id, timestamp) 键的表，一 partition 服务“某用户帖子、最新优先”。失败模式：查询 fan-out 到所有节点再 filter。Fix：为该查询建专用表。

## 11. Spanner 数据模型：关系型 + 全球分布

**Spanner**：relational（关系型）：database → table 带 row、primary key、schema、SQL、secondary index、interleaved table（交错表：子行和父行物理上 co-located，共置，保 locality）。数据按 key range shard 成 tablet，tablet 可跨 zone / region 复制。原则：关系模型 + 全球分布，locality 靠 interleaving。

需要 SQL + 事务 + 全球规模时用 Spanner。别把它当 wide-column store。例：Customers 表 + interleaved Orders 行存在每个客户旁边。失败模式：随机 primary key 打散相关行。Fix：interleaving + locality-aware key 设计。

## 12. Spanner 架构：每 Tablet 一个 Paxos 组 + 跨组 2PC

分层：tablet（key-range shard）存在 Colossus（分布式文件系统）上；每个 tablet 的 replica 组成一个 Paxos group（组），有 leader；lock table + transaction manager 跨 Paxos 组协调 two-phase commit（2PC），做多 shard 事务；directory layer（目录层）把 key 映射到 tablet。原则：每 tablet Paxos 做 replication + 跨 tablet 2PC 做事务。

全球一致的关系数据用 Spanner。别假设单个 Paxos 组持有整个数据库。例：触及两个 tablet 的事务经跨组 2PC 提交。失败模式：跨 tablet 事务更慢。Fix：co-locate 相关数据（interleaving）+ 事务保持窄。

## 13. Spanner TrueTime：把不确定性显式暴露

**TrueTime** 是 clock API（时钟 API），返回 time interval（时间区间）[earliest, latest]，保证包含真实时间，用 GPS + atomic clock（原子钟），bounded uncertainty（有界不确定性，典型几毫秒）。操作：TT.now() 返回区间；TT.after(t) 等到 t 一定过去；TT.before(t) 等到 t 一定未来。

原则：显式暴露不确定性，不假装时钟精确。用 TrueTime 区间给外部事件排序。别把单个 timestamp 当精确时间。例：commit timestamp 选 TT.now().latest，然后 commit 等出不确定性。失败模式：忽略不确定性，给几毫秒内的两个事件排序。Fix：等出区间（commit wait）。

## 14. Spanner 操作：TrueTime + Commit Wait = 全球有序可串行化事务

**读**：单 tablet 读走 Paxos leader 或 timestamp 上的 snapshot read（stale 读允许换更低延迟）；多 tablet 读用 2PC。**写**：leader 从 TrueTime 分配 commit timestamp，经 Paxos 复制，再等出不确定性（commit wait），这样后来的事务拿到更大 timestamp。**External consistency（外部一致性）**：事务 T2 在 T1 commit 后（真实时间）开始，T2 的 timestamp > T1 的。

原则：TrueTime + commit wait = 全球有序可串行化事务。External consistency 重要时用 Spanner（如跨 client 排序）。别指望零延迟 —— commit wait 要几毫秒。例：两个大陆的 client 看到事务按同一真实时间顺序。失败模式：忽略 timestamp 的读看到 stale snapshot。Fix：timestamp bound 或 leader 读。

## 15. FaunaDB / Calvin：Determinism 替代 Agreement

**FaunaDB** 建在 Calvin 协议上：复制 transaction input（输入），不是 effect（效果），所有非失败节点执行同样确定性序列、到达同样状态，不用 2PC。分层：sequencing（经共识给输入排序，batch 成 epoch，约 10 ms）、scheduling（按 log 顺序确定性 locking，允许并发）、storage（物理布局、多版本）。每个节点扮演三角色：query coordinator（收请求、speculative 执行找读/写 intent —— Optimistic Lock Location Prediction, OLLP）、data replica（存数据）、log replica（排序输入）。

读-写事务经 deterministic pessimistic locking（确定性悲观锁，deadlock-free，因为锁顺序跟 log 顺序）严格可串行化；只读事务可串行化，或经 linearized endpoint（no-op 写）严格可串行化。没有 interactive transaction（交互式事务，只支持一次声明；经 snapshot 读 + compare-and-swap 模拟）。

原则：determinism（确定性）替代 agreement。想要可串行化多 region 事务、不用 2PC 时用 FaunaDB。别指望交互式多语句事务。例：多 shard 转账在 log 里排序一次，所有 replica 一样执行。失败模式：speculative 读/写集到执行时变了 → 事务（全体一致）abort 并重试。Fix：retry + 事务保持短。

## 这门课怎么教的（Module 17）

Comparative（三系统比较）结构：HBase / Bigtable（单 master、LSM、强 per-row）、Cassandra（点对点、可调 quorum、默认最终一致，带一段仔细的 linearizability 绕行）、Spanner（全球分布、TrueTime、外部一致性），加 FaunaDB / Calvin 当 determinism 替代。每个系统都走：数据模型 → 架构 → 操作 → guarantee → gotcha。课程反复对每个 store 问同样的问题（谁给写排序？读看到什么？分区下会坏什么？），比较是 apples-to-apples（同标准）。最密的是 Cassandra 的 linearizability 分析和 Spanner 的 TrueTime / commit-wait 推理。

值得偷的：**同一批问题问每个系统**。讲比较课时，不要每个系统换一套问题；固定问题清单，观众自己就能比。**“谁排序、读看到什么、分区下坏什么”** 是评价任何 store 的三问，要钉在黑板上。**绕行（detour）要标出来**：Cassandra 的 linearizability 绕行是“QUORUM ≠ linearizable”这个反直觉结论，讲的时候要先给结论，再给反例。

## 自测 4 问（带答案要点）
1. 评价一个 store 的三问？
   → 谁给写排序？读看到什么？分区下会坏什么？
2. 为什么 QUORUM ≠ linearizable？
   → Quorum 重叠必要但不充分；还要单个 total order + 返回前 read repair；并发写排序不同 + 时钟偏差。
3. Spanner 怎么做到外部一致性？
   → TrueTime 区间 + commit timestamp 选 latest + commit wait 等出不确定性。
4. Calvin / FaunaDB 用什么替代 2PC？
   → Determinism：复制事务输入，所有节点执行同样确定性序列；锁顺序跟 log 顺序，deadlock-free。
