# Teaching Lens Part 10：Modules 22–31 + 全课讲法总结

> 这篇是讲法审计，不是技术复读。覆盖 Modules 22–31（Part 10 已读部分）。全课 31 modules 至此读完。

## 1. 最后一部分：Pattern 按“问题 → 机制 → 坑”讲

Module 22（Intro）是一课桥接，reframe 课程最后一部分：从学系统到可复用的 practice / pattern，工程师能直接用。列出前面的 pattern 组（communication、coordination、synchronisation、shared-nothing、locking、compatibility、failure、tracing）。

Module 23（Communication）按数据生命周期组织：先怎么表示数据（serialisation），再怎么搬（sync / async transfer），再 async 消息住哪（datastore），再拓扑（model）。用 request / response vs queue vs log 例子；强调每个选择对 coupling、latency、reliability 的 trade-off。

Module 24（Coordination）一课对比两种协调多步工作的方式：orchestration vs choreography。用多服务业务 workflow 当 running example；决策规则是“谁拥有 flow state”。

Module 25（Synchronization）从一般问题（副本/视图保持同步）到两个具体机制：event sourcing（存 log，派生 state）和 CDC（从源 log 发布变更）。用 primary → derived-view 例子贯穿；强调 ordering、duplicate、replay。

Module 26（Shared-nothing）先诊断共享为什么 hurt（协调、竞争、单点失败），再讲 shared-nothing 当设计回应：partition 让每个节点拥有自己的数据/状态。两课用同一共享资源例子做对比。

Module 27（Locking）两课弧线：先 lease 原语（有时限的锁 + 续租），再安全洞（lease 可能在持有者还在行动时过期）和 fix（fencing）。用单个 leader 持锁的故事；fencing-token 例子是 memorable takeaway。

Module 28（Compatibility）先定义 backwards vs forwards compatibility，配 schema / API 例子，再给维护实践（additive change、versioning、expand-then-contract）。用字段加/删演练；关键动作是“旧 reader 必须忽略它不懂的东西”。

Module 29（Failure）先 taxonomy 再应用：先 toolbox（retry、timeout、circuit breaker、bulkhead、fallback、backpressure），再什么时候组合，再深挖 retry、containment、backpressure（它是什么、怎么反应）。用 cascading-failure 故事；强调阻止局部失败变全局。

Module 30（Tracing）一课讲可观测性：经 correlation / trace ID、span、collector 跟踪一个请求跨很多服务。用多服务请求 timeline；要点是 log alone 不能重建分布式执行。

Module 31（Recap）回顾 + 指针。Recap 重述课程 spine（foundations → transaction / consensus / time / order / network / security → case study → pattern）和“no silver bullet”信息；最后一课指向更深材料（CockroachDB 当 worked example、gray / partial failure、TLA+ 形式验证）和两门后续课程。

值得偷的：**最后一部分要按“问题 → 机制 → 坑”讲 pattern**，不按“概念清单”讲。**决策规则收尾**：每组 pattern 最后给一条决策规则（Module 24 的“谁拥有 flow state”、Module 29 的“按失败模式匹配技术”），观众离开时带走的是判断，不是名词。**Recap 当 checklist**：最后一课不是总结，是设计 checklist + 深入指针，观众能直接用。

## 2. 全课讲法总结（Modules 1–31）

- **同一批原语反复出现**：partitioning、replication、quorum、consistency、consensus —— 每章不是新概念，是新组合。讲的时候要显式说“这是第 X 次出现”。
- **先给 workload，再给设计**：GFS、HBase、Spanner —— 每个设计选择都要能追回 workload 假设。
- **同一例子 / 同一图反复用**：3 节点执行图、双 host 消息、同一 counting 例子 —— 换方案不换图，观众直接比。
- **同一批问题问每个系统**：谁排序、读看到什么、分区下坏什么 —— 固定问题清单，比较才 apples-to-apples。
- **每个协议/pattern 讲完问“会坏在哪”**：coordinator 挂了、lease 过期了、watermark 太快了 —— 失败模式是记忆点。
- **Ablation / 对比当证明**：去掉它会坏什么、两个设计同一任务跑一遍 —— 比“它多厉害”更有说服力。
- **Recap 当 checklist + 指针**：最后一课不是总结，是设计 checklist + 深入指针。

## 自测 4 问（带答案要点）
1. 最后一部分为什么按“问题 → 机制 → 坑”讲 pattern？
   → 观众带走的是判断，不是名词；每组 pattern 最后给决策规则。
2. 全课最核心的讲法是什么？
   → 同一批原语反复出现；每章不是新概念，是新组合；讲的时候要显式说“这是第 X 次出现”。
3. 为什么“同一例子 / 同一图反复用”？
   → 换方案不换图，观众直接比。
4. Recap 应该当什么讲？
   → 设计 checklist + 深入指针，不是总结。
