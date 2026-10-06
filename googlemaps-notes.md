# Google Maps — 提炼笔记（121–126，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 121. 开场：按用户问题框题
- 三个用户问题：最优路线 / 每条路多远 / 每条路多久。
- 多面使用者：消费者、企业（500 万+商家用 Maps API）、物流——规模驱动力。
- Building blocks 先点名：搜索、KV、pub-sub、图存储。路网建模为图是前置决策。

## 122. 需求与估算
- 功能：定位（lat/long）、路线推荐（距离/时间/交通方式）、导航（分步指引）。路网 = 图（路口是点，路段是边）。
- 非功能：HA、可扩展、**2–3 秒**内算出路线和 ETA、**ETA 准**。
- 估算：3200 万 DAU 当 RPS 代理 → 64K RPS/台 → **500 台**（声明假设比数字重要）；路网数据 **20+ PB**（基本静态，日更新可忽略）；50 请求/人/天 × 200B → 18518 RPS → 入 **29.6 Mb/s**；回包 2MB 图块 + 5KB 文本 → 出 **297 Gb/s**——**图块主导 egress**。

## 123. 设计：两阶段路由
- 组件：location finder、route finder、**navigator**（实时跟踪+偏航重算）、distributed search（地名→坐标）、**area search service**（圈定起终点之间的地图区域）、graph processing service（在子图上跑最短路）、graph DB、Kafka（偏航事件）、第三方路网数据+建图服务、LB。
- 链路：定位 → 目的地 typeahead → area search 圈定区域 → 图处理跑最短路 → 返回 → navigator 流式播报 → 偏航 → Kafka 事件 → 重算。
- API：currLocation（长连接实时更新）、findRoute（transport_type 可选，默认开车）、directions（转向提醒）。
- **招牌动作：两阶段路由**——先 area search 把问题收敛到有界子图，再跑最短路。Trade-off：框要够大包住真实最优路线（高速绕行可能在紧框之外），又要够紧省算力。

## 124. 核心挑战：分段 + ETA
- **图分段**：全球路网切成 segments（如一座城切几百个 5×5 英里块）；**段内最短路离线预计算 + 缓存**，重算力不上实时路径。
- 用户一般在**边**上不在顶点：snap 到最近顶点（比到边两端点的距离）。
- **跨段**：exit points（共享边界顶点），每段预计算 exit↔exit 最短路，长路线在 exit-point 图上拼接——**hierarchical routing**（只有起终点需要局部细节）。
- 用 **haversine** 算空中距离圈定搜索范围（如 10km 半径）。
- **ETA**：pub-sub 收实时位置流（userID/时间/坐标）→ analytics 得路况密度（高/中/低）、均速、周期规律（如工作日 8–10 点高速堵）→ 调整静态基线。

## 125. 详细设计：三库 + 实时管线
- 存储三分：**KV**（segment 元数据：segmentID、serverID、边界坐标、邻段）/ **graph DB**（路网）/ **RDBMS**（历史路况：edgeID、hourRange、rush 标记）；ID 来自 Sequencer。
- 加段流程：segment adder 拿 ID → allocator 选主机部署图 → KV 写映射。服务请求：坐标→段→查 KV 找服务器→段内查或跨段拼接。
- **实时管线**：WebSocket 传 GPS → LB（注意单机连接数上限）→ pub-sub → **Spark ML** 做路况预测/热点/新路发现 → map update service 按 KV 找到属主服务器打补丁。**更新要聚合节流**：红灯这种瞬时波动不值得重算图，只推持久有意义的变化。

## 126. 评估：需求→机制映射表
- Availability ← 小图查询 + LB + 复制；Scalability ← 图分区 + 分布式部署；Latency ← 缓存子图 + KV；Accuracy ← 实时数据 + analytics。
- 优化项：**非均匀分段**（市区小、郊区大）、lazy loading 地图数据。

## 自测 4 问（带答案要点）
1. 整图跑最短路太慢，标准解法？
   → 图分段，段内最短路离线预计算+缓存；跨段用 exit points 拼接（hierarchical routing）；haversine 圈定搜索范围。
2. 用户一般在边上不在顶点，怎么处理？
   → snap 到最近顶点（比较到边两端点的距离）。
3. ETA 怎么算准？
   → 静态基线 + 实时流：pub-sub 收位置流 → analytics 得路况密度/均速/周期规律 → 调整基线。
4. 两阶段路由的 bounding box 有什么 trade-off？
   → 框要够大包住真实最优（高速绕行可能在框外），又要够紧省算力。
