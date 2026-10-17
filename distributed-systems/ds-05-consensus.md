# Consensus（Module 8）

> 这是全课最密的模块。读法：先接受 FLP（异步下 safety + termination 不可兼得），再看 Paxos 怎么在 safety 上死磕，最后看 Raft 怎么把同样的保证讲得更易懂。结论先行：**不要自己手写 consensus，用成熟实现**。

## 1. Consensus 问题：Agreement + Validity + Termination

**Consensus（共识）**：所有 correct 节点对一个值达成一致（agreement），该值必须是被提议过的（validity），所有 correct 节点最终决定（termination），哪怕有 crash / 延迟。

它是 leader election（选主）、replicated log（复制日志）、atomic commit 背后的原语。副本要保持同步时用。别用 timeout 手搓 agreement。例：三个副本对下一条 log entry 达成一致。失败模式：两个节点决定不同值。Fix：用保证 agreement（safety）的协议，哪怕它会 stall（停顿，liveness）。

## 2. FLP Impossibility：异步下别想全都要

**FLP impossibility（FLP 不可能性）**：完全异步系统 + 哪怕一个 crash 失败，任何确定性协议都不能同时保证 consensus 的 safety 和 termination。直觉：adversary（对手）可以一直延迟消息，让系统永远停在 bivalent（未决）态，因为慢节点和死节点不可区分。

原则：必须牺牲确定性、纯异步、或保证终止 —— 真实系统用 timeout / 随机化 / 部分同步。FLP 是给 timeout 和选主找理论依据。别试图证明一个异步确定性共识协议正确。例：一个慢节点让投票一直平局。失败模式：永远等死节点。Fix：timeout + 随机 backoff / 部分同步假设。

## 3. Paxos：多数交集 + 采纳已接受的最大值

角色：proposer（提议者）、acceptor（投票者）、learner（学习者）。Proposal number（提案号）全局唯一递增。

- Phase 1：proposer 发 prepare(n)；acceptor 承诺不再接受更低号提案，并返回已接受的最大值。
- Phase 2：proposer 发 accept(n, v) —— v 是返回的值（如果有），否则是自己的值；acceptor 在没承诺更高号时接受。
- 一个值在多数接受时被 chosen（选定）。

原则：多数交集保证至多一个值被选定；采纳已接受的最大值保住它。单值共识 / 复制日志用 Paxos。别用 naive 单提议者而不处理竞争提议者。例：两个 proposer 竞争；高号胜，低号必须用被采纳的值重试。失败模式：duelling proposer（对决提议者）livelock（活锁）。Fix：稳定 leader（Multi-Paxos）。

**Intricacies（细节）**：提案号必须全局唯一递增；acceptor 必须跨 crash 持久化 promise / 已接受最大值；learner 只在多数接受后学习；看到已接受值的 proposer 必须采纳它。原则：durability（持久性）+ 多数交集 = 跨 crash / 重启的 safety。实现 / review Paxos 时用这些检查。别跳过持久状态或提案号纪律。例：acceptor 重启忘了 promise → 可能接受过期提案、破坏 agreement。失败模式：crash 后 promise 丢失。Fix：ack 前先 durable log。

**Paxos in real life**：生产通常是 Multi-Paxos：一个稳定 leader 把 Phase 1 跑一次，然后用 Phase 2 驱动很多 instance，加 batching（批处理）、pipelining（流水线）、leader lease（leader 租约，做线性化读）、snapshot（快照截断日志）。原则：把昂贵的选主阶段摊薄。

复制日志用 Multi-Paxos（或 Raft），不是每条命令跑一次单 shot Paxos。别每写都重跑完整 Paxos。例：leader 选一次，然后每条 log entry 一次往返追加几百条。失败模式：leader 频繁更替导致反复 Phase 1。Fix：稳定选主 + 随机 timeout + lease。

## 4. Replicated State Machine：对日志顺序达成共识

**Replicated state machine / RSM（复制状态机）**：反复跑共识，每条 log slot 一次，对命令的有序序列达成一致；每个副本按同样顺序把同样命令应用到确定性状态机，状态保持相同。原则：对日志顺序达成共识，不直接对状态达成共识。

复制服务（KV store、协调服务）用 RSM。别让副本按不同顺序应用命令。例：三个副本对 [put A=1, put B=2] 达成一致，都按该顺序应用。失败模式：gap（缺口）或重排命令让状态分叉。Fix：log index、snapshot、catch-up（追赶）。

**Distributed transactions via consensus（经共识的分布式事务）**：不用 2PC 的阻塞 coordinator，把 commit / abort 决策跑共识：参与者提议投票，共识决定 commit 或 abort，所有人学到同样结果。原则：共识去掉单阻塞 coordinator —— coordinator 挂了，另一个节点能把同一共识推完。

Atomic commit 必须 survive coordinator 失败时用。别假设它更便宜 —— 共识要更多消息。例：shard 经 Paxos 对是否提交转账投票；所有人学到同样决策。失败模式：2PC 里 coordinator 中途死（block）vs 共识（另一节点完成它）。Fix：把决策路由经共识。

## 5. Raft：同样的保证，更好懂的拆法

**Raft**：为可理解性设计的共识算法，把问题拆成 leader election（选主）、log replication（日志复制）、safety。任何时刻每个节点是 leader、follower、candidate 之一；leader 服务所有 client 写，follower 复制。

原则：强 leader + term（任期）选主简化推理。想要 Paxos 等价保证、但更易实现/运维时用 Raft。Byzantine 场景别用。例：一个 leader 追加 entry，follower 复制。失败模式：同一 term 两个 leader。Fix：term 号 + 每 term 只投一次。

**两个 RPC**：RequestVote（candidate 为某 term 拉票；投票者只在 candidate 日志至少一样新时给票）、AppendEntries（leader 发 heartbeat + log entry；follower 在 prev entry 匹配时追加，否则拒绝）。原则：log-matching（日志匹配）+ up-to-date-voter（投票者看日志新旧）规则保 safety。整个通信面就是这两个 RPC，别加旁路。例：日志旧的 stale candidate 输掉选举。失败模式：follower 接受和自己日志冲突的 entry。Fix：查 prevLogIndex / Term，不匹配就截断。

**实现**：选主：follower timeout（随机化）→ candidate 递增 term、投自己、拉票；多数票胜 → leader。日志复制：leader 追加 client 命令，经 AppendEntries 复制，多数持有后 commit，再应用。Safety：Leader Completeness（已提交 entry 在所有未来 leader 日志里）+ State Machine Safety（副本按同样顺序应用同样 entry）。Membership（成员变更）：joint consensus（联合共识）。用随机 timeout 避免 split vote（选票分裂）。别用固定 timeout。例：5 节点，leader 拿到 3 个 ack 后 commit。失败模式：候选人同时参选 split vote。Fix：随机 election timeout + 重试。

## 6. Standing on the Shoulders of Giants：用成熟实现

共识很微妙，正确实现是很大的工程。原则：优先成熟、久经考验的实现（Raft / Paxos 库、etcd / ZooKeeper / Spanner 式服务），别自己从头写。

协调、锁、复制状态用现有系统。别为单个功能重实现共识。例：用 etcd 做选主，而不是写 Paxos。失败模式：手写协议测试通过，却在罕见 crash / 分区交错下挂。Fix：复用 proven 代码 + fault injection（故障注入）测试。

## 自测 4 问（带答案要点）
1. Consensus 三个条件？FLP 说什么、逼我们用什么？
   → agreement + validity + termination；异步 + 一个 crash 下确定性协议不能同时保证 safety + termination；用 timeout / 随机化 / 部分同步。
2. Paxos 两阶段各干什么？多数交集保证什么？
   → Phase 1 prepare + 承诺 + 返回最大值；Phase 2 accept（采纳返回的值）；多数交集保证至多一个值被选定。
3. RSM 是对什么达成共识？为什么不直接对状态达成共识？
   → 对日志顺序；副本按同样顺序应用同样命令，状态自然相同。
4. Raft 相对 Paxos 赢了什么？两个 RPC 各是什么？
   → 可理解性：强 leader + term 选主 + 拆成选主/复制/安全；RequestVote + AppendEntries。
