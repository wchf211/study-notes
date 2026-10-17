# Practices & Patterns + Tracing + Recap（Modules 22–31）

> 全课最后一部分：从“学系统”转到“可复用的 practice / pattern（实践/模式）”。读法：每组 pattern 先问“它解决什么问题”，再看“机制”，最后看“坑”。收尾 recap 把全课 spine（脊柱）再说一遍 —— foundations → building blocks → case studies → patterns —— 同一批原语反复出现。

## 1. 桥：Pattern 是可迁移的部分（Module 22）

桥接：课程现在收集可复用的 practice（实践），建分布式系统 —— 服务怎么通信、协调、保持同步、避免共享、安全加锁、保持兼容、处理失败、保持可观测。原则：pattern 是 transferable（可迁移）部分；具体系统是实例。

设计新服务时把 pattern 部分当 checklist。别把 pattern 当 rigid recipe（僵硬配方）。例：新服务需要 communication + coordination + failure handling —— 每组选一个 pattern。失败模式：抄某系统架构却没抄它的 pattern。Fix：点名你在用哪些 pattern。

## 2. Communication Pattern：按数据生命周期组织（Module 23）

**Creating and Parsing Data（创建和解析数据）**：发送数据必须先 serialise（序列化，结构化数据 → byte）再 parse（解析）回来。选项：native language serialisation（容易、不安全、版本脆弱）、text format（JSON / XML —— 可读、冗长、无 schema）、schema-based binary library（Protobuf、Thrift、Avro —— 紧凑、typed、versioned）、IDL-based tool（定义 schema 一次，生成代码）。

原则：format 是 compatibility contract（兼容性契约），不只是编码。跨服务、演进的 API 用 schema-based format。跨信任/版本边界别用 native serialisation。例：给 Protobuf 消息加字段 —— 旧 reader 忽略它。失败模式：format 变更破坏旧 parser。Fix：schema evolution 规则（optional field、default、永不复用 field number）。

**Transfer of Data（传输数据）**：**Synchronous（同步）**：sender 调用并等 reply（RPC / HTTP）—— 简单、耦合、caller 被阻塞、失败立即可见。**Asynchronous（异步）**：sender 交给 queue / log / stream 就继续；receiver 之后处理 —— 解耦、缓冲，但 ordering / delivery / dedup 变成 app 的问题。还有 shared database 当传输机制（一服务写、另一服务读）—— 简单但耦合 schema。

原则：在 coupling vs latency / availability 之间选。需要立即反馈的 request / response 用 sync；fire-and-forget、缓冲、fan-out 用 async。长 workflow 别用 sync 链。例：checkout 同步调 payment，但异步发布“order placed”。失败模式：sync 链里一个慢服务拖住所有 caller。Fix：async + timeout + fallback。

**Datastores for Async Communication（异步通信的存储）**：异步通信需要地方在发送和接收之间存消息：message queue（瞬态、每消息投递，如 SQS / RabbitMQ）、event log（持久、可重放、每 partition 有序，如 Kafka）、database（共享表当 queue —— 简单但 polling + 耦合）、file / object store（大 payload，消息里放 pointer）。

原则：store 决定 retention、replay、ordering。Replayable event 用 log；一次性任务用 queue。规模大了别用数据库表当 queue。例：order event 存在 Kafka，新 consumer 从头重放。失败模式：queue 投递即删，丢了 crash consumer 没处理的消息。Fix：ack-after-processing + redelivery。

**Communication Models（通信模型）**：拓扑：point-to-point（一对一，如 RPC、direct queue）、publish-subscribe（一 publisher → 多 subscriber，经 topic；subscriber 和 publisher 解耦）、request-response（配对 request / reply，sync 或 async）、broadcast / gossip（所有人听到，做 membership / dissemination）。

原则：模型决定谁知道谁。Fan-out 和可扩展用 pub / sub；定向工作用 point-to-point。Topic 能做的别用 broadcast。例：价格变更事件发布一次，pricing、analytics、notification 各自独立消费。失败模式：point-to-point fan-out，加 consumer 就要改 producer。Fix：pub / sub。

## 3. Coordination Pattern：Orchestration vs Choreography（Module 24）

多步 workflow 的两种 pattern：**orchestration（编排）**（中央 orchestrator / saga coordinator 按顺序调每个服务、处理失败/补偿 —— flow 逻辑在一个地方，易看易 debug，但 orchestrator 是耦合点）vs **choreography（编排式/事件编排）**（每个服务监听事件、自己触发下一步 —— 没有中央控制器，更解耦，但整体 flow 是隐式的、更难 trace / debug）。

原则：flow 集中还是分布 —— 刻意选。复杂 flow 需要可见/控制时用 orchestration；简单、松耦合事件链用 choreography。别不点名谁拥有 flow 就混用两者。例：order → payment → inventory → shipping 由 orchestrator 驱动 vs 每个服务对上一步事件反应。失败模式：choreographed flow 失败时没人知道当前步骤。Fix：orchestration 或共享 saga state + tracing。

## 4. Data Synchronization：Event Sourcing + CDC（Module 25）

**Data synchronisation（数据同步）**：让数据副本/视图保持一致：选项有 dual write（app 代码里写两个 store —— 脆弱）、two-phase commit（强、阻塞）、asynchronous replication（log-based，最终一致）、event-driven sync（发布变更，consumer 更新视图）。

原则：同步是 freshness、coupling、complexity 之间的 trade-off。Derived view（派生视图）用 log / event-driven 同步；dual write 避免。没有协议别假设两个 store 原子更新。例：search index 经 change log 和主 DB 保持同步。失败模式：dual write 里第二次写失败，视图悄悄分歧。Fix：单一事实来源 + change propagation（变更传播）。

**Event sourcing（事件溯源）**：存 event 序列（发生了什么）当事实来源，replay（重放）它派生当前 state（常用 snapshot 加速 replay）。好处：完整 audit history、time travel（时间旅行）、易重建视图、天然 fit CQRS（读写模型分离）。代价：event schema evolution、存储增长、replay 时间、需要 idempotent projection（幂等投影）。

原则：log 是 truth；state 是 cache。Audit-heavy、workflow-like 领域用 event sourcing。简单 CRUD 别用。例：银行账户靠 replay 存取款重建。失败模式：event schema 变更破坏旧 event 的 replay。Fix：versioned event、upcaster、snapshot。

**CDC（Change Data Capture，变更数据捕获）**：从源数据库的 transaction log 发布 row-level 变更（insert / update / delete）给下游 consumer，不 polling、不改 app。方法：log-based（读 WAL / binlog —— 低影响、有序、完整）、trigger-based（DB trigger 写 outbox —— 更多开销）、query-based（poll updated timestamp —— 漏 delete、滞后）。

原则：复用 DB 自己的 log 当 change stream。Log-based CDC（如 Debezium）喂 search index、cache、warehouse、或其它服务。规模大了别 poll 表查变更。例：Postgres WAL 流到 Kafka，再到 Elasticsearch。失败模式：CDC consumer 处理不了 delete 或乱序 event，分歧。Fix：tombstone（墓碑）、ordering key、幂等 consumer。

## 5. Shared-nothing：去掉共享，去掉协调（Module 26）

**Sharing problem（共享问题）**：跨节点共享资源（disk、memory、lock、database）带来 contention（竞争）、协调开销、single point of failure、限制 scaling：每个节点都要协调才能用它。**Shared-nothing（无共享）** 回应：partition（分区）数据和工作，每个节点拥有自己的一份、不需要共享资源（每个节点有自己的 CPU、memory、disk）。剩余共享（如网络、coordination service）最小化。

原则：去掉共享，去掉协调。Horizontal scale 用 shared-nothing（sharding + 独立节点）。别共享一个 database / lock 给所有节点还指望线性 scale。例：sharded store，每 shard 由一组节点服务。失败模式：共享锁串行化所有写。Fix：分区，让每次写只碰一个 shard。

**Benefits（好处）**：近似线性 scale-out（加节点、加容量）、fault isolation（一节点失败不拖垮其它）、无共享 contention。**Drawbacks（代价）**：跨 partition 操作更难（join、事务）、节点加入/离开时 rebalancing（再平衡）痛、每 partition 重复逻辑、需要 routing / coordination 层。

原则：shared-nothing 用跨 partition 简单换 scale。多数操作是单 partition 时用。多数操作跨 partition 时别用。例：按用户 sharded 的服务靠加 shard 扩展。失败模式：“get all users”查询 fan-out 到每个 shard。Fix：scatter-gather 限制、secondary index、或不同布局。

## 6. Distributed Locking：Lease + Fencing（Module 27）

**Lease（租约）**：time-bounded grant（有时限的授予，如“你持有这个 lock / 是 leader，接下来 10 秒”），由 coordination service 发；持有者必须在过期前 renew（续租），过期后别人可接管。原则：时间界限把 stuck lock（卡住的锁）变成可恢复的 —— 不用手动清理。

Leader election、lock ownership、cache validity 用 lease。持有者可能 crash 的系统里别用 unbounded lock（无界锁）。例：leader 持 10 秒 lease，每 3 秒续；它死了，另一节点过期后接管。失败模式：lease 太短导致 constant churn（频繁更替）；太长延迟 failover。Fix：调 lease 时长 vs 续租间隔 vs clock skew。

**Lease 的安全洞 + Fix**：危险：持有者的 lease 过期了（GC pause、网络延迟），但它还以为自己持有 lock、还在行动；新持有者也在行动 —— 两个写者。Fix 用 **fencing token（围栏令牌）**：lock service 给每次 lease 发 monotonically increasing token（单调递增令牌）；每次写带 token；storage 层拒绝 stale token 的写（它记得见过的最高 token）。

原则：别靠“我觉得我持有 lock”—— 让 resource 强制执行。任何 guard 写（保护写）的锁都用 fencing token。别让前持有者 unchecked 写。例：旧 leader 带 token 7 写；storage 见过 token 8，拒绝。失败模式：忘了 fencing、只靠 lease 过期。Fix：fencing token + 单调排序。

## 7. Compatibility Pattern：Additive + Expand-then-Contract（Module 28）

**Backwards compatibility（向后兼容）**：新版本能读数据 / 和旧版本互操作（新 reader、旧 writer）。**Forwards compatibility（向前兼容）**：旧版本能处理新版本的数据（旧 reader、新 writer）—— 通常靠忽略未知字段。原则：rolling upgrade（滚动升级）里旧新版本共存，所以每次变更在两个方向都要兼容，直到所有节点升级。

把它当任何 schema / API 变更的 gate（门）。别假设 big-bang upgrade（一次性全升）可行。例：新服务版本读旧消息，旧版本忽略新字段。失败模式：新字段被旧代码误解成有意义。Fix：optional field、default、永不 repurpose（重用）字段。

**维护向后兼容的实践**：先只做 additive change（加新字段/endpoint，永不在一步里删/改名/改类型）；用 versioned API 和 wire format；**expand-then-contract（先扩展、再收缩）**（部署同时处理新旧的代码，迁移数据，再删旧路径）；保持 preserve 旧行为的 default；测 mixed-version 部署。

原则：兼容靠 process（流程），不是靠运气。任何 breaking change 用 expand-contract。别在加替代品当天就删字段。例：加字段 v2，部署接受两者的 reader，backfill，再删 v1。失败模式：删了旧 client 还在用的 endpoint。Fix：deprecation window（弃用窗口）+ 删前 usage monitoring（使用监控）。

## 8. Dealing with Failure：Toolbox + 组合（Module 29）

**Toolbox（工具箱）**：timeout（限等待时间）、retry（重尝试 transient 失败）、circuit breaker（断路器：失败达阈值后停止调失败依赖、快速失败，再 half-open 测恢复）、bulkhead（舱壁：按依赖隔离资源，一个失败不能耗尽所有线程/连接）、fallback（降级响应）、hedging（对冲：发重复请求，取先回）。

原则：快速失败、隔离、降级 —— 别让一个失败 cascade（级联）。到处用 timeout；关键依赖加 breaker + bulkhead。Retry 别无界、别无幂等。例：payment 调用 timeout → breaker 打开 → caller 拿到快速 fallback。失败模式：无界 retry 放大 outage（retry storm，重试风暴）。Fix：bounded retry + backoff + breaker。

**组合**：按依赖组合技术：timeout + bounded retry 给 transient 错误；breaker + fallback 给持续失败；bulkhead 保护共享池；hedging 只给幂等、低延迟敏感读。在正确层应用（client、proxy、service），阈值从真实延迟/错误数据调。

原则：没有单一技术适合所有失败 —— 按失败模式匹配技术。用 per-dependency policy（每依赖策略），不用全局策略。别给 payment 和只读 lookup 同一套 retry 策略。例：只读 lookup 用 hedging；payment 不 hedging、至多一次谨慎 retry。失败模式：hedging 非幂等写，复制它。Fix：hedging 只限安全读。

**Retry**：只在失败可能 transient 且操作可安全重复时 retry：用 idempotency key / 幂等操作、bound 尝试次数、exponential backoff + jitter（避免同步 retry storm）、尊重 Retry-After / breaker 状态。区分 retryable（timeout、503）和 non-retryable（400、auth）错误。

原则：retry 用额外负载换更高成功率 —— 只有系统能吸收负载时才用。默认 retry 策略：backoff + jitter + 幂等。别在紧循环里立即 retry。例：timeout 的请求 100ms、200ms、400ms 后带 jitter 重试。失败模式：outage 后几千 client 同时 retry（thundering herd，惊群）。Fix：jitter、backoff、breaker。

**Containing impact（控制影响）**：限制 blast radius（爆炸半径）：隔离 tenant / 依赖（bulkhead）、shed load（只接能服务的）、rate-limit（限流）、graceful degradation（优雅降级：服务 cached / stale 数据或缩减功能集）、feature flag / kill switch（快速关掉失败路径）。

原则：假设失败会发生 —— 设计成一个失败不能拖垮一切。用 bulkhead + load shedding + graceful degradation。别让非关键依赖拖垮整个请求。例：推荐服务挂了 → 首页仍用 cached 列表服务。失败模式：慢下游耗尽所有线程、杀死 caller。Fix：bulkhead、timeout、shedding。

**Backpressure（背压）**：下游组件过载的信号，告诉上游慢下来（而不是丢工作或无界排队）。机制：bounded queue（有界队列：满了 block / reject）、flow control（TCP 式 window、gRPC / HTTP2 流控）、pull-based consumption（consumer 控制速率，如 Kafka consumer poll）、显式信号（503 + Retry-After、rate limit）。

原则：最慢阶段定节奏 —— 别让队列无界增长。用 bounded queue + pull-based flow。别用 unbounded queue（无界队列）。例：快 producer 在 consumer 缓冲满时被 block。失败模式：无界队列藏住过载，直到内存耗尽。Fix：bounded queue + backpressure + shedding。

**Reacting to backpressure（应对背压）**：上游反应：slow down（降速、batch）、shed（丢低优先级工作，返回 429 / 503）、短暂 buffer（有界）、或 degrade（服务更便宜的结果）。按优先级选：保护关键流量，shed 其余；让 shedding 显式且可观测（metric、alert）。

原则：backpressure 是信息 —— 行动，别忽略。过载时用 load shedding + 优先级。别立即 retry 被 shed 的请求。例：API 过载时返回 429 + Retry-After；client back off。失败模式：client 全速 retry 被 shed 的请求，加深过载。Fix：client 侧 backoff、jitter、尊重 Retry-After。

## 9. Distributed Tracing：Correlation ID 把很多 Log 变成一个故事（Module 30）

**Distributed tracing（分布式追踪）** 跟踪一个请求跨服务：每个请求一个 trace ID（在 header 里传播）；每个工作单元是一个 span（服务、操作、开始/结束、tag）；span 连成 trace tree；span 被收集、可视化（如 Jaeger / Zipkin）。

用它找延迟累积处和错误来源。原则：correlation ID（关联 ID）把很多无关 log 变成一个故事。Latency debug 和 dependency mapping 用 tracing。别只靠每服务 log。例：慢 checkout 被 trace 到一个下游调用占 90% 时间。失败模式：context propagation（上下文传播）缺失（一服务丢了 trace ID），trace 断了。Fix：经每个 hop 传播 context，包括 queue 和 async 调用。

## 10. Recap + 深入指针（Module 31）

**Recap（回顾）**：foundations（为什么分布、谬误、部分失败、safety / liveness、模型、失败分类、exactly-once、state）、building block（partitioning、replication、quorum、consistency / isolation、transaction、consensus、time / order、networking、security）、真实系统（文件系统、coordination、data store、messaging、cluster management、ledger、data processing）、practice（communication、coordination、synchronisation、shared-nothing、locking、compatibility、failure handling、tracing）。

原则：同一批原语反复出现 —— 学会认出它们。把 recap 当设计 checklist。别把任何一个系统当答案。例：一个新 store 是“partitioned + replicated + quorum + X consistency”。失败模式：从 buzzword（流行词）设计，不从 guarantee。Fix：显式点名 guarantee 和 trade-off。

**Some more things to discover（再深入的指针）**：把 CockroachDB 当 worked example 学（pipelined consensus、单往返并行 2PC）、读 Amazon Builders Library 和 Hamilton、Brewer 的 paper 学 operator 课、留意 gray failure（灰色失败：微妙，不同部分观测不同 —— 不是干净的 up / down）和 partial failure（部分失败：系统一部分失败，带全局失败一样的后果，常来自设计缺陷）、学 formal verification（形式验证，TLA+）证明 safety / liveness、生产前找设计 bug。

原则：这课是基础，不是天花板。用这些指针上一个台阶。别因为 gray / partial failure 难检测就忽略它们。例：disk 慢但没死 —— gray failure。失败模式：只测干净 crash 失败。Fix：fault injection（故障注入）、gray-failure 检测、形式建模。

## 这门课怎么教的（Modules 22–31）

Module 22（Intro）是一课桥接，reframe（重新框定）课程最后一部分：从学系统到可复用的 practice / pattern，工程师能直接用。列出前面的 pattern 组（communication、coordination、synchronisation、shared-nothing、locking、compatibility、failure、tracing）。

Module 23（Communication）按数据生命周期组织：先怎么表示数据（serialisation），再怎么搬（sync / async transfer），再 async 消息住哪（datastore），再拓扑（model）。用 request / response vs queue vs log 例子；强调每个选择对 coupling、latency、reliability 的 trade-off。

Module 24（Coordination）一课对比两种协调多步工作的方式：orchestration（一个 controller 驱动步骤）vs choreography（每个服务对事件反应）。用多服务业务 workflow 当 running example；决策规则是“谁拥有 flow state”。

Module 25（Synchronization）从一般问题（副本/视图保持同步）到两个具体机制：event sourcing（存 log，派生 state）和 CDC（从源 log 发布变更）。用 primary → derived-view 例子贯穿；强调 ordering、duplicate、replay。

Module 26（Shared-nothing）先诊断共享为什么 hurt（协调、竞争、单点失败），再讲 shared-nothing 当设计回应：partition 让每个节点拥有自己的数据/状态。两课用同一共享资源例子做对比。

Module 27（Locking）两课弧线：先 lease 原语（有时限的锁 + 续租），再安全洞（lease 可能在持有者还在行动时过期）和 fix（fencing）。用单个 leader 持锁的故事；fencing-token 例子是 memorable takeaway（最记得住的收获）。

Module 28（Compatibility）先定义 backwards vs forwards compatibility，配 schema / API 例子，再给维护实践（additive change、versioning、expand-then-contract）。用字段加/删演练；关键动作是“旧 reader 必须忽略它不懂的东西”。

Module 29（Failure）先 taxonomy（分类）再应用：先 toolbox（retry、timeout、circuit breaker、bulkhead、fallback、backpressure），再什么时候组合，再深挖 retry、containment、backpressure（它是什么、怎么反应）。用 cascading-failure 故事；强调阻止局部失败变全局。

Module 30（Tracing）一课讲可观测性：经 correlation / trace ID、span、collector 跟踪一个请求跨很多服务。用多服务请求 timeline；要点是 log alone（光 log）不能重建分布式执行。

Module 31（Recap）回顾 + 指针。Recap 重述课程 spine（foundations → transaction / consensus / time / order / network / security → case study → pattern）和“no silver bullet（没有银弹）”信息；最后一课指向更深材料（CockroachDB 当 worked example、gray / partial failure、TLA+ 形式验证）和两门后续课程。

值得偷的：**最后一部分要按“问题 → 机制 → 坑”讲 pattern**，不按“概念清单”讲。**决策规则收尾**：每组 pattern 最后给一条决策规则（Module 24 的“谁拥有 flow state”、Module 29 的“按失败模式匹配技术”），观众离开时带走的是判断，不是名词。**Recap 当 checklist**：最后一课不是总结，是设计 checklist + 深入指针，观众能直接用。

## 自测 4 问（带答案要点）
1. Orchestration vs choreography 怎么选？
   → 复杂 flow 需要可见/控制用 orchestration；简单、松耦合事件链用 choreography；决策规则：谁拥有 flow state。
2. Event sourcing 和 CDC 各是什么？各的代价？
   → Event sourcing：存 event 序列当事实来源，replay 派生 state；代价 schema evolution、存储、replay；CDC：从源 DB log 发布变更；代价 consumer 处理 delete / 乱序。
3. Lease 的安全洞是什么？Fencing 怎么修？
   → Lease 过期但持有者还在行动，两个写者；fencing token 单调递增，storage 拒绝 stale token 写。
4. 失败处理怎么组合？
   → 按失败模式匹配技术：timeout + bounded retry 给 transient；breaker + fallback 给持续失败；bulkhead 保护共享池；hedging 只给幂等读。
