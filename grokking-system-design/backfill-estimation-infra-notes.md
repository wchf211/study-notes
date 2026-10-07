# 估算 + DNS/LB/DB/KV 回补（新版课 Ch5,7,8,9,10）— 提炼笔记（2026-10-06）

> 老版课 21–38 已下线；这是新版课最对等的 17 lessons。只留数字、公式、trade-off。

## 估算：背公式
- 参考量级：Facebook \~600 万赞/小时；Twitter \~6500 推/s；YouTube \~400 小时视频/分钟上传；Instagram \~1 亿照片/天（\~35PB 总量）；Google \~35 亿搜索/天（\~20PB/天处理）；WhatsApp \~650 亿消息/天；典型大 web app 基线：10 万 DAU、1 万 req/s。
- **Jeff Dean 延迟表（背）**：L1 0.5ns；L2 7ns；内存 100ns；1KB 经 1Gbps \~10,000ns；SSD 读 4KB \~150,000ns；磁盘 seek 10ms；CA→荷兰→CA 包 \~150ms。
- 2 的幂速查：2^10≈1K、2^20≈1M、2^30≈1B、2^40≈1T；天↔秒 ÷86400（≈10^5）。
- **五步 recipe**（背顺序）：(1) 显式假设（用户、读写比、retention）→ (2) 流量 → (3) 存储 → (4) 带宽 → (5) 服务器数。
- TinyURL 例：月 2000 万新 URL、读写比 100:1、5 年 retention、500B/条。
  - 流量：2000万/30/86400 ≈ **8 写/s**，读 ≈ **800/s**。
  - 存储：1.2B 对象 × 500B = 600GB → **715GB**（+20% margin）。
  - 带宽：入 8×500B ≈ 4KB/s；出 800×500B ≈ 400KB/s。
  - 服务器：内存和带宽**分别算取上限**（memory-bound vs bandwidth-bound）。
- 口径：日均 ×2–5 规划峰值；读写差量级时**独立估算**；先报假设再算数。

## DNS
- 分层分布式数据库；同时是**粗粒度全局 LB**（一域名多 A 记录）。
- 记录类型：A（v4）、AAAA（v6）、CNAME（别名）、MX（邮件）、NS、TXT、SOA、PTR（反查）。
- 解析：root → TLD → authoritative；迭代 vs 递归；browser/OS/resolver 多层缓存，**TTL 定一切**。
- 关键数字：UDP 53 端口（快），大包/zone transfer 切 TCP；root 约 1000 个 anycast 实例。
- Trade-off：**短 TTL** = 新鲜+快速 failover，但权威服务器查询压力大；**长 TTL** = 轻负载，但故障/迁移时记录过期——DNS LB 的 failover 慢且无健康感知。

## Load Balancer
- 位置：**每个无状态 tier 前都放 LB**；算法：round robin、加权、least connections、least response time、hash（IP/URL）、random；健康检查剔除死节点；**sticky session** 只在 tier 有本地状态时用（且要说清为什么不是无状态）。
- 类型：硬件（F5，吞吐高、贵、不灵活）/ 软件（HAProxy/NGINX，便宜灵活）/ 云托管（ELB/ALB，零运维、vendor lock-in、规模大时贵）。
- GSLB：DNS-based（简单，受 TTL 缓存拖累 failover 慢）vs **anycast**（BGP 同 IP 多站点，路由级快速 failover，但运维复杂、流量突变要容量规划）。多区域设计：顶层 GSLB（anycast/DNS）+ 每区域本地 LB。
- 深度细节（说 1–2 个显资深）：**connection draining**（摘节点前让在途请求跑完）、**TLS termination**（LB 卸 crypto 负载；内网明文可接受，公网不行）、**ECMP**（三层线速分流）、L4 vs L7（L7 能按 path/host 路由但 CPU 贵）、consistent hashing 防重排。

## Databases
- SQL（ACID、join、垂直优先）vs NoSQL 四家：**key-value**（Redis/Dynamo，O(1) key 查）/ **document**（Mongo，schema 灵活）/ **wide-column**（Cassandra/HBase，巨型稀疏表）/ **graph**（Neo4j，关系遍历）。
- 选库口径：**永远带 access-pattern 理由**——transactions → ACID 关系库；超大规模 → NoSQL；实时流 → Kafka/Kinesis。
- **复制**：单 leader（写单点，同步/异步跟）/ 多 leader（冲突解决）/ 无 leader（Dynamo 式，客户端写 N 个副本，quorum 决）。
- **Quorum 公式（背）**：**W + R > N** 保证读到最新写；**W > N/2** 写 quorum；经典 **N=3, R=2, W=2**；读快 W=1,R=N；写快 R=1,W=N；永不丢写 W=N；最大可用 W=1,R=1（纯 eventual）。Sync = 一致性换延迟/可用；async = 快但有复制 lag。
- **分区**：range（范围扫描好，热点）、hash/modulo（均匀，无范围扫）、**consistent hashing**（成员变时重排最小）、directory（多一跳）、geo。分区键是最高杠杆决策——坏键（如时序写用 timestamp）造 celebrity 热点；跨分区 join/事务变难，复杂度推给应用。
- **量化口径**：10K QPS、10 shard：全 fan-out → 每 shard 10K（贵）；按时间 range → 最新 shard 吃 10K（热点）；按 user hash → 每查询只打 1 shard（每 shard 1K）。**量化每个方案的每 shard 负载**再下结论。

## Key-Value Store（Dynamo 式）
- 场景：购物车、session、配置——**按 key 访问、规模/可用优先**。
- 一致性哈希环（MD5 128-bit）；**virtual nodes**（物理节点占多个虚拟 token，异构机器按比例分负载，加减节点只扰动小范围）。
- 复制：key 存到顺时针 N 个后继节点（例 N=5），coordinator 负责写并转发 preference list。**AP 选型**（可用+分区容忍，放弃强一致）。
- **Vector clock**：每写一版本，并发写变 siblings，读时 reconcile（read repair）；LWW 是简单替代（默默丢并发写）。
- 故障三件套（配对背）：
  - **Transient** → **sloppy quorum + hinted handoff**（N 个健康节点先接，带 hint 等原节点恢复再转交）；
  - **长期分歧** → **Merkle tree** anti-entropy（按 key-range 建 hash 树，比根找差异子树，只传不一致数据；重算成本是代价）；
  - **检测** → **gossip**（随机交换成员信息，最终一致；超时判死；分区时会误判）。
- 别在每次瞬时故障都 rebalance——多数故障很短；hinted handoff 挡不住灾难性故障，要跨数据中心复制。

## 自测 4 问（带答案要点）
1. TinyURL 估算链？
   → 日 8 写/s、800 读/s；1.2B 对象 600GB→715GB；入 4KB/s、出 400KB/s；服务器按内存/带宽分别算取上限。
2. Quorum 公式和三个 canonical 调参？
   → W+R>N 读到最新写；N=3/R=2/W=2 强一致；读快 W=1,R=N；写快 R=1,W=N；永不丢写 W=N。
3. 故障三件套各应对什么？
   → transient → hinted handoff；长期分歧 → Merkle anti-entropy；检测 → gossip。
4. Sharding 量化：10K QPS、10 shard，三种方案？
   → 全 fan-out 每 shard 10K；时间 range 最新 shard 吃 10K 热点；按 user hash 每 shard 1K。
