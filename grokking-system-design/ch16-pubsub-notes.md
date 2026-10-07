# Ch16: Pub-Sub — 提炼笔记（71–73，2026-10-06）


## 71. Pub-Sub 抽象
- 核心：用 topic + event broker 把 publisher 和 subscriber **解耦**——双方互不知晓，可独立扩展、故障隔离。
- 消息流：producers → topic 存储 → broker（路由、缓冲、保证投递）→ per-subscriber 投递队列 → reader。
- 关键 trade-off：
  - 队列拓扑：共享 topic 队列 vs 每消费者独立队列 vs 混合。
  - 订阅过滤：subject-based（按 topic 名，简单）vs content-based（按消息属性过滤，灵活但贵）。
  - 延迟 vs 持久：放内存延迟低但可能丢；落盘安全但慢。
- Kafka 是标准参照实现：topic 切 partition、immutable log segments、offset 寻址、多节点复制。

## 72. 需求与 API
- 四大用例：推送通知（代替昂贵轮询）、ingestion（Meta Scribe 缓冲海量日志再入库）、实时监控、数据复制（leader→follower 经 topic 广播）。
- 功能需求：建 topic、publish、subscribe/unsubscribe、read、**retention 策略**、过期自动删除——能说出 retention 说明你想到了存储生命周期，不止 happy path。
- API：create / write / read / subscribe / unsubscribe / delete_topic。
- Building blocks：DB（元数据+订阅关系）、MQ（写缓冲）、KV（消费者状态如 read position）。
- 数字：单条消息上限 **1MB**。

## 73. 设计：从朴素到 broker
- 朴素设计：message director 查 DB 找订阅者，**每条消息复制到每个消费者的独立队列**——规模一上来就崩：百万级队列的元数据/内存爆炸 + N 倍存储放大。
- Broker 设计（Kafka 式）：topic 按 **weighted round-robin** 切 partition；每个 partition 是 **immutable append-only log**（segments + offset 寻址），天然 per-partition 有序，无需复制数据。
- Cluster manager：broker/topic 注册表、**leader-follower 复制（3 副本/partition）**、leader 选举、鉴权。
- Consumer manager：验消费者、执行 retention、支持 push/pull（**pull 保护慢消费者不被打爆**）、offset 存 KV store 断点续读。
- Retention 默认 **7 天**可配：banking 要数周（可 replay），analytics 可立即丢——存成本 vs replay 能力的 trade-off。
- 口径：开场先讲 decoupling（独立扩展+故障隔离），delivery guarantees 三档要会说。

## 自测 4 问（带答案要点）
1. 为什么不用"每个 subscriber 一个独立队列"的朴素设计？
   → 百万级队列的元数据/内存爆炸 + 每条消息复制 N 份的存储放大。
2. Kafka 式 broker 设计的核心是什么？
   → topic 切 partition（weighted round-robin）；partition 是 immutable append-only log + offset 寻址，per-partition 有序；3 副本 leader-follower 复制。
3. Consumer offset 存在哪？有什么用？
   → 存 KV store；断点续读，消费者挂了从上次位置继续。
4. Retention policy 怎么定？trade-off 是什么？
   → 默认 7 天可配；banking 数周（要 replay），analytics 可立即丢；存成本 vs replay 能力。
