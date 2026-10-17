# Teaching Lens Part 2：Modules 4–8 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 4–8（Part 2 已读部分）；Modules 9–31 的讲法等后续再补。

## 1. Module 4：一个桥接课，只干一件事

用一个多实体例子（人 + 银行账户 + 转账）说明单机 ACID 为什么不能免费扩展，并把接下来两章分工说清：isolation = 并发事务看到什么；atomicity = 跨节点 all-or-nothing。桥接课的价值就是分工 —— 观众知道接下来两章各解决一半问题，不会混。

## 2. Module 5：先定义，再并排比较机制

从形式定义起（serializable = 等价于某个串行顺序），再把机制并排摆：先锁（PCC / 2PL）vs 先执行后检查（OCC）vs 版本快照（MVCC）vs 快照 + 危险检测（SSI）。用同样的 read / write 时间线 + 锁/版本表反复出现，观众可以比较。难处：术语重叠（serializable vs snapshot vs SSI）。课程的解法是“同一冲突形状反复出现”，不是多讲定义。

值得偷的：**机制并排 + 同一冲突形状**。讲隔离级别时，不要一个一个孤立讲，要把同一个 write skew 例子分别用四种机制跑一遍，观众立刻看到差别。

## 3. Module 6：从不可能直觉到三个协议

用两节点转账故事 + coordinator 图起手。先给 impossibility 直觉（crash 下 commit / abort 必须一致），再讲 2PC，然后 3PC 当成 liveness 补丁，最后 quorum commit 当去中心化替代。重点放在每个协议在 crash vs partition 下各保证什么，不放在代码。值得偷的：**每个协议讲完都问一句“coordinator 挂了会怎样”** —— 这是读者真正会拿去用的判断题。

## 4. Module 7：合成表 + 务实出口

合成表式的一课（isolation 机制 × atomicity 机制），再用一个由前面零件拼成的示例应用展示组合，然后 pivot（转向）到 long-lived 工作：saga 是 2PC 式阻塞不可接受时的务实出口。值得偷的：**先给组合表，再给逃生口**。观众先知道“正统解法怎么拼”，再知道“正统解法太贵时怎么办”。

## 5. Module 8：先给 FLP，再讲 Paxos，最后给 Raft

先以 leader election / 复制日志为动机，早期就讲 FLP，让学习者对 safety-vs-liveness trade-off 有预期，再细讲 Paxos（角色、提案、承诺、接受），再讲实践形态（Multi-Paxos、稳定 leader、lease），再复用它做 RSM 和事务，最后把 Raft 当成“可理解的重新设计”讲。用编号阶段、分步消息流、反复的“哪里会坏”检查。最密的模块；Raft 部分有意比 Paxos 更清晰。

值得偷的：**FLP 前置**。先告诉观众“完美解不存在”，再讲协议，观众就不会问“为什么不直接……”；**Raft 当 Paxos 的重讲**，而不是并列新算法 —— 同样的保证，换一个更易懂的拆法。

## 6. 容易讲砸的坑

1. **术语重叠不给对照表**：serializable / snapshot / SSI、2PC / 3PC / quorum commit，连讲时必须有对照表（各解决什么、各在什么情况下坏）。
2. **协议只讲成功路径**：每个协议必须讲 coordinator 挂了 / 分区了会怎样，否则观众学到的是“协议名字”，不是“协议判断”。
3. **Paxos 讲成代码**：Paxos 要讲成“多数交集 + 采纳最大值”的一张图 + 消息流，而不是代码。代码是实现，图是理解。

## 自测 4 问（带答案要点）
1. Module 4 桥接课的唯一任务是什么？
   → 分工：isolation = 并发看到什么；atomicity = 跨节点 all-or-nothing。
2. Module 5 怎么帮观众比较四种隔离机制？
   → 先给 serializable 定义，再并排机制，用同一冲突形状反复跑。
3. Module 6 每个协议讲完必须问什么？
   → coordinator 挂了 / 分区了会怎样。
4. Module 8 为什么 FLP 前置？Raft 应该当成什么讲？
   → 先接受完美解不存在，再讲协议；Raft 是 Paxos 的可理解重讲，不是并列新算法。
