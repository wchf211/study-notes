# Basic Concepts + Theorems（Module 3）

> 顺序就是学习顺序：先机制（partition → replication → quorum），再形式化（safety、ACID、CAP、consistency、isolation），最后统一（consistency vs isolation、模型层级、为什么要形式化）。别跳过形式化去抄 case —— 形式化是用来读厂商文档的。

## 1. Partitioning：切分数据/工作

**Partitioning / sharding（分区/分片）**：把数据/工作切到多个节点，每个节点只持有一部分。动机：单节点装不下 / 扛不住；并行 + 局部性更好。

原则：选 partition key（分区键）和方案，让负载均衡、相关数据放一起。负载不大别提前分区。例：100 GB 图书目录按 key 范围切。失败模式：hot partition（热分区）/ key 倾斜。Fix：rebalance（重平衡）、更好 key、拆热区间。

**水平分区四种算法**：
- Range partitioning（范围分区）：连续 key 区间，好 scan（范围扫描），热区间风险。
- Hash partitioning（哈希分区）：hash(key) mod N，分布均匀，范围扫描差。
- Consistent hashing（一致性哈希）：环上放节点，加/删节点时重映射最少，需 virtual nodes（虚拟节点）补均衡。
- Lookup / directory（目录）：显式映射表，灵活，多一跳查询。

Trade-off：scan 局部性 vs 均衡 vs churn（节点变化时的搬动量）。点查用 hash / 一致性哈希；有序扫描用 range；要精细控制放 directory。失败模式：naive mod 下加节点全量重洗牌。Fix：一致性哈希或 directory。

## 2. Replication：多副本 + 一致性工作

**Replication（复制）**：同一份数据存多份。动机：扛节点挂、就近读、分摊读负载。代价：写时要做一致性工作。没有 divergence（分歧）预案就别复制。

例：用户画像三副本，一个挂了读还能继续。失败模式：读到 lagging replica（滞后副本）的旧数据。Fix：read-your-writes（读己之写）、quorum read（quorum 读）、或更强的同步。

**Primary-backup（一主多备）**：一个 primary 排序 + 应用写，ship log / state 给 backup，backup 按序应用；primary 挂了提一个 backup。Sync（同步）：等 backup 再 ack —— 更强、更慢。Async（异步）：先 ack —— 更快、可能丢最近的写。

适合简单单写工作负载。Primary 是瓶颈或 failover（故障转移）必须瞬时时别用。失败模式：分区后两个 primary（split brain，脑裂）。Fix：fencing（隔离）、quorum election（quorum 选主）、epoch number（任期号）。

**Multi-primary（多主）**：多个节点同时接受写，再 reconcile（调和）。动机：多区域写可用 / 分区时可用。原则：并发写会冲突，要有解决规则（last-write-wins、merge、CRDT、人工）。可用性 > 立刻一致、冲突少或可合并时用。计数器 / 钱这类没安全 merge 的别用。例：两个区域改不同用户画像能干净合并；改同一字段冲突。失败模式：last-write-wins 下 lost update（更新丢失）。Fix：version vector（版本向量）、CRDT、或 per-key 单写。

**Quorum（仲裁）**：N 副本，写要 W 个 ack，读要 R 个，R + W > N，保证任意读集合与任意写集合 overlap（交集），读到最新写。调 R/W 做读写偏向。适合：不用全同步也能调一致性。别假设 R + W ≤ N 时 quorum 读一定新鲜。例：N=3，W=2，R=2，至少交于一个节点。失败模式：持最新写的节点不在读 quorum 里。Fix：抬 R 或 W，或 read repair（读修复）。

## 3. Safety：不变量是可测试的

**Safety guarantee（安全保证）**：永不违反的 invariant，如唯一性、已提交写不丢、no double-spend。Safety 限制哪些历史是允许的。原则：fault / partition 下 safety 也要成立，liveness 可以降级。

把 safety 当代码 / review 里的可测试 invariant。别把 safety 和性能混淆。例：两个 leader 绝不能同时认为自己是 leader。失败模式：stale leader（过期 leader）在新 leader 选出后还发写。Fix：fencing token / epoch。

## 4. ACID：四个保证是打包的，别按标签信

**ACID**：Atomicity（原子性，全有或全无）、Consistency（一致性，事务把 DB 从一个合法态带到另一个合法态）、Isolation（隔离性，并发事务看不到彼此的半成品）、Durability（持久性，已提交的 survives crash）。

ACID 是打包的几个保证，厂商实现的是子集。多步更新绝不能部分持久时用 ACID。别假设标签 = 最强隔离。例：借记 + 贷记必须同生同灭。失败模式：中途 crash，钱被凭空创造/丢失。Fix：atomic commit（原子提交）+ durable log / recovery（持久日志/恢复）。

## 5. CAP：分区时二选一，不是日常三选二

**CAP theorem（CAP 定理）**：网络分区时，系统不能同时保证 strong consistency（强一致：每次读都看到最新写）和 full availability（完全可用：每次请求都得非错误回复）。P（分区容忍）不是可选项 —— 分区会发生。

原则：CAP 是“分区时的行为”，不是日常运行。分区下选 CP（拒绝 / 阻塞保一致）还是 AP（服务可能旧的数据）。别把系统标成“CA”。例：分裂的两边不能既接受冲突写又保持一致。失败模式：宣称分布式存储是 CA。Fix：写明 CP 还是 AP + 分区时行为。

**CAP 里的 C = linearizability（线性一致性）**：操作看起来在开始和结束之间的某个瞬间原子生效，按 real-time 顺序。这是最强的单寄存器保证。协调类工作（锁、选主、唯一 ID）要用它。别把它等同于 serializability（那是多操作事务的），也别等同于“所有副本永远相同”。例：写完成后的读必须看到该写。失败模式：ack 后的新写，旧副本还服务旧值。Fix：quorum / consensus 或 leader 读。

## 6. Consistency Models：契约，按“应用能忍”选最弱的

**Consistency model（一致性模型）**：并发 / 复制下读者能观察到什么的契约：strong / linearizable（读起来像只有一个副本，按实时序）、sequential（顺序一致）、causal（因果）、eventual（最终一致）、及更弱变体。

原则：越强越好推理、成本/延迟越高；越弱越快，但应用要能忍 anomaly（异常）。用“应用能正确忍受的最弱模型”。别不声明就混用模型。例：社交 feed 可以 eventual；银行余额要更强。失败模式：以为 eventual = “很快”有界。Fix：量 lag，需要时加 read-your-writes。

## 7. Isolation：Anomaly 表格是选型工具

**Isolation level（隔离级别）** 控制并发事务能观察到什么。**Anomalies（异常）**：dirty read（读到未提交数据）、non-repeatable read（事务中值变了）、phantom read（新行冒出来）、lost update（两个写互相覆盖）、write skew（两个事务各自保 invariant，合起来破了）。

越弱的隔离 admit（容纳）越多 anomaly，换速度。按事务故意选级别。别假设“serializable”标签准确。例：两个医生各看到一个空闲 on-call 名额，都约上 = write skew。失败模式：并发递增 lost update。Fix：更强隔离、原子操作、或 OCC 重试。

**Prevention（防异常）**：read-uncommitted 几乎都放行；read-committed 拦 dirty read；repeatable-read 拦 non-repeatable read（常拦 lost update）；snapshot isolation 拦很多但不拦 write skew；serializable 全拦。“防止”是按 anomaly 集合定义的，不是营销名。别假设某级别防它没声称的 anomaly。例：snapshot isolation 仍允许 write skew。失败模式：信旧 ANSI SQL 标签。Fix：查厂商实际 anomaly 保证 + 测试。

## 8. Consistency vs Isolation + 模型层级 + 为什么形式化

**统一视角**：两者都关心并发操作能观察到什么。Consistency 是单操作在复制数据上；isolation 是多操作事务。Sequential consistency 泛化后 = 事务的 serializable isolation。用一套词汇想分层系统。别当成不相关的旋钮。例：对客户端看起来 serializable 的事务，底层 store 至少要 sequentially consistent。失败模式：强事务层 + 弱一致 store 漏 anomaly。Fix：对齐各层或加协调。

**Hierarchy（层级）**：linearizability > sequential > causal > eventual；serializable > snapshot > repeatable-read > read-committed > read-uncommitted（有些分支不可比）。强蕴含弱（linearizable 的 store 也满足 sequential），反之不成立。用来比厂商声称。别假设同层两个模型等价。例：causal 强于 eventual、弱于 sequential。失败模式：选了“强”标签却强在另一个维度。Fix：点名精确模型 + 它的 anomaly。

**Why formalities（为什么要形式化）**：形式模型给 safety / consistency / isolation 精确、可比的定义，才能检查 store 是否满足应用需求。含糊术语藏不同保证。读文档时用形式名，怀疑厂商声称。别信没定义的“consistent”或“serializable”。例：老牌 DB 的“serializable”仍允许 anomaly。失败模式：按 buzzword 选 store，上线发现 anomaly。Fix：读文档的精确模型 + 测试 + 要求厂商明确保证。

## 自测 4 问（带答案要点）
1. R + W > N 为什么保证读到最新写？N=3 时举一组 R/W。
   → 读集合与写集合必 overlap；N=3，W=2，R=2。
2. CAP 的 C 到底是什么？为什么不能标 CA？
   → linearizability（实时序原子生效）；P 不是可选项，分区时只能 CP 或 AP。
3. Snapshot isolation 防什么、不防什么？
   → 防很多，不防 write skew；按 anomaly 表格选级别，别信营销名。
4. Consistency 和 isolation 的关系？强模型和弱模型的关系？
   → 前者单操作复制数据，后者多操作事务；sequential ≈ 事务 serializable；强蕴含弱，反之不成立。
