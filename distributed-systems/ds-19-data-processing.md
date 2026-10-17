# Distributed Data Processing（Module 21）

> 这一章是 chronological / evolutionary（按时间演进）结构：batch 先（MapReduce）→ in-memory batch + 多阶段（Spark）→ true streaming（Flink）。每系统都走：motivation → model → execution → failure handling → time 语义。课程用同一批 counting / aggregation 例子横跨三系统，读者能比出什么变了（disk vs memory、batch vs stream、processing-time vs event-time）。最密的是 Spark 的 RDD lineage / DAG 和 Flink 的 watermark / checkpointing。

## 1. 开篇：按数据到达方式选模式（Module 21.1）

**Data processing system（数据处理系统）** 把大数据集跨多机变成结果。两种模式：**batch（批处理）**（有限数据集，job 跑到完成）+ **stream（流处理）**（无界数据，持续处理）。框架：MapReduce（batch）、Spark（batch + micro-batch / stream）、Flink（true streaming，真流）。

原则：按数据到达方式选模式 —— batch 给完整数据集，stream 给持续数据。Batch 做日报；stream 做实时 dashboard / fraud（欺诈）。别把 streaming 强加到一次性 job 上，或把 batch 强加到低延迟需求上。例：夜间销售汇总（batch）vs 实时 click 计数（stream）。失败模式：把 stream 当 batch，等“结束”。Fix：window（窗口）、watermark（水位）、增量结果。

## 2. MapReduce：两个函数 + 框架干分布（Module 21.2）

**Programming model（编程模型）**：两个函数：**Map** 拿 input key-value 对，emit（发出）intermediate key-value 对；**Reduce** 拿一个 key + 它所有 value，emit output。框架处理 partitioning（分区）、shuffling / sorting by key（按 key 洗牌/排序）、scheduling（调度）、fault tolerance（容错）。

原则：把 job 表达成 per-key grouping + aggregation（按 key 分组 + 聚合）；系统做分布。大、简单、能 map / shuffle / reduce 的 batch job 用 MapReduce。Iterative（迭代）或 interactive（交互）job 别用 —— 每阶段写 disk。例：word count —— map emit (word,1)，shuffle 按 word 分组，reduce 求和。失败模式：需要多次迭代的 job 每次重写 disk。Fix：Spark / Flink 或换算法。

## 3. MapReduce Manager-Worker：中心调度 + 确定性重执行 = 容错（Module 21.3）

一个 manager（master / JobTracker）拆 input、把 map 和 reduce 任务分配给 worker、经 heartbeat 跟踪进度、重分配失败/慢任务（speculative execution，推测执行，治 straggler，掉队者）。Map worker 把 intermediate output 按 key 分区写本地 disk；reduce worker 取自己的 partition、排序、reduce。Reduce 任务不能在所有 map 输出就绪前开始（barrier，屏障）。**Data locality（数据局部性）**：优先在 input block 所在处跑 map 任务。

原则：中心调度 + 确定性重执行 = 容错。Embarrassingly parallel（天然并行）batch 工作在文件系统上用 MapReduce。别指望低延迟 —— barrier + disk 写占主导。例：失败的 map 任务在另一个 worker 上从 input split 重跑。失败模式：straggler 在 barrier 处拖住整个 job。Fix：speculative execution（跑个重复，取先完成者）。

## 4. Spark：RDD + Lineage，不复制中间数据（Module 21.4）

**Spark** 泛化 MapReduce：在 **RDD（Resilient Distributed Dataset，弹性分布式数据集）** 上工作 —— immutable（不可变）、partitioned（分区）集合，带 lineage（血缘：它怎么被派生的记录）—— 外加更高 API（DataFrame / Dataset、SQL、streaming、ML、graph）。RDD 操作是 lazy（惰性）：transformation（map、filter、join）建 DAG；action（count、collect、save）触发执行。Fault tolerance 经 lineage：丢掉的 partition 从祖先重算，不从复制数据。

原则：数据跨阶段留在内存，失败时重算。Iterative、多阶段、交互式分析用 Spark。Per-record 低延迟别用（本质是 batch / micro-batch）。例：迭代 ML 在 20 轮里复用同一数据集 —— 缓存内存。失败模式：缓存太多 spill（溢出）到 disk 或 OOM。Fix：persistence level（持久化级别）+ partitioning。

## 5. Spark DAG of Stages：Shuffle 是昂贵屏障（Module 21.5）

Spark job 是 DAG of stages（阶段有向无环图）：scheduler 在 shuffle boundary（洗牌边界）处拆 transformation 图。**Narrow dependency（窄依赖）**（map、filter —— 每个输出 partition 只需一个输入 partition）留在同阶段，可 pipeline；**wide dependency（宽依赖）**（groupByKey、join —— 每个输出需多个输入）强制 shuffle + 新阶段。阶段内任务跨 partition 并行跑；shuffle 写是昂贵 barrier。

原则：最小化 shuffle；能 co-partition（共分区）join 就共分区。用 DAG 视图找 shuffle bottleneck（瓶颈）。别用 groupBy，那里 reduceByKey / aggregateByKey 够用。例：map → filter → map 是一个阶段；groupByKey 开新阶段。失败模式：skewed key（倾斜 key）让一个 shuffle partition 巨大（straggler）。Fix：salting、repartitioning、或 combiner（合并器）。

## 6. Spark 的 Perks：一个引擎干多种活（Module 21.6）

相对经典 MapReduce 的优势：跨阶段 in-memory caching（快迭代 job）、更丰富的 API（SQL、DataFrame、streaming、MLlib、GraphX）在一个引擎、lazy evaluation + whole-stage optimisation（Catalyst optimizer、Tungsten execution）、lineage-based recovery 不复制中间数据。原则：一个引擎干多种 workload，减少 glue code（胶水代码）。

团队需要 SQL + ML + streaming 一起时，Spark 当默认 batch / 交互引擎。别指望 streaming 延迟低于 micro-batch 粒度。例：一个 Spark job 读、join、聚合、训模型。失败模式：用 Spark 做单记录 lookup 或亚秒事件处理。Fix：用 store 或 true streaming 引擎（Flink）。

## 7. Flink：True Streaming，Batch 是 Bounded Stream（Module 21.7）

**Flink** 是 true streaming 引擎：把 batch 当 streaming 的特例（bounded stream，有界流）。模型：dataflow graph（数据流图），source → operator（map、filter、keyBy、window、join）→ sink，分布式 pipeline 执行，带 backpressure（背压：下游慢，上游减速，不丢）。State（状态）：keyed state、operator state、checkpoint（检查点）做恢复。

原则：记录到就处理，带 event-time 语义。低延迟 streaming、事件驱动 app、exactly-once pipeline 用 Flink。Batch-only job，Spark 更简单时别用 Flink。例：欺诈检测，每笔交易到就打分。失败模式：keyed operator 里 unbounded state（无界状态）增长。Fix：TTL、state cleanup、windowed state。

## 8. Flink Time + Watermark：用 Watermark 换 Progress（Module 21.8）

三种时间：**event time（事件时间）**（事件发生时，来自它的 timestamp）、**ingestion time（摄入时间）**（Flink 收到时）、**processing time（处理时间）**（operator 处的 wall clock）。**Watermark（水位）** 是标记，说“不再有 timestamp ≤ W 的事件了”—— 让 window 在部分事件 late（迟到）时也能 fire（触发）。**Window（窗口）**：tumbling（滚动：固定、不重叠）、sliding（滑动：固定、重叠）、session（会话：按 gap）。

Watermark 后的 late 事件可 drop、side-output（侧输出）、或 allowed lateness（允许迟到）后接受。原则：watermark 用 completeness（完整性）换 progress（进展）。正确性依赖事件何时发生（不是何时到达）时用 event time + watermark。事件有序结果别用 processing time。例：1 分钟 event-time window 统计 click，watermark 延迟 5 秒容忍 late 到达。失败模式：watermark 推进太快丢 late 事件。Fix：bounded lateness、side output、幂等 late 处理。

## 9. Flink Failure Recovery：Checkpoint + Replay + 2PC Sink = Exactly-Once（Module 21.9）

**Asynchronous Barrier Snapshotting（ABS，异步屏障快照）**，Chandy-Lamport 式算法：JobManager 定期把 barrier（屏障）注入 stream；每个 operator 收到所有 input 的 barrier 后 snapshot 自己的 state（逻辑上 copy-on-write，物理拷贝在后台），并转发 barrier；所有 task ack 后 checkpoint 完成。恢复：所有 task 重置到最近 checkpoint，从那里 replay（重放），丢弃 checkpoint 后的 state。两阶段协议：阶段 1 = 拍 checkpoint，阶段 2 = commit 它。

State storage（状态存储）：in-memory、embedded KV、或 external store —— 有 MVCC 时，checkpoint 后的更新被版本化、恢复时忽略；没有 MVCC 时用 WAL（预写日志），在阶段 2 commit。**Exactly-once**：replayable source（Kafka offset）+ two-phase sink（TwoPhaseCommitSinkFunction：beginTransaction / preCommit / commit / abort）给 exactly-once 效果；非 replayable source 和外部 side effect（副作用）不覆盖。

原则：checkpoint + replay + 2PC sink = exactly-once 处理。长期 streaming job 用 ABS checkpoint。Repartitioning 后别指望 ordering，外部 side effect 别指望保证。例：Kafka source offset 被 checkpoint，sink 只在 checkpoint 完成后 commit Kafka 事务。失败模式：sink 在 checkpoint 完成前 commit，恢复时重复。Fix：2PC sink + checkpoint-coordinated commit。

## 这门课怎么教的（Module 21）

Chronological / evolutionary 结构：batch 先（MapReduce），再 in-memory batch + 多阶段（Spark），再 true streaming（Flink）。每系统都走：motivation → model → execution → failure handling → time 语义。课程用同一批 counting / aggregation 例子横跨三系统，读者能比出什么变了（disk vs memory、batch vs stream、processing-time vs event-time）。最密的是 Spark 的 RDD lineage / DAG 和 Flink 的 watermark / checkpointing。

值得偷的：**同一例子横跨三系统**。讲演进课时，不要每个系统换例子；同一 counting 例子，观众直接看到“什么变了”。**按演进讲，不按功能讲**：先讲 MapReduce 的局限（disk、barrier），再讲 Spark 怎么解（memory、DAG），再讲 Flink 怎么解（streaming、watermark）—— 每个新系统都是前一个的“修复”，观众知道它为什么存在。

## 自测 4 问（带答案要点）
1. MapReduce、Spark、Flink 各解决前一个的什么局限？
   → MapReduce：disk + barrier；Spark：memory + DAG + lineage；Flink：true streaming + event-time + watermark。
2. Spark 的 lineage 怎么给容错？
   → 丢 partition 从祖先重算，不复制中间数据。
3. Narrow vs wide dependency？
   → 窄：输出 partition 只需一个输入，可 pipeline；宽：需多个输入，强制 shuffle + 新阶段。
4. Flink 的 exactly-once 怎么来的？
   → Checkpoint + replay + 2PC sink；非 replayable source 和外部 side effect 不覆盖。
