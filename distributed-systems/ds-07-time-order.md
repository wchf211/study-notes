# Time + Order（Modules 9–10）

> 这两章回答一个问题：“没有全局时钟时，怎么定先后？”答案分两层：物理时间管“真实世界什么时候”，逻辑时间管“谁导致了谁”。混了这两层，就会用 wall 时间戳判因果，或用逻辑时钟卡真实 deadline —— 都是错。

## 1. Time：为什么“同时”在分布式里没有定义（Module 9）

**核心困难**：单机一个时钟给所有事件排序；多机每台有时钟，时钟漂移、消息耗时未知，“哪个先发生”没有唯一显见答案。原则：排序需要显式机制，不是 wall time（墙上时间）。用这个 framing（框架）再去碰时间戳。别假设两节点的 timestamp 可比。例：两个学生约 2 点见面，一人手表快 —— 他们对“现在”有分歧。失败模式：按不同 host 的本地 timestamp 给事件排序。Fix：逻辑时钟，或带已知误差界的同步时间服务。

**Practical perspective（实践视角）**：真实时钟漂移（石英、温度）、速率略不同，靠 NTP（网络时间协议）这类协议在未知延迟的网络上估 offset（偏移）来纠正。缓解：量 interval（间隔）用 monotonic clock（单调时钟）；定期同步；留 skew（偏差）余量（如 Spanner 的 TrueTime 暴露 uncertainty interval，不确定区间）。

Wall time 只用于人类-facing deadline（截止）和粗略排序。正确性关键的排序别用它。例：token “10:00 过期”在慢时钟上可能还有效。失败模式：假设两服务器对过期达成一致。Fix：时钟误差界 + lease 留安全 margin + 精确处用逻辑排序。

**Theoretical perspective（理论视角）**：哪怕时钟完美，simultaneity（同时性）也定不下来：相对论说两个远处事件是否同时取决于观察者 frame（参考系）；分布式里消息不能比它的延迟更快传信息。原则：“同时”是观察者依赖的；well-defined（定义良好的）是 causality（因果：一个事件是否可能影响另一个）。用因果，不用同时，当基础。别试图给正确性定义全局“现在”。例：两个节点“同时”行动，但彼此都不知道对方。失败模式：设计需要精确同时的协议。Fix：用 happened-before（先发生）推理。

**Logical clock（逻辑时钟）**：只跟踪相对事件顺序的计数器，不跟踪物理时间：本地事件 tick（加一），消息里带值，接收方 advance（推进）到超过它。原则：逻辑时间抓住因果相关事件的“before”，不用同步时钟。排序重要时用它。别用 bare counter（裸计数器）证明两事件 concurrent（并发）。例：消息盖戳 5，节点在 3 收到 → 接收方跳到 5 之后。失败模式：把相等逻辑 timestamp 当同时。Fix：要排序用 Lamport，要因果用 vector clock。

## 2. Order：从词汇到工具箱（Module 10）

**Total vs partial order（全序 vs 偏序）**：全序每对事件可比（一个先）；偏序有些对可比、有些不可比（concurrent）。原则：分布式执行天然是偏序 —— 只有因果相关事件有确定顺序。单序列 agreement（一致）要求的地方（复制日志）才用全序。别到处强加全序 —— 要 coordination 成本。例：一条消息链沿链有全序，两条独立链不可比。失败模式：给独立工作强加全序，把它串行化。Fix：只在因果或 agreement 要求处排序。

**Causality / happened-before（因果 / 先发生，→）**：a→b 如果 a、b 在同一节点且 a 先来，或 a 是 send、b 是匹配的 receive，或有传递链。既非 a→b 也非 b→a 时，两者 concurrent（||）。原则：因果是影响正确性的偏序。用它判断排序是否会影响结果。别把因果和时间混淆。例：回复因果上在请求之后；两个无关请求 concurrent。失败模式：假设并发操作有真实顺序。Fix：两个顺序都处理，或经共识强加一个。

**Lamport clock（Lamport 时钟）**：每节点一个计数器：本地 / 发送事件前 +1；收到时设为 max（本地，已收）+ 1。保证 clock condition（时钟条件）：a→b 则 C(a) < C(b)。原则：便宜、单个数字，给一致（但非全因果）的排序；tie-break（平局打破）用 node ID 得全序。

因果相关事件排序 + 简单全序时用。别用它推断因果 —— C(a) < C(b) 不蕴含 a→b。例：两个并发事件可以拿到 timestamp 2 和 5，没有因果链。失败模式：从更早 timestamp 推出 cause（原因）。Fix：要因果检测用 vector clock。

**Vector clock（向量时钟）**：每节点一个向量 [c0..cN]，每个事件自己 entry +1；收到时和 sender 向量逐项取 max。比较：V(a) < V(b) 当且仅当每项 ≤ 且至少一项 <。满足 strong clock condition（强时钟条件）：V(a) < V(b) 当且仅当 a→b，不可比的向量 = concurrent。原则：精确因果检测，代价是每 timestamp O(N) 大小。

冲突检测、causal delivery（因果投递）、小/中系统版本跟踪时用。参与者集合巨大动态时别用 —— 向量随 N 长。例：[1,0,0] vs [0,1,0] 不可比 → concurrent。失败模式：漏掉 concurrency 来源（如 client）不给 entry，冲突看起来像因果。Fix：每个 concurrency 来源给 entry，或用可扩展变体。

**Version vector（版本向量）**：把 vector-clock 思想用到数据项：每个 replica 给该 item 记一个向量，数它从每个 replica 见过多少更新。比向量知哪个版本 dominate（支配），还是冲突。原则：per-item 因果，不是 per-event。Multi-master 写的多副本 store（Dynamo 式）用。别假设版本向量给所有操作全局排序。例：replica A [2,0]，replica B [1,1] → 谁都不支配 → 冲突待解决。失败模式：按 wall time “latest wins（最新胜）”处理冲突。Fix：显式 merge / 应用层解决。

**Dotted version vector（点版本向量）**：可扩展改进：不给每版本一个完整向量，记 compact base vector（紧凑基向量）+ 一个 dot（点，节点 + 序列号）标识新事件。原则：用 frontier（前沿）+ 一个新事件表示因果历史，很多情况下避免每版本 O(N)。

版本向量随很多 replica / client 变太大时用。不懂编码别用 —— sync / merge 时 dot / base bookkeeping（记账）要管好。例：同步一次新写只需它的 dot + 共享 base，不用完整向量。失败模式：merge 时丢 dot / base 记账 → 假冲突或漏因果。Fix：严格按协议的 merge 规则。

**Distributed snapshot problem（分布式快照问题）**：快照要抓住每节点状态 + 在途消息，且 consistent（一致）：快照包含一个 receive，就必须包含匹配的 send（no orphan receive，无孤儿接收），即对应一个 consistent cut（一致割）。原则：快照用于 checkpointing（检查点）、debug、死锁检测、恢复。

要 coherent global view（连贯全局视图）又不停全世界时用。别按 wall-clock 时间各节点各拍快照 —— 可能不一致。例：B 的快照显示钱已收到，A 的显示从没发过。失败模式：inconsistent cut（不一致割）让恢复重放钱。Fix：用记录 channel state（通道状态）的快照算法。

**Chandy-Lamport（Chandy-Lamport 算法）**：initiator（发起者）记录自己状态，给每个 outgoing channel（出通道）发 marker（标记）；节点第一次收到 marker 时记录自己状态、标记 channel 为 recording（记录中）、转发 marker；某 channel 上收到 marker 时，记录该 channel 上自开始以来收到的消息。假设 FIFO channel、无失败。原则：marker 分隔 pre-snapshot 和 post-snapshot 消息。

FIFO、无失败场景下一致快照用。非 FIFO channel 或 crash 下别直接用，要扩展。例：marker 流冻结一张跨所有节点的割 + 在途消息。失败模式：非 FIFO channel 在 marker 之前投递 post-snapshot 消息 → channel state 坏。Fix：FIFO 强制或用变体算法。

**Physical vs logical time：收尾对照**

- Physical time（物理时间）：连续、由物理现象度量，用于真实世界 deadline，但跨节点永不同步完美。
- Logical time（逻辑时间）：离散、由事件和因果定义，给排序，不用 wall 时钟。

原则：按问题选时间概念：“真实世界什么时候？”→ 物理（带误差界）；“谁导致谁 / 什么顺序？”→ 逻辑。两者都用：物理管 timeout / lease，逻辑管因果/排序。别互相替代。例：lease 用带 margin 的物理时间；冲突检查用 vector clock。失败模式：用 wall timestamp 判因果，或用逻辑时钟卡真实 deadline。Fix：选对时钟，说清它的误差/排序保证。

## 这门课怎么教的（Modules 9–10）

Module 9 是短 framing 模块：先给人类 scheduling puzzle（两个学生，2 点/3 点，一块表快）说明 wall-clock agreement 并不 trivial，再分 practical（真实时钟漂移，NTP 帮但不完美）vs theoretical（相对论：同时性取决于观察者；消息延迟让“同时”无定义），最后引出 logical clock 当 wall time 的替代。视觉是 timeline + 时钟面；基调是概念，不是公式堆。

Module 10 按工具箱顺序建：排序词汇（全序 vs 偏序）→ 因果定义（happened-before）→ Lamport（便宜、弱）→ vector（精确、贵）→ version vector（per-item 变体）→ dotted version vector（可扩展变体）→ snapshot（全局一致割）→ Chandy-Lamport 解法 → 物理-逻辑合成。每课时钟都复用同一张 3 节点执行图，读者可以跨方案比较 timestamp。最密的是时钟条件和向量大小下界。值得偷的：**同一张执行图反复用** —— 换一个时钟方案，不换图，观众直接比 timestamp。

## 自测 4 问（带答案要点）
1. 为什么 wall 时间戳不能判因果？正确做法？
   → wall 时钟漂移 + 消息延迟未知；因果用逻辑时钟，精确处用带误差界的同步时间。
2. Lamport 和 vector clock 各保证什么？为什么 Lamport 不能推因果？
   → Lamport：a→b 则 C(a)<C(b)，反向不成立；vector：V(a)<V(b) 当且仅当 a→b；Lamport 丢了因果信息。
3. Version vector vs dotted version vector？
   → 前者每 item 记完整向量，判 dominate / 冲突；后者 base + 一个 dot，更可扩展，但 merge 记账要严。
4. 一致快照的要求？Chandy-Lamport 假设什么？
   → 含 receive 必含匹配 send（无孤儿接收）；假设 FIFO + 无失败；marker 分隔 pre / post-snapshot。
