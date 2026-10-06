# 非功能需求（新版课 6 lessons，替代老版 10–19）— 提炼笔记（2026-10-06）

> 老版课的 Backups/Heartbeats/Checksum/Replication/Partitioning/Encryption 已下线；新版课换成 6 课精简版。自己的话总结。

## Availability
- 公式：Availability% = (总时间 − 宕机) / 总时间 × 100。
- **Nines 表（背）**：99% ≈ 3.65 天/年；**99.9% ≈ 8.8 小时/年**；**99.99% ≈ 53 分钟/年**；**99.999% ≈ 5.3 分钟/年**——每多一个 9，允许宕机 ÷10。
- 口径：SLA 数字别直接比——各家时钟起点、partial outage 是否计、计划维护/攻击是否剔除都不一样；每加一个 9 成本不成比例上升（冗余、多区域、on-call）。

## Reliability
- 可靠性 = 系统在一段时间内持续正确执行功能的概率——是"行为一致"，不是"还活着"。
- 公式：**MTBF** = (总时间 − 宕机) / 故障次数（越大越好）；**MTTR** = 总修复时间 / 修复次数（越小越好）；MTTF 给不可修部件（坏盘）。
- 2×2：**高可用可配低可靠**（故障少但恢复极慢），**高可靠可配低可用**（频繁坏但秒恢复）——availability 量丢的时间，reliability 量故障频率和 blast radius，是两个独立轴；目标是双高。

## Scalability
- 垂直（加盒子，简单但有硬顶）vs 水平（加机器，弹性但要无状态+LB+数据分布）；reactive vs proactive/predictive autoscaling。
- 数据层：读先上 read replica，写或数据量顶破单主才 sharding（key-range/hash）；服务独立扩展 vs 单体简单；async 队列削峰；cache/CDN 减读负载。
- 口径：从 load 数学开场（QPS、存储、增长），小规模垂直，默认 web-scale 答案是**水平+无状态+LB**，写重上 sharding，尖峰异步用队列——每个选择都点名放弃了什么（成本 vs headroom）。

## Maintainability
- 能多便宜地被理解、修改、修、扩展——决定长期团队速度的属性。
- 抓手：模块边界+稳定接口、文档与编码规范、测试+CI/CD、可观测性（日志/指标/链路， incident 时回本）、**boring tech**（好招人好运维）vs 前沿工具、版本与向后兼容。
- Meta trade-off：今天的上线速度 vs 明天的技术债利息。口径："怎么演进？"→ 服务边界、API 契约、迁移、feature flag、运维性（部署/监控/回滚）。

## Fault Tolerance
- 在组件挂掉时仍输出正确行为——大规模下故障是日常不是意外。
- 冗余：**active-active**（瞬时接管，双倍成本）vs **active-passive**（便宜，failover 慢）；自动 vs 手动 failover；心跳调优（太密→开销+误报，太疏→发现慢）。
- **Containment 工具箱**：timeout、指数退避+jitter 重试、circuit breaker、bulkhead——防级联；优雅降级（压力下砍非关键功能）；数据持久用复制/quorum，拿一致性和延迟换；消灭单点。
- 口径结构：**detect → isolate → recover**；具体故障模式（进程崩、消息丢、网络分区）点名；冗余选型挂 RTO/RPO。

## NFR for System Design Interviews
- 面试输赢在 NFR——要**显式问、排序、设计时管理冲突**。
- 工具箱：Performance（缓存，如活跃用户预计算 feed、非活跃按需生成；空间索引给高频更新的司机位置；LB）；Availability（跨服务器/区域冗余、failover、限流抗尖峰、CDN、压测+实时监控）；Scalability（垂直/水平、autoscale、sharding、模块化服务、缓存+CDN）。
- 现成映射：Google Maps——路网图按 segment 切到多台复制服务器后挂 LB（可用），随查询涨加 segment/主机（扩展）；YouTube——ISP/CDN + 多层缓存 + 按用途存（blob 存视频、wide-column 存缩略图、轻服务器）降延迟，sharding+replication+heartbeat 买可靠。
- 罐装映射（背）：transactions → ACID 关系库；超大规模 → NoSQL；实时流 → Kafka/Kinesis。"no one-size-fits-all"——澄清、排序、论证 trade-off 才是得分点。

## 自测 4 问（带答案要点）
1. 99.9/99.99/99.999 各对应年宕机？
   → 8.8 小时 / 53 分钟 / 5.3 分钟；每多一个 9，允许宕机 ÷10。
2. 可用性高但可靠性低，可能吗？举例。
   → 可能：故障极少但一次坏掉恢复三天（高可用低可靠）；或频繁坏但秒恢复（高可靠低可用）。是两个独立轴。
3. 读重系统先上 replica，什么时候上 sharding？
   → 读先上 read replica；写压力或数据量顶破单主时才 sharding（key-range/hash）。
4. 容错回答的结构和工具箱？
   → detect → isolate → recover；timeout、退避+jitter 重试、circuit breaker、bulkhead、优雅降级、active-active vs active-passive。
