# Teaching Lens Part 7：Module 17 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Module 17（Part 7 已读部分）。Modules 18–31 的讲法等后续再补。

## 1. 结构：比较，不是百科

Comparative 结构：HBase / Bigtable（单 master、LSM、强 per-row）、Cassandra（点对点、可调 quorum、默认最终一致，带一段仔细的 linearizability 绕行）、Spanner（全球分布、TrueTime、外部一致性），加 FaunaDB / Calvin 当 determinism 替代。每个系统都走：数据模型 → 架构 → 操作 → guarantee → gotcha。

值得偷的：**每个系统同一结构**。讲比较课时，不要一个系统讲架构、一个讲性能、一个讲历史；同一结构，观众才能比。**Gotcha 收尾**：每个系统最后给一个坑（HBase 热区、Cassandra QUORUM ≠ linearizable、Spanner 跨 tablet 慢、FaunaDB 无交互事务），观众带走的是“什么时候别用它”。

## 2. 同一批问题问每个系统

课程反复对每个 store 问同样的问题（谁给写排序？读看到什么？分区下会坏什么？），比较是 apples-to-apples。最密的是 Cassandra 的 linearizability 分析和 Spanner 的 TrueTime / commit-wait 推理。

值得偷的：**固定问题清单**。讲比较课时，不要每个系统换一套问题；固定问题清单，观众自己就能比。**“谁排序、读看到什么、分区下坏什么”** 是评价任何 store 的三问，要钉在黑板上。

## 3. 绕行要标出来

Cassandra 的 linearizability 绕行是“QUORUM ≠ linearizable”这个反直觉结论，讲的时候要先给结论，再给反例。

值得偷的：**反直觉结论先给**。观众以为 QUORUM = linearizable，先打破这个假设，再给反例（读 quorum 没包含见过写的 replica），记忆才深。**绕行要标出来**：告诉观众“这里是绕行”，他们就知道这段是为一个结论服务的，不是新章节。

## 4. 容易讲砸的坑

1. **每个系统换一套问题**：比较课最大的坑。固定问题清单，否则观众比不起来。
2. **QUORUM = linearizable**：不纠这个误解，观众会拿 QUORUM 做 leader election。讲 Cassandra 时必须给反例。
3. **TrueTime 讲成“精确时钟”**：TrueTime 的重点是“显式暴露不确定性”，不是“更精确”。讲的时候要先说“它返回区间，不是时间点”。

## 自测 4 问（带答案要点）
1. Module 17 每个系统的讲解结构？
   → 数据模型 → 架构 → 操作 → guarantee → gotcha。
2. 评价 store 的三问？
   → 谁给写排序？读看到什么？分区下会坏什么？
3. 为什么反直觉结论要先给？
   → 先打破假设，再给反例，记忆才深。
4. TrueTime 的重点是什么？
   → 显式暴露不确定性（返回区间，不是时间点）。
