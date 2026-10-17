# Distributed Transactions（Modules 4–7）

> 结构就是一张组合表：隔离机制（2PL / OCC / MVCC / SSI）× 原子性机制（2PC / 3PC / quorum commit）= 完整的分布式事务。先分开学，再看怎么组合；组合错了（如提前放锁）会漏未提交数据。

## 1. 为什么单机 ACID 不能免费扩展（Module 4）

**Distributed transaction（分布式事务）**：一个事务触及多节点数据（如一个 shard 上借记、另一个上贷记）。单机 ACID 的想法仍然适用，但 commit 和并发现在要跨失败和分区协调。

什么时候用：多节点更新必须 all-or-nothing + isolated。别假设本地事务库能给这个。例：转账触及两个 account shard。失败模式：一个 shard 提交、另一个 abort → 钱凭空出现/消失。Fix：atomic commit + isolation 协议（Modules 5–6）。

## 2. Isolation：Serializability 是基准（Module 5）

**Serializability（可串行化）**：并发执行的结果等于某个串行顺序执行同批事务的结果。它是隔离的正确性目标；更弱的级别按“允许哪些 anomaly”来定义。拿它当比较 PCC / OCC / MVCC / SSI 的基准。别和 sequential consistency（单操作）混淆 —— serializability 是关于多操作事务。例：两个并发转账交错，但最终结果像一个跑完再跑另一个。失败模式：不存在等价串行顺序的执行（如 write skew）。Fix：用 serializable 机制。

**PCC / 2PL（悲观并发控制 / 两阶段锁）**：用之前先锁；**Two-Phase Locking（两阶段锁）**：growing phase（加锁，不释放）→ shrinking phase（释放，不再加锁）。读用 shared lock（共享锁），写用 exclusive lock（排它锁）；strict 2PL 把写锁持有到 commit / abort，abort 才干净。

冲突频繁时用。长锁杀吞吐时别用。例：两个写同一行 → 第二个 block 到第一个提交。失败模式：deadlock（死锁：A 等 B，B 等 A）。Fix：timeout / abort、锁顺序、或死锁检测。

**OCC（乐观并发控制）**：不锁；在 private copy（私有副本）上执行，提交时 validate（校验）中间没有冲突事务提交，再写。阶段：read → validate → write。冲突少、读多时用。重度 contention（竞争）下别用 —— abort / retry 风暴。例：很多人读 catalog，很少同时改同一 item。失败模式：contention 下反复校验失败。Fix：回退到锁、backoff（退避）、或分区 hot key。

**Snapshot Isolation（快照隔离）**：**MVCC（多版本并发控制）** 下每个事务从开始时刻的一致快照读，写产生新版本而不是原地覆盖。First-committer-wins（先提交者胜）阻止两个并发写同一行。

读多、需要稳定读、不用读锁时用。别当成 serializable —— 它仍允许 write skew。例：报表读稳定快照，写同时进行。失败模式：两个事务各查一个共享 invariant 都通过、都提交（write skew）。Fix：SSI、显式锁、或 serializable 模式。

**SSI（Serializable Snapshot Isolation，可串行化快照隔离）**：保留 snapshot isolation 的廉价读，加 dangerous pattern（危险模式）检测 —— read-write / write-read antidependency（反依赖）可能形成环，即 write skew 的前兆 —— 检测到就 abort 一个事务。原则：只 abort 有风险的 pattern，不 abort 所有并发。

想要 serializable 正确性 + snapshot 性能时用。别指望零 abort —— false positive（误报）会发生。例：两个 on-call 预订事务读对方的名额 → 一个 abort。失败模式：高 contention 下 spurious abort（误 abort）。Fix：retry（幂等）+ 事务保持短。

## 3. Atomicity：Agreement 是难点（Module 6）

**Atomicity（原子性）跨节点**：所有参与者一致 commit 或一致 abort，哪怕节点 crash、消息丢。原则：agreement 是难点；本地原子性（write-ahead log，预写日志）不够用。

分布式更新绝不能部分持久时用 atomic commit 协议。别靠“best effort + 希望”。例：转账借记了一个 shard，credit 另一个之前 crash。失败模式：部分节点提交、部分 abort → 不一致态。Fix：commit 协议（2PC / 3PC / quorum）+ 日志恢复。

**2PC（两阶段提交）**：coordinator 问所有参与者 prepare / vote（yes/no，带 durable log），全 yes 才发 commit，否则 abort。投 yes 的参与者必须等决策。原则：unanimous vote（一致投票）+ durable decision（持久决策）= crash 下的原子性。

小而短、要强原子性的事务用。Coordinator 失败或长阻塞不可接受时别用。例：两个 DB shard 都 prepare，coordinator 提交两者。失败模式：prepare 后 coordinator crash → 参与者 block，不知该如何。Fix：timeout + 恢复，或用非阻塞协议 —— 但 2PC 本身会 block。

**3PC（三阶段提交）**：prepare 和 commit 之间加 pre-commit 阶段，参与者不用在 coordinator 失败时无限等；pre-commit 里没报告失败，就可以提交。原则：多一阶段换 crash 下的非阻塞行为。

2PC 阻塞是主要关切、且不假设分区时用。分区下别信它 —— 它会分裂并违反原子性。例：pre-commit 后 coordinator 死 → 存活者仍可提交。失败模式：分区让一边 commit、一边 abort。Fix：分区可能时用 consensus-based commit。

**Quorum-Based Commit（基于 quorum 的提交）**：quorum 参与者同意即 commit，不用全员；overlap 的 quorum 保证后来者能找到决策。原则：去中心化 agreement，容忍少数失败，无单 coordinator 瓶颈。

要比 2PC 更高可用的 atomic commit 时用。别假设它免费 —— quorum sizing（quorum 规模）和恢复仍重要。例：5 副本，3 个接受即 commit。失败模式：stale quorum（过期 quorum）错过 commit 决策。Fix：quorum intersection（R + W > N）+ 恢复时传播决策。

## 4. 组合 + Sagas（Module 7）

**组合**：isolation 机制（2PL、OCC、MVCC / SSI）和 atomicity 机制（2PC、3PC、quorum commit）组合出完整 ACID-like 分布式事务。原则：隔离和原子性独立选，再检查它们交互正确（如锁持有到 commit 决策）。用组合表按 workload 选。别假设任何一个机制单独给 ACID。例：低冲突跨 shard 写用 OCC + 2PC。失败模式：commit 决策持久化前放锁，漏未提交数据。Fix：锁 / 版本持有到决策 durable。

**Saga**：事务跨秒/分钟或外部服务时，跨它持有锁 / 2PC 会 block 所有人。Saga 拆成 local step（本地步骤），每步配 compensating action（补偿动作）；失败时按反序跑补偿。原则：用 eventual consistency（最终一致性）+ 显式 undo 换 atomic isolation。

多服务业务 workflow（订机票 → 酒店 → 车）用。中间态绝不能可见时别用。例：酒店订了，机票失败 → 补偿取消酒店。失败模式：补偿本身失败，或不是真正逆操作（如不可退预订）。Fix：幂等补偿 + 重试 + 设计步骤时就让它可补偿。

## 自测 4 问（带答案要点）
1. 2PL / OCC / MVCC / SSI 各适合什么 workload？各防不了什么？
   → 2PL 冲突频繁；OCC 冲突少读多；MVCC 读多要稳定读（防不了 write skew）；SSI 要 serializable + snapshot 性能（有误 abort）。
2. 2PC 为什么会 block？3PC 修了什么、没修什么？
   → prepare 后 coordinator crash，参与者等决策；3PC 加 pre-commit 修 crash 下阻塞，没修分区下原子性。
3. Quorum commit 相对 2PC 赢了什么？代价是什么？
   → 去中心化、容少数失败、无单 coordinator 瓶颈；quorum sizing + 恢复仍重要。
4. Saga 什么时候用？它 trade 掉了什么？
   → 长事务 / 外部服务，2PC 阻塞不可接受时；trade atomic isolation 换 eventual consistency + 显式补偿。
