# Foundations（Modules 1–2）

> 一句话：分布式的难，不是“机器多了”，而是“部分失败 + 没有全局真相”。这章把判断标准、失败分类、几个反直觉结论先立起来，后面所有协议都是在这套地基上长出来的。

## 1. 为什么分布

**Distributed system（分布式系统）**：多台独立机器协作，让用户看到一个连贯的系统。三个动机：scale（并行 + 更多内存/磁盘）、availability（单节点挂了还能活）、lower latency（工作放离用户近的地方）。

反向约束：负载不大就别分布 —— 协调成本会吃掉收益。例：单机小店 → 拆 front-end、DB、cache 到不同 host。失败模式：以为分布 = 免费加速。Fix：量协调 overhead。

## 2. 八大谬误：设计检查表

**Fallacies of distributed computing（分布式计算谬误）**：八个错误假设 —— 网络可靠、延迟为零、带宽无限、网络安全、拓扑固定、一个管理员、传输成本为零、网络同构。

用法：当设计检查表。别把远程调用当本地函数调用写。例：假设零延迟 → 写出 chatty RPC（来回很多次的小调用），一上真实 RTT 就垮。失败模式：把 timeout 当成 failure。Fix：retry + idempotency（幂等）+ 显式 timeout。

## 3. 核心困难：部分失败

**Partial failure（部分失败）**：一部分节点 / 消息失败，其余继续跑，全局状态变得 ambiguous（模棱两可）。其它困难：concurrency（并发）、no shared clock / memory（无共享时钟/内存）、network unpredictability（网络不可预测）、heterogeneous nodes（异构节点）。

每个跨节点步骤都要用部分失败的视角想。例：付款发了，ack 丢了 —— 到底付了没？失败模式：盲目重试 double-charge（重复扣款）。Fix：幂等操作 + 对账（reconciliation）。

## 4. 正确性：Safety + Liveness

**Safety（安全性）**：坏事永不发生（如 no double-spend、no two leaders）。**Liveness（活性）**：好事最终发生（如请求最终完成）。Safety 是 invariant（不变量），liveness 是 progress（进展）。

定协议时两个都要写出来，别悄悄 trade。例：永不启动的烤箱是 safe 但 useless；烧糊食物的烤箱是 safety 失败。失败模式：只优化 availability，破了 invariant。Fix：先写 safety，再写 liveness bound。

## 5. System Models：假设越强，算法越容易，现实越远

**Synchronous（同步）**：延迟、处理、时钟都有界。**Partially synchronous（部分同步）**：界最终成立 / 未知。**Asynchronous（异步）**：无界。

强假设让证明 / 算法更容易，但更不现实。同步推理只在界真的被工程保证时用。异步系统里，别指望靠 timeout 可靠地检测失败。例：已知 timeout 下选 leader 容易，没有时很难。失败模式：把同步算法部署到公网。Fix：选部分同步 / 异步容忍的设计。

## 6. 失败分类：先选模型，再选算法

严重程度递增：fail-stop（干净停机）、crash（停机，可能丢易失态）、omission（丢消息/回复）、timing（迟到）、response（错回复）、arbitrary / Byzantine（任意行为，含恶意）。

受控集群用 crash 模型；节点可能敌对时才用 Byzantine。别假设野外是 fail-stop。例：静默磁盘损坏看起来像 Byzantine，不是 crash。失败模式：只为干净停机设计。Fix：checksum、timeout、冗余，该 Byzantine 时上 Byzantine 容忍协议。

## 7. Exactly-Once：Delivery ≠ Effect

**At-least-once（至少一次）** + **at-most-once（至多一次）** 理论上 = exactly-once，但实践中 sender 和 receiver 都可能 crash 时，exactly-once delivery 不可能：sender 分不清“送到了但 ack 丢了”还是“根本没送到”。

关键区分：delivery（送达）vs processing / effect（处理/效果）。做法：at-least-once + idempotent receiver（幂等接收方），或 loss 可接受时用 at-most-once。别在文档/宣传里承诺 exact delivery。例：付款 ack 丢了 → 重试不能再扣一次。失败模式：重试产生重复副作用。Fix：dedup key（去重键）+ 幂等 handler。

## 8. 失败检测本身不可靠

沉默节点可能是慢、分区、或死了。**Suspicion（怀疑）** 是概率性的：heartbeat + timeout 要按 workload 调，不是 certainty（确定性）。用怀疑 + 最终移除 / fencing（隔离）保 liveness，但要防 false positive（误判）。

别让检测器单方面宣布永久死亡。例：GC 停顿在 peers 看来像 crash。失败模式：把活节点踢出去，分裂集群。Fix：quorum-based membership（基于 quorum 的成员管理）+ fencing token + 优雅重加入。

## 9. Stateless vs Stateful：状态放哪，决定扩容难度

**Stateless（无状态）**：请求之间不留 per-client 持久状态，任意 replica 都能服务任意请求。**Stateful（有状态）**：留 session / 数据状态，路由、迁移、恢复都要操心。

原则：想轻松 scale-out，就把状态推到 durable store（持久存储）。Stateless front-end + stateful store。别把内存里藏状态的服务假装成无状态。例：给 web 层加登录历史 → 要么 sticky routing（粘性路由），要么共享 store。失败模式：重启丢内存 session。Fix：状态外置或复制。

## 自测 4 问（带答案要点）
1. 分布式的核心困难是什么？为什么 ack 丢了不能盲目重试？
   → 部分失败，全局状态 ambiguous；重试可能 double-charge；用幂等 + 对账。
2. Safety vs liveness 各是什么？为什么先写 safety？
   → Safety = 坏事永不发生；liveness = 好事最终发生；先定 invariant，再定进展 bound。
3. 为什么 exactly-once delivery 在实践中不可能？正确做法？
   → sender 分不清送到但 ack 丢 vs 没送到；at-least-once + 幂等接收方，或 loss 可接受时 at-most-once。
4. Stateless vs stateful 对扩容的影响？
   → Stateless 任意 replica 服务任意请求；stateful 要操心路由/迁移/恢复；状态推 durable store。
