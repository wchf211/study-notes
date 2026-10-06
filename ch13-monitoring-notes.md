# Ch 13: Distributed Monitoring — 提炼笔记（49–56，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 49. 问题陈述（可跳过）
- 监控 = 分布式 infra 的可见性层：采集、解读、展示进程交互数据，提前发现趋势和告警信号，保 SLA。
- 答题骨架先摆出来：server-side + client-side 两半，再往下走。

## 50. 为什么需要监控
- 单点故障会级联：例子——视频上传链路里复制服务挂了，写继续进 X 库，X 再挂时读切到过期 Y 库 → "Video not found"。
- 两类故障：**server-side**（5xx，监控看得见）vs **client-side**（4xx/网络问题，请求没到服务器时服务端日志完全看不见）。
- 面试谈资数字：2021.10 社交巨头宕机 ~9 小时 × 约 $13M/小时；2021.12.7 AWS 宕机——自动扩容引发 client retry storm 打垮网络设备，约 $66,240/分钟。

## 51. 前置知识：metrics 与告警
- **Pull vs push（必考决策）**：pull = 监控方按自己节奏 scrape，流量可控，默认选它；push = 被防火墙挡住 inbound 时才用（server 主动上报）。
- 持久化：规模上去用 **TSDB**（时序库：按时间存、可做趋势分析与事件关联）；应用指标靠**代码插桩**（OS 看不到的业务信号）。
- 告警 = 条件 + 动作（如 CPU > 90% → 电话告警），替代拍脑袋运维。

## 52. 需求与高层设计
- 信号清单（能脱口而出显 senior）：进程 crash、CPU/内存/磁盘/网络异常、load average、硬件故障（内存/磁盘）、关键外部服务连通性、交换机/LB 状态、功耗与电力事件、DNS/路由、DC 内外延迟、peering 点、全局服务健康（如 CDN 表现）。
- 三件套架构：**TSDB（存指标）+ data collector（采集入库）+ querying API（给可视化与告警查）**；metric 数据块用 blob storage。

## 53. 详细设计与扩展
- 存储三件套：TSDB（热数据，读写快）+ blob storage（长期保留）+ rules DB（告警规则：条件→动作）。
- Collector：pull 模式，从各 DC 经消息队列读应用日志（DigitalOcean 用 pull 监控全球数百万机器）；**service discoverer**（对接 EC2/K8s/Consul）让新实例自动被发现；alert manager 按 rules DB 评估后发邮件/Slack；dashboard 看大盘。
- 单监控服务器的毛病：**单点** + 扩展不上 + 高精度数据存不起（要 retention/downsampling 策略）。
- 扩展解法：**分层 hybrid pull/push**——secondary 在各 cluster 本地 pull（约 5000 节点/台），聚合后 push 到 primary，再到 global；配 blob storage + Elasticsearch + visualizer。记住锚点数字：**~5000 nodes per secondary**。

## 54. 可视化：heat map
- 百万台机器一眼看出谁挂了：按 DC→cluster→row 排序的二维颜色矩阵，绿正常、红失联。
- 招牌估算：**每台 1 bit（0/1 表健康），100 万台 = 1M bits ≈ 125 KB**——面试官最爱这种 back-of-the-envelope。

## 55. Client-side 错误为什么难
- DNS 解析失败、路由故障、第三方 middlebox/CDN 挂了——**根本不经过你的服务器**，服务端日志是瞎的；流量下跌也不可靠（自然波动、误报、小范围影响看不见）。
- 谈资：BGP route leak——某 AS 误宣告 3 万+前缀，入向流量暴涨 13 倍，故障方自己毫无察觉。

## 56. Client-side 监控设计
- Prober（全球布探针发合成请求）被否：互联网 10 万+ AS，全覆盖不现实；合成流量不代表真实用户。
- 正解：**客户端嵌 agent，上报到独立 collector**；collector 必须和主服务**不在同一个 failure domain**（不同 IP、不同域名、不同 AS，防 AS 劫持）；最后一公里断网时 agent 本地缓冲，恢复后补报。
- 三个加分点：**采样**（如只收 1% 用户，控成本）+ **隐私最小化**（只报成功请求的 server weblog 里本来就有的东西，不收 traceroute/DNS resolver 等定位信息，端到端加密防篡改）+ 分层聚合 + 流处理做近实时分析。
- Opt-in：用户同意后用自定义 HTTP header 约定上报策略与 endpoint。

## 自测 4 问（带答案要点）
1. 监控数据采集 pull vs push 怎么选？
   → 默认 pull（监控方控节奏、防打爆）；防火墙挡 inbound 时用 push。
2. 监控存储三件套各存什么？
   → TSDB 存热指标（快读快写）；blob storage 存长期数据；rules DB 存告警规则（条件→动作）。
3. 单监控服务器撑不住百万机器时，怎么扩展？锚点数字？
   → 分层 hybrid：secondary 各 cluster 本地 pull（约 5000 节点/台），聚合 push 到 primary 再到 global。
4. 为什么需要 client-side monitoring？collector 部署的关键要求是什么？
   → DNS/路由/CDN 类故障不经过服务器，服务端日志看不见；collector 必须在独立 failure domain（不同 IP/域名/AS）+ 采样控成本 + 隐私最小化。
