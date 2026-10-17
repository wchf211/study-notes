# Coordination Service：ZooKeeper（Module 16）

> 这一章的论点一句话：**别在每个 app 里手搓 leader election（选主）/ lock（锁）/ config（配置）**。把 agreement 集中到一个久经考验的小服务（ZooKeeper），app 只用它的 API 拼 primitive（原语）。读法：先看 coordination 问题 → znode 数据模型 → guarantee → ZAB → recipe。Recipe 才是这章的产出。

## 1. Coordination Service：把 Agreement 集中到一个小服务

**Coordination service（协调服务）**：一个小、高可用、强一致的 store（存储），分布式系统用它干需要 agreement（一致）的活：configuration（配置）、group membership（组成员）、leader election（选主）、lock（锁）、barrier（屏障）、naming（命名）。原则：把 coordination 集中到一个 well-tested（久经考验）服务，而不是在每个 app 里嵌 ad-hoc agreement（临时拼的共识）。

几个节点必须同意“谁是 leader / 当前配置是什么 / 谁活着”时用。别把它当通用数据库 —— 它为小 metadata（元数据）和多读优化。例：所有 worker watch（监听）一个 config 节点；它变了，大家都 reload（重载）。失败模式：每个服务自己用 timeout 发明选主。Fix：用共享协调服务。

## 2. ZooKeeper：小 API + Watch，拼出 Primitive

**ZooKeeper** 暴露 hierarchical namespace（层级命名空间）的 znode（像个小文件系统），每个 znode 有 data、version、children。Client（客户端）用 session（会话）连接；znode 可以是 persistent（持久）、ephemeral（临时：会话结束就删）、sequential（顺序：服务器追加 monotonic number，单调递增号）。

API：create、delete、exists、get / set data、list children，加 one-time watch（一次性监听，znode 变了通知 client 一次）。原则：一个小、定义良好的 API + watch，让 client 搭更高 primitive。Coordination metadata 用 ZooKeeper，bulk data（批量数据）别用。别在 znode 里存大值。例：leader 建 ephemeral 节点；它死了，节点消失，follower 注意到。失败模式：把 watch 当持久订阅（它只触发一次）。Fix：每次通知后重新注册 watch。

## 3. ZooKeeper 的 Guarantee：写强，读便宜但可能 Stale

- **Single-system image（单一系统镜像）**：每个 replica 按同样顺序应用同样写。
- **Linearizable write（线性化写）**：所有写走 leader，total order（全序）。
- **Sequential consistency（顺序一致性）**：client 的操作按它发出的顺序被看到，所有 client 看到同样的写顺序。
- **FIFO client order（FIFO 客户端顺序）**：client 的请求按它发送的顺序执行。
- **Liveness（活性）**：多数服务器活着，服务就可用。
- **Durability（持久性）**：ack 过的写经得起 crash。
- **读**：follower 读可能 stale（过期，默认不是线性化）；要线性化读用 sync。

原则：写强，读便宜但可能 stale。Freshness（新鲜度）重要时用 sync / leader 读。别假设每个读都是线性化。例：写 ack 后，每个后来的写都看到它，但 follower 读可能还没。失败模式：从 lagging follower（落后 follower）读 stale config。Fix：读前先 sync。

## 4. ZAB：ZooKeeper 的共识引擎

**ZAB（ZooKeeper Atomic Broadcast）** 给所有写排序。两阶段：leader election（没 leader 时，服务器选 log 最新者当 leader，用 epoch，任期）+ broadcast（leader 给每个 proposal 分配 transaction ID（epoch, counter），follower 追加到 log，多数 ack 后 leader commit）。

原则：单个 leader + 多数 ack + epoch 号 = 写全序。把 ZAB 当 ZooKeeper 的共识引擎；operator（运维）不用实现它。别把 ZAB 和通用 Paxos 混淆 —— 它是为单 leader 复制日志特化的。例：一个写被提议，多数追加，然后提交、应用。失败模式：分区后老 leader 回来，试图提交 stale proposal（过期提案）。Fix：epoch 号 —— follower 拒绝旧 epoch 的 proposal。

## 5. Recipe：Ordering + Ephemerality + Watch = 大部分协调原语

从 znode + watch + ephemeral / sequential flag 搭出的 recipe：

- **Configuration management（配置管理）**：config 发布在 znode 里；client watch 它，变了就重读。
- **Group membership（组成员）**：group 节点下每成员一个 ephemeral child；列 children 得成员；死成员的节点自动消失。
- **Simple lock（简单锁）**：尝试建 ephemeral lock znode；已存在就 watch 它，等它消失再重试；释放时删除，死了自动删。
- **Lock without herd effect（无惊群效应的锁）**：每个竞争者建 sequential ephemeral 节点在 lock 路径下；最小 sequence 持有锁；其它只 watch 前驱（predecessor），每次释放只唤醒一个 waiter。

原则：ordering + ephemerality + watch = 大部分协调原语。用 sequential 节点强加公平顺序，用前驱 watch 避免 thundering herd（惊群）。别让所有 waiter watch 同一个节点。例：十个锁竞争者各只 watch 自己的前驱 —— 每次交接只唤醒一个。失败模式：herd effect（惊群效应），每次释放所有 waiter 都醒来争。Fix：sequential 节点 + 前驱 watch。

## 这门课怎么教的（Module 16）

先给动机 —— 每个分布式系统都要对 membership、leadership、configuration 达成 agreement，每个系统自己重实现容易错 —— 再给出 ZooKeeper 当一个小、强一致的服务，让很多系统共享。从 coordination 问题 → ZooKeeper 数据模型（znode）→ guarantee → 协议（ZAB）→ 基于 API 的 recipe。用文件系统式树图 + client / session 演练。每个 guarantee 映射到具体 recipe 时最易懂；难处是 ZAB 的两阶段/epoch 细节和 Paxos 重叠。

值得偷的：**先给动机，再给模型，最后给 recipe**。Recipe 是这章的产出，不是附带。讲 guarantee 时要绑定 recipe（哪个 guarantee 让哪个 recipe 成立），否则 guarantee 只是名词。**“一个小服务，很多人共享”** 是这章的框架 —— 讲 coordination 时先问“为什么不每个 app 自己做”，再讲共享服务。

## 自测 4 问（带答案要点）
1. 为什么不用每个 app 自己做 leader election / lock？
   → ad-hoc agreement 容易错；集中到一个 well-tested 服务，app 只拼 primitive。
2. Znode 三种类型 + watch 的关键性质？
   → persistent / ephemeral / sequential；watch 是一次性的，通知后要重新注册。
3. ZooKeeper 的写和读各什么保证？
   → 写线性化（走 leader，全序）+ 持久 + 活性；读默认可能 stale，要线性化读用 sync。
4. 无惊群锁怎么做？
   → sequential ephemeral 节点 + 只 watch 前驱；每次释放只唤醒一个 waiter。
