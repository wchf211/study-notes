# Teaching Lens Part 8：Modules 18–20 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 18–20（Part 8 已读部分）。Modules 21–31 的讲法等后续再补。

## 1. Module 18：Lever 当 Trade-Off 教

先给解耦问题（producer 不该等 consumer），再讲 Kafka 的 log 模型（topic → partition → 不可变 segment，offset 是位置），再把 tuning knob 当“lever”讲 —— 每个 lever trade durability / latency / ordering —— 再从 lever 推 guarantee，再加事务和存储细节。用 producer → broker → consumer 图和 offset timeline。

值得偷的：**Lever 当 trade-off 教**。Kafka 的 knob 不是“配置清单”，是“每个 lever trade 一样东西”—— 观众记住 trade-off，就不用背配置。**单个 lever 映射到一个 guarantee**：讲的时候要先说“这个 lever 换什么”，再说“它给什么保证”。

## 2. Module 19：Reconciliation Loop 当组织思想

短、架构优先。先给编排问题（放工作、保持运行、暴露它），再走控制平面 vs worker 拆分，reconciliation loop 是组织思想：desired state 在 etcd，controller 把 actual 推向 desired。用标准 manager / worker 图；独特动作是把 Kubernetes 框成 eventually-consistent、CAS-guarded state machine。

值得偷的：**一个思想组织全章**。Kubernetes 组件多，讲的时候要有一个组织思想（reconciliation loop），否则观众只记得一堆名字。**框成状态机**：把 Kubernetes 框成“CAS-guarded 状态机”，观众立刻知道“为什么有 resourceVersion”。

## 3. Module 20：对比当锚

先给“区块链不适合受监管金融”的理由（所有人看所有东西、无法律身份、概率终局），再讲 Corda 当“need-to-know”账本：state、contract、flow、notary。走数据模型 → 节点架构 → 向后兼容。用 IOU / 现金转移例子和 transaction-chain 图。

值得偷的：**对比当锚**。Corda 对 Bitcoin / Ethereum 的对比是理解 need-to-know 的最快路径。讲新系统时，先给“它不是什么”，再给“它是什么”，观众的锚就有了。

## 4. 容易讲砸的坑

1. **Kafka knob 讲成配置清单**：不讲 trade-off，观众不知道 acks=all + min ISR=1 为什么不给 durability。讲 lever 必须讲 trade。
2. **Kubernetes 讲成组件清单**：不讲 reconciliation loop，观众只记得一堆名字。讲组件时要绑定“它在循环里干什么”。
3. **Corda 讲成区块链变体**：不讲“它不是什么”，观众会把 Corda 当成“私有链”。讲的时候要先给三个区别。

## 自测 4 问（带答案要点）
1. Module 18 为什么 lever 当 trade-off 教？
   → 观众记住 trade-off，就不用背配置。
2. Module 19 的组织思想？
   → Reconciliation loop：desired state 在 etcd，controller 把 actual 推向 desired。
3. Module 20 为什么先给“它不是什么”？
   → 对比当锚；先给三个区别，观众的锚就有了。
4. 讲系统课的通用顺序？
   → 先给问题，再给模型，再给 lever / 保证。
