# Ch20–21: Distributed Logging + Task Scheduler — 提炼笔记（93–100，2026-10-06）


## Logging

### 93. 采集三选一
- **Logs vs metrics**：metrics 看当前行为，logs 是历史记录，做 root-cause 和异常检测。
- 采集：centralized（简单，不扩展）/ decentralized（跟机器同生共死）/ **hybrid（本地缓冲 + 转发中央）→ 正解**。
- 中间加 **Kafka-like queue**：ingestion 和处理解耦 + 可 replay。
- **ES vs Hadoop**：ES 索引搜得快但贵，Hadoop 追加存便宜但查得慢 = 成本 vs 查询延迟。
- 真实参照：FB Scribe（agent → 聚合树 → 持久存储，机器挂了不丢）、Google **Dapper（1/1024 采样**成 trace 树，保真度换存储）、Twitter（Thrift 结构化日志 → Scribe → ES）。

### 94. 六组件与流水线
- 六件套：agent/collector、MQ、processing unit、storage、querying unit、user。
- 流水线：generate → collect → buffer → process → index → query。
- **结构化 vs 非结构化**：JSON 字段齐（好过滤，存得多）vs 纯文本（便宜，难解析）；常见做法是文本 + 最小 tag（时间/级别）。

### 95. 估算（链条要会现场算）
- 功能：多设备采集、按 ID/内容/级别搜、删、导出 CSV、dashboard。非功能：扩展、HA、容错、安全合规。
- 双路径：**batch 路径**（LB→collector→Kafka→ELK）+ **实时告警路径**（另起一条，低延迟）；ingestion 用 push（简单）或 pull（保护下游不被慢生产者打爆），入口加 rate limiter。
- 估算链：100 万用户、1 万台服务器、100 条/秒/台 → **约 100 万 logs/s**；1KB/条 → **1GB/s → 约 86TB/天**。
- 成本：ES 热数据（7 天 + 3 副本）约 **$8K/月**；S3 级冷存 30 天约 **$60K/月**——**按数据年龄分热冷**。
- 查询：1 万并发 × 1 QPS × 10ms CPU → 约 100 核。

## Task Scheduler

### 96. 六种调度算法（trade-off 背下来）
| 算法 | 优点 | 毛病 |
|---|---|---|
| FCFS | 公平 | convoy effect（长任务堵死后面） |
| Round Robin | 无 starvation | 上下文切换多 |
| SJF | 平均等待最优 | 要预估运行时间；饿死长任务 |
| SRTF | 平均等待最优 | 切换更多；可能 starvation |
| Priority | 紧急优先 | **aging** 修 starvation |
| **MLFQ** | 响应+公平最好 | 最复杂（面试最爱） |

- 真实参照：Linux CFS（按 vruntime 的红黑树）、FB Twine、Twitter Mesos。

### 97. 需求与估算
- 功能：任务全生命周期（提交/开始/暂停/恢复/取消）、metadata、优先级、资源感知、周期任务、时间约束、状态通知、API。
- 非功能：横向扩展、低延迟队列管理、容错（持久化+恢复+**有界重试**）、安全、可维护。
- 估算：1 亿 DAU、10% 活跃 → **1000 万任务/天**；均值 **116/s**、峰值 **231/s**；1KB/任务 → 10GB/天 → **3.65TB/年**。metadata 很便宜，难的是吞吐和调度逻辑。

### 98. 六阶段流水线
提交器（校验+限流+打 ID）→ 分布式关系 DB（持久化）→ **排队服务（临执行才入内存优先级队列）** → 调度器（按最早开始时间 + bin packing 分配）→ 执行器 → 监控器（状态/重试/通知 + 按需扩缩执行器）。
- **关键设计：提交即入库，临执行才入队**——队列只保留 hot set，不把数百万远期任务放内存。
- 依赖用 **graph DB 存 DAG**（完成后 walk 找下游）。

### 99. 队列管理
- 任务三类：urgent（立即跑）/ delayable（可等，如备份）/ periodic（周期，如日志轮转）。
- FIFO（简单，队头阻塞）vs priority heap（紧急优先，会 starvation）vs **multi-queue**（每类一队 + 上层策略，隔离性最好，多一层复杂度）。
- 静态策略（RR/固定优先级，可预测）vs 动态（MLFQ/负载感知，自适应但难推理）。
- 任务-资源匹配要考虑硬件异构和地理位置（延迟+成本）。

### 100. 评估清单（每道设计题收尾用）
可用性（全组件复制）、durability（提交即持久化）、可扩展（每层横向）、容错（**任务留队直到成功**、有界重试、超时杀死无限循环并通知用户）、**有界等待**（超 max wait 直接告诉用户重试，别无限等）。

## 自测 4 问（带答案要点）
1. 日志采集 centralized / decentralized / hybrid 怎么选？
   → hybrid：本地缓冲 + 转发中央；中间加 queue 解耦 ingestion 与处理、可 replay。
2. 1 万台服务器、每台每秒 100 条日志、1KB/条，日增量和月成本？
   → 100 万 logs/s → 1GB/s → 约 86TB/天；热数据 ES 约 $8K/月，30 天冷存 S3 约 $60K/月。
3. 调度器为什么"提交即入库、临执行才入队"？
   → 队列只保留 hot set，不把数百万远期任务放内存里。
4. 六种调度算法里，starvation 怎么修？
   → priority 加 aging；SJF/SRTF 天生饿长任务；MLFQ 多级反馈是面试最爱。
