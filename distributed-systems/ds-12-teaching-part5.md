# Teaching Lens Part 5：Modules 13–15 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 13–15（Part 5 已读部分）。Modules 16–31 的讲法等后续再补。

## 1. Module 13：每协议同一弧线

协议一个一个讲 Module 12 的目标。每课同一弧线：解决什么问题 → 谁信任谁 → 消息流 → 哪一步跳过会坏。TLS 是 confidential + authenticated 通道的 worked example；PKI vs Web of Trust 对比集中 vs 去中心化信任；OAuth 对比 delegation vs 密码共享。图是 handshake / certificate-chain 流。

值得偷的：**每协议同一弧线**。观众不用每课重新找“这课在讲什么结构”，直接比协议。**“哪一步跳过会坏”** 是协议课的灵魂 —— 跳过证书校验会怎样，观众立刻记住。

## 2. Module 14：一课桥接，定预期

一课桥接，显式 reframing 课程后半部分：同一批概念（partitioning、replication、consensus、consistency）现在要在真实系统里认出来，而不是当新理论引入。定预期：case study 是比较，不是百科。

值得偷的：**桥接课要短，要定预期**。观众知道后半部分是“认原语”，不是“背产品”，读 case study 的方式就对了。

## 3. Module 15：先给 Workload，再给设计

用 GFS 当 running system，HDFS 当紧密比较。先给 workload 假设（大文件、多 append、多读者），每个设计选择都是它推出来的，再按执行顺序走操作（create / read → write / delete → consistency）。用 manager / chunk-server 图和 chain replication 演练。

值得偷的：**先给 workload，再给设计**。每个设计选择都要能追回 workload 假设，否则观众会觉得“为什么不直接做强一致”。**紧密比较**（GFS vs HDFS）比单讲一个系统更能显出设计选择。

## 4. 容易讲砸的坑

1. **协议课只讲消息流，不讲信任**：TLS 握手不讲“谁信任谁”，观众只记得四步。讲协议先讲信任模型，再讲消息。
2. **Case study 讲成产品百科**：GFS 讲成“GFS 有这些功能”，观众记不住。讲成“workload 假设 → 设计选择 → 保证”，观众能迁移。
3. **弱一致性不给例子**：GFS 的 consistent-but-undefined 不给并发 append 例子，观众不知道它在说什么。讲弱模型必须给具体并发场景。

## 自测 4 问（带答案要点）
1. Module 13 每协议的弧线？
   → 解决什么 → 谁信任谁 → 消息流 → 哪一步跳过会坏。
2. Module 14 桥接课的任务？
   → 定预期：后半部分是“在真实系统里认原语”，不是“背产品”。
3. Module 15 为什么先给 workload？
   → 每个设计选择都要能追回 workload 假设，否则观众会觉得“为什么不直接做强一致”。
4. 讲弱一致性模型必须给什么？
   → 具体并发场景（如并发 append），否则 consistent-but-undefined 是空话。
