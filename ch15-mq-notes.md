# Ch15: Distributed Messaging Queue — 提炼笔记（64–69，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。
> 注：课程内该章现标为 Chapter 17（章节编号有漂移），按 lesson 编号 64–69 推进。

## 64. 消息队列是什么
- 生产者/消费者之间的**异步缓冲**：双方独立、并发工作，生产者永不阻塞。
- 三种模式：point-to-point（一对一）、pub/sub（一对多，经 topic）、request/reply（同步双向）。
- 选型：RabbitMQ（AMQP、路由灵活）/ Kafka（高吞吐流式、日志聚合）/ SQS（AWS 全托管）。
- 最佳实践：**消费幂等**、监控打点、失败用重试 + 死信队列 + 熔断。

## 65. 需求
- 功能：建队列（名/大小/最大消息）、发/收/删消息、删队列。
- 非功能：**durability**（收到即持久化，生产消费可独立挂）、**scalability**（三个维度：队列数、生产者数、消费者数）、可用性、性能（高吞吐低延迟）。
- 单机是 strawman：可用性和扩展性都不行 → 引出分布式。

## 66. 核心 trade-off（必考）
- **Delivery semantics**：at-most-once（快，可能丢）/ **at-least-once**（持久，可能重复）/ exactly-once（理想，最贵最难）。
- **Push vs pull**：pull 简单，且不会打爆消费者。
- **有序 vs 吞吐**：严格 FIFO 要额外同步，伤吞吐和延迟；放松有序换吞吐。

## 67. 设计 Part 1：估算
- 架构：**无状态 frontend**（横向扩展）→ **metadata service**（鉴权、取队列/用户元数据）→ backend。**元数据路径与数据路径分离**，热路径保持精简。
- 估算（背锚点）：400M 请求/天 ≈ **4630 req/s**；消息 400TB/天；元数据 500GB（4000 万队列×10KB + 2000 万用户×5KB）；缓存按**活跃集**算：40GB 用户 + 40GB 队列；frontend 单机 200K 请求/小时（≈56 req/s）→ 约 **90 台**。
- 口径：缓存大小按活跃用户/队列算，不按总量。

## 68. 设计 Part 2：后端放置模型
- **Primary–secondary**：每队列有归属 primary，internal cluster manager 管心跳、failover、把队列分区打散到多个 primary。控制面复杂，单队列归属清晰。
- **独立集群**：多集群各存自己队列全量副本，消息 ID 带 cluster 前缀，external cluster manager 只管集群健康，不管单机。隔离性好、天然多地。
- 白板要点：会画两种模型，对比 internal vs external cluster manager。

## 69. 评估：删除机制与收官
- **消息删除两种机制**：
  - **Offset tracking**（Kafka 式）：消费后消息保留，consumer 自己记 offset，后台清过期 → 支持多 consumer group。
  - **Visibility timeout**：消费后隐藏一段时间，处理完显式删除；crash 则消息重现 → **at-least-once**。
- 扩容：队列存到 ~80% 就扩；热队列要性能隔离，不连累别人。
- 收官金句：**分布式 FIFO 本质是"严格有序 vs 吞吐/延迟"的平衡术**；放松有序赢吞吐，严格有序要时间戳/因果协调；复制 + 分区换横向扩展。

## 自测 4 问（带答案要点）
1. 三种 delivery semantics 的 trade-off？
   → at-most-once 快但可能丢；at-least-once 持久但可能重复；exactly-once 理想但最贵最难。
2. 消息删除的两种机制？各对应什么语义？
   → offset tracking（Kafka 式，消费后保留、多 consumer group、后台清过期）；visibility timeout（隐藏→显式删，crash 重现 → at-least-once）。
3. 400M 请求/天、单机 200K 请求/小时，要多少 frontend？
   → 400M/86400 ≈ 4630 req/s；200K/hr ≈ 56 req/s → 约 90 台。
4. 严格有序和吞吐怎么取舍？
   → 严格 FIFO 需额外同步（时间戳/因果协调），伤吞吐延迟；放松有序换吞吐和延迟。
