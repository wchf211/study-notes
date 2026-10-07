# Lessons from System Failures（Module 47，2026-10-06）

> 老版课 193–197 已下线；这是新版课最对等的替代（4 lessons）。自己的话总结。

## 1. Lessons from System Failures：五大韧性原则
- 故障两大根因：需求多变逼系统频繁更新（更新带来不稳定）；复杂系统的涌现行为（整体比局部难推理）。
- 四类故障：**system**（进程崩，内存丢，盘/副本存活，重启恢复）/ **method**（执行错或死锁，操作暂停）/ **communication-medium**（网络层）/ **secondary-storage**（副本挂，主节点重建副本）。
- 五大原则：**优雅降级**（只影响小部分用户、短时间）；**独立监控视角**（监控和服务不能同命）；**独立状态通道**（状态页搭在同一基建上会一起挂，用外部渠道）；**failure domains / blast radius**（隔离故障传播）；**第三方检测**（Downdetector，客观众包）。

## 2. Facebook 2021 宕机：自动化级联
- 链：例行维护命令带 fault → 所有数据中心断 backbone；审计工具没拦下；DNS 作为 fail-safe **撤了 BGP 路由**（本地"安全"决策，全球把 facebook.com 从互联网摘掉）；公共 resolver 刷新不了 → 域名不可解析。**6 小时，\~1 亿美元**，市值蒸发数十亿。
- 教训：级联故障；恢复慢（远程工具断了，工程师要物理进机房）；**自动化放大坏命令**（快+大规模）；DNS fate-sharing（独立 DNS 供应商 = 成本 vs 韧性）；安全 vs 恢复（物理安保拖慢修复）；**retry storm**（客户端狂刷 DNS 火上浇油）；冷启动（全重启时的电力/流量 surge）。

## 3. AWS Kinesis 2020 宕机：隐藏的资源上限
- 2020-11-25 US-East-1，凌晨 2:44–3:47 扩容：新服务器超了 **OS 最大线程数** → cache 构建失败 → shard map 废掉 → 前端路由瘫痪。
- 级联：Cognito 有 latent buffering bug 卡住鉴权；CloudWatch 挂掉（指标全变 INSUFFICIENT DATA）；AutoScaling 延迟；Lambda 内存拥塞；**连 Health Dashboard 都挂了（因为它依赖 Cognito）**。Adobe、Coinbase 等第三方被波及。
- Amazon 修复：**更少但更强的服务器**（减少跨舰队线程数）；前端改架构（冷启动优化、cache 独立舰队）；**分区前排 fleet** 隔离关键服务（CloudWatch 不被拖下水）。
- 口径：**隐藏资源上限**（线程数、fd）是面试里常被忽略的深度点；控制面共享依赖（auth、监控）是 fate-sharing 重灾区。

## 4. AWS 2021 全区域宕机：控制面 vs 数据面
- 2021-12-7，**8+ 小时**：内网自动扩容 → 客户端 surge 压垮连接内网与主网的网络设备 → **retry storm**（激进重试自我喂养）→ 拥塞切断了**实时监控（运维变瞎）**→ 人工挪走内网 DNS 流量才恢复。
- 关键区分：**数据面存活、控制面瘫痪**——EC2 实例照跑，但 EC2 API 挂了，RDS/EMR/Workspace 开不出新实例；LB 存量可用但新建不了。
- 教训：验证隔离（号称独立的内网有隐藏依赖）；black-swan 应急预案；运维训练（没监控也要能动手）；多区域/多云（贵）；对网络设备和 retry storm 做极限压测。引 Lamport："一个你根本不知道存在的机器的故障，可以让你自己的机器瘫痪。"
- 口径：设计里永远讲 **backoff+jitter + circuit breaker** 防 retry storm；监控/管理通道与数据通道隔离。

## 自测 4 问（带答案要点）
1. 四类故障分别是什么？
   → system（进程崩，内存丢，盘/副本存活）/ method（执行错或死锁）/ communication-medium（网络）/ secondary-storage（副本挂，主重建）。
2. Facebook 2021 宕机的级联链？
   → 维护命令故障断 backbone → 审计工具没拦 → DNS 撤 BGP（本地"安全"决策）→ 域名全球不可解析；6 小时、\~1 亿美元。
3. Kinesis 宕机根因和隐藏上限？
   → 新机超 OS 最大线程数 → cache 构建失败 → shard map 废掉；隐藏资源上限（线程/fd）是关键。
4. Control plane vs data plane 故障区别？
   → AWS 12.7：数据面存活（EC2 运行），控制面瘫痪（API 挂，新实例开不出）；监控/管理通道和数据通道隔离，状态页不能和主服务同命。
