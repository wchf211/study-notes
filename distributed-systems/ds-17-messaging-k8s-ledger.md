# Messaging + Cluster Management + Ledger（Modules 18–20）

> 这三章是三个“系统级答案”：Kafka（消息解耦）、Kubernetes（集群管理）、Corda（账本）。共同读法：每个系统先看它解决什么问题 → 核心模型 → 调参/保证 → 坑。Module 18 最密：投递语义是 producer + broker + consumer 三方合谋，不是单个 knob（旋钮）给的。

## 1. Kafka：Log 是中心（Module 18.1）

**Kafka**：分布式、持久的 log（日志），做 messaging（消息）：producer（生产者）append（追加）record（记录）到 topic（主题），topic 拆成 partition（分区，每个是有 offset（偏移）的有序、不可变 log），broker（代理）存 partition + replication（复制），consumer（消费者）从 offset 读并跟踪位置。

一种系统里两种消费样式：**queue（队列）**（consumer group（消费组）共享 topic，每条记录给一个成员）+ **publish-subscribe（发布-订阅）**（每个 group 拿到全部记录）。原则：log 是中心 —— retention（保留）、replay（重放）、ordering（排序）都从它来。

用 Kafka 解耦 producer / consumer 和缓冲 burst（突发）。别把它当数据库或每消息 ack 语义的 queue。例：click 事件追加到 topic；analytics 和 billing 两个 group 各自独立消费。失败模式：假设 consumer 读后消息被删。Fix：理解 retention + offset —— consumer 自己控制位置。

## 2. Kafka Levers：每个 Knob 都在 Trade（Module 18.2）

Tuning knob（调参旋钮）：**acks**（0 = fire-and-forget 只管发；1 = leader ack；all = ISR ack —— durability vs latency）、**min.insync.replicas**（必须 ack 的 replica 数）、**replication factor**、**retention**（时间/大小）、**partition count**（并行度 vs 排序/故障转移代价）、**producer retry + idempotence（幂等）**、**consumer auto-commit vs manual offset commit**、**fetch / batch size**。

原则：每个 lever 都 trade 一样东西 —— durability、latency、throughput、ordering、或 availability。Durable 写用 acks=all + min ISR ≥ 2；要速度用更少。别设 acks=all + min ISR=1 还指望 durability。例：payment topic 用 acks=all、RF=3、min ISR=2。失败模式：acks=1，leader 在复制前死了就丢数据。Fix：acks=all + 足够 ISR。

## 3. Kafka Messaging Guarantees：投递语义是三方合谋（Module 18.3）

**Ordering（排序）**：一个 partition 内保证，跨 partition 不保证。**Delivery（投递）**：默认 at-least-once（至少一次，retry 可能重复）；at-most-once（至多一次，consumer 处理前 commit offset，可能丢）；effectively-once（“恰好一次”效果，经 idempotent producer + transactional producer / consumer + 处理完再 commit offset）—— 即 at-least-once 投递 + 幂等处理 + 原子 offset commit。

原则：投递语义是 producer、broker、consumer 的 joint property（联合属性）—— 没有单个 knob 能给。常用安全 pattern：幂等 consumer + 处理后手动 commit offset。Loss 不可接受时别在处理前 commit offset。例：consumer 处理 payment，再 commit offset；commit 前 crash 会重放，所以 handler（处理函数）必须幂等。失败模式：crash / retry 后重复处理。Fix：幂等 handler / dedup key（去重键）。

## 4. Kafka 事务、存储布局、其它保证（Module 18.4）

**Storage（存储）**：每个 partition 是 append-only log，拆成 segment file（段文件）在 disk 上（.log、.index、.timeindex），sequential 写 + OS page cache + zero-copy 读 = 高 throughput。**Transaction（事务）**：transactional producer（transactional.id、initTransactions）能原子写多 partition 并 commit consumer offset（read-process-write），有 transaction coordinator + marker；isolation.level=read_committed 的 consumer 只看到已提交事务（aborted marker 被过滤）。其它保证：log compaction（保留每 key 最新值）、ISR 复制、每 partition leader election（选主）。

原则：sequential disk + batching + zero-copy = throughput；事务 = 原子多 partition 写 + offset commit。Read-process-write pipeline 必须不 double-process（重复处理）时用事务。别每写都用事务 —— 要 latency 代价。例：从 topic A 消费、写 topic B、commit offset —— 全原子。失败模式：非事务 consumer 读到未提交/已 abort 的数据。Fix：read_committed + transactional producer。

## 5. Kubernetes：声明 Desired State，让控制循环收敛（Module 19.1）

**Kubernetes** 编排容器：把工作调度到节点、保持 desired replica 数运行、重启失败容器、暴露服务、管理 config / storage。对象：Pod（一个或多个容器 + 共享网络/存储）、Deployment / ReplicaSet（desired replica 数）、Service（稳定 endpoint + 负载均衡）、Namespace、ConfigMap / Secret。Desired state（期望状态）被声明（YAML）、存在 etcd、controller（控制器）reconcile（调和）actual（实际）向 desired 靠。

原则：声明期望状态；让 control loop（控制循环）收敛。容器编排 at scale 用 Kubernetes。别把它当简单 process launcher（进程启动器）。例：Deployment 声明 3 个 web pod 副本 —— Kubernetes 随节点失败创建和替换它们。失败模式：假设 Pod 是永久的。Fix：为 Pod churn（更替）设计（stateless + 外部状态）。

## 6. Manager + Worker 组件：控制平面决策，Worker 执行（Module 19.2）

**Manager（control plane，控制平面）**：API server（前端，校验并把对象存 etcd）、Scheduler（把未调度 Pod 按约束/亲和/locality 分到节点）、Controller Manager（跑 controller：Node、Replication、Endpoints 等 —— 每个是 control loop，经 API server 看状态、把 current 推向 desired）。**Worker**：kubelet（每节点 agent，保证分配给它的 Pod spec 运行且健康）、kube-proxy（网络代理，维护规则让 Pod 在集群内外可达）、container runtime（容器运行时）。

**Concurrency（并发）**：Kubernetes 是 eventually consistent（最终一致）；每个对象在 etcd 里有 resourceVersion（资源版本），用于 compare-and-swap（CAS）更新，防止 lost update（丢失更新）。原则：控制平面决策，worker 执行，etcd 是 source of truth（事实来源）。并发更新用 CAS / resourceVersion。别 blind overwrite（盲目覆盖）对象。例：两个 controller 抢着更新 Deployment —— CAS 让一个 retry。失败模式：stale read-modify-write 丢失更新。Fix：resourceVersion CAS + retry。

## 7. Corda：Need-to-Know 账本，不是公共链（Module 20.1）

**Corda**：给受监管业务网络（银行、贸易、保险）的分布式账本，各方需要共享、可信记录，但不需要公开广播。和公共区块链的区别：permissioned identity（许可身份：已知法律实体，不是匿名矿工）、need-to-know 数据共享（只有交易各方看到它，不是全网）、没有全局广播/挖矿、deterministic finality（确定性终局，经 notary，不是概率确认）。

原则：账本是相关各方的共享事实来源 —— 不必然是所有人的。Multi-party 业务 workflow + 隐私 + 法律身份时用 Corda。数据必须私有时别用公共链。例：两家银行记录贷款转移；其它银行看不到。失败模式：把私有交易条款放公共账本。Fix：need-to-know 共享 + 私有点对点 flow。

## 8. Corda 数据模型：Validity + Uniqueness = 正确账本（Module 20.2）

核心对象：**State（状态）**（不可变事实，如一笔 cash 或 IOU，带 owner + contract 引用）、**Contract（合约）**（code / rule，校验消费/产生 state 的 transaction 是否有效）、**Transaction（交易）**（消费 input state、产生 output state；state 是 UTXO 式 —— 花掉的 state 被消费，新 state 被创建）、**Flow（流）**（多方协议，构建、签名、finalise 交易）、**Notary（公证人）**（服务，防止 double-spend，检查 input 是否已被花掉）。

原则：validity（合约规则）+ uniqueness（notary）= 正确账本。用 state + contract 把业务规则编码上链。别把 state 当可变 —— 消费它、重建它。例：付 £10，消费一个 £10 state，创建两个输出（£10 给收款方，找零回来）。失败模式：同一 state 在两个交易里 double-spend。Fix：notary uniqueness 检查。

## 9. Corda 架构：Consensus 在需要处，Privacy 在别处（Module 20.3）

Corda 节点包含：vault（本地相关 state 存储）、transaction storage、contract code、flow、network service（identity、network map、notary）。节点点对点通信（flow），不广播；每个节点只存它参与过的交易（外加 notary attestation）。Network map service（网络地图服务）发布节点身份/地址；identity service（doorman，门卫）接纳许可节点。Consensus 是 per-transaction（每交易）：counterparty（对手方）+ notary 同意，不是全网。

原则：consensus 在需要处，privacy 在别处。Multi-party agreement 用 Corda 的 flow 框架。别指望每个节点上有全局账本副本。例：Alice 和 Bob 跑一个 flow，notary 签字，双方都存结果。失败模式：节点缺它需要的交易（它不是当事方）。Fix：当事方需要时经 flow 显式分享交易。

## 10. Corda 向后兼容：去中心化网络里不能强制同时升级（Module 20.4）

去中心化网络里，节点跑不同 Corda / CorDapp 版本，升级不能同时。Corda 给 API & ABI 向后兼容（公开 CorDapp API：旧 CorDapp 在新 Corda 上继续工作）、wire compatibility（线兼容：不同版本节点仍能通信）、data / schema compatibility（vault schema 演进不破坏旧 state）。

原则：compatibility（兼容性）是账本的一等设计约束 —— 不能强制所有节点同时升级。用版本化 API 和 wire 格式；测 mixed-version network（混合版本网络）。别不带 migration path（迁移路径）就破坏 wire 格式或公开 API。例：跑旧 CorDapp 的旧节点仍和新节点交易。失败模式：破坏性 API 变更 stranded（搁浅）不能立即升级的节点。Fix：deprecation window（弃用窗口）、versioned flow、schema migration。

## 这门课怎么教的（Modules 18–20）

Module 18（Kafka）先给解耦问题（producer 不该等 consumer），再讲 Kafka 的 log 模型（topic → partition → 不可变 segment，offset 是位置），再把 tuning knob 当“lever”（杠杆）讲 —— 每个 lever trade durability / latency / ordering —— 再从 lever 推 guarantee，再加事务和存储细节。用 producer → broker → consumer 图和 offset timeline。单个 lever 映射到一个 guarantee 时最易懂；难处是投递语义（at-least / at-most / effectively-once）依赖 producer + broker + consumer 三方。

Module 19（Kubernetes）短、架构优先。先给编排问题（放工作、保持运行、暴露它），再走控制平面 vs worker 拆分，reconciliation loop（调和循环）是组织思想：desired state 在 etcd，controller 把 actual 推向 desired。用标准 manager / worker 图；独特动作是把 Kubernetes 框成 eventually-consistent、CAS-guarded state machine（状态机）。

Module 20（Corda）先给“区块链不适合受监管金融”的理由（所有人看所有东西、无法律身份、概率终局），再讲 Corda 当“need-to-know”账本：state、contract、flow、notary。走数据模型 → 节点架构 → 向后兼容。用 IOU / 现金转移例子和 transaction-chain 图。和 Bitcoin / Ethereum 对比时最易懂；难处是 notary + contract validation 交互。

值得偷的：**先给问题，再给模型，再给 lever / 保证**。讲系统课时，不要先讲 API；先讲它解决什么问题。**Lever 当 trade-off 教**：Kafka 的 knob 不是“配置清单”，是“每个 lever trade 一样东西”—— 观众记住 trade-off，就不用背配置。**对比当锚**：Corda 对 Bitcoin / Ethereum 的对比是理解 need-to-know 的最快路径。

## 自测 4 问（带答案要点）
1. Kafka 里 ordering 和 delivery 各是什么保证？Effectively-once 怎么来的？
   → Ordering：partition 内；Delivery：默认 at-least-once；effectively-once = at-least-once + 幂等处理 + 原子 offset commit。
2. Kafka 的 lever 各 trade 什么？举一例。
   → 每个 lever trade durability / latency / throughput / ordering / availability；如 acks=all + min ISR=1 不给 durability。
3. Kubernetes 的 reconciliation loop 是什么？CAS 干什么？
   → Desired state 在 etcd，controller 把 actual 推向 desired；resourceVersion CAS 防 lost update。
4. Corda 和公共链的三个区别？Validity + uniqueness 各是谁？
   → 许可身份、need-to-know、无全局广播/挖矿 + 确定性终局；validity = contract，uniqueness = notary。
