# Teaching Lens Part 9：Module 21 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Module 21（Part 9 已读部分）。Modules 22–31 的讲法等后续再补。

## 1. 按演进讲，不按功能讲

Chronological / evolutionary 结构：batch 先（MapReduce），再 in-memory batch + 多阶段（Spark），再 true streaming（Flink）。每系统都走：motivation → model → execution → failure handling → time 语义。

值得偷的：**按演进讲**。先讲 MapReduce 的局限（disk、barrier），再讲 Spark 怎么解（memory、DAG），再讲 Flink 怎么解（streaming、watermark）—— 每个新系统都是前一个的“修复”，观众知道它为什么存在。**不要按功能讲**：按功能讲（MapReduce 有这些 API，Spark 有这些 API），观众不知道为什么要有三个系统。

## 2. 同一例子横跨三系统

课程用同一批 counting / aggregation 例子横跨三系统，读者能比出什么变了（disk vs memory、batch vs stream、processing-time vs event-time）。

值得偷的：**同一例子横跨**。讲演进课时，不要每个系统换例子；同一 counting 例子，观众直接看到“什么变了”。**对比表收尾**：讲完三系统，用一张表对比（disk vs memory、batch vs stream、time 语义），观众带走的是对比，不是三个孤立系统。

## 3. 容易讲砸的坑

1. **Flink 讲成“更快的 Spark”**：Flink 不是更快的 Spark，是 true streaming + event-time 语义。讲的时候要先说“它把 batch 当 bounded stream”，再讲 API。
2. **Watermark 只讲定义**：Watermark 是“用 completeness 换 progress”，要给例子（1 分钟 window，watermark 延迟 5 秒），观众才知道它在 trade 什么。
3. **Exactly-once 讲成“Flink 保证”**：Exactly-once 是 checkpoint + replay + 2PC sink 三件套，非 replayable source 和外部 side effect 不覆盖。讲的时候要说清边界。

## 自测 4 问（带答案要点）
1. Module 21 为什么按演进讲？
   → 每个新系统都是前一个的“修复”，观众知道它为什么存在。
2. 同一例子横跨三系统的好处？
   → 观众直接看到“什么变了”，不用每系统重新理解例子。
3. Flink 不是“更快的 Spark”，是什么？
   → True streaming + event-time 语义；batch 是 bounded stream。
4. Watermark 在 trade 什么？
   → Completeness 换 progress；推进太快丢 late 事件。
