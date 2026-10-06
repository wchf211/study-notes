# Ch18: Blob Store — 提炼笔记（80–84，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## 80. Blob store 是什么
- 存非结构化二进制（图/音/视频/二进制包），两大定义特征：**flat namespace**（container 不能嵌套）+ **immutable/WORM**（write-once read-many，更新 = 传新版本，不原地改）。
- Storage tiers + lifecycle rules：成本 vs 取回延迟的 trade-off，自动分层/过期。
- 规模感：YouTube 每天 ingest **1PB+**；多分辨率 + 多副本让实际占用远超上传字节数。
- 真实映射：Netflix→S3、YouTube→GCS、Facebook→Tectonic。

## 81. 需求与估算（YouTube 模型）
- API：建 container、put/get（系统生成 URL）、删（带 retention 窗口再真删）、list blobs/containers、删 container。
- 非功能：可用性、durability、**数十亿 blob**、吞吐、可靠性、**强一致**。
- 估算锚点：500 万 DAU、日 25 万上传、50MB 视频 + 20KB 缩略图、人均日读 20 次；I/O-bound 取保守 **500 RPS/台** → 约 **1 万台**；新增单副本单分辨率 **12.51 TB/天**；入 **1.16 Gb/s**、出 **462.96 Gb/s**——**egress 远大于 ingress**（读多），报数字时先声明 single-copy/single-resolution 假设，再说分辨率×副本×地域的乘数。

## 82. 设计：中央 manager
- 三组件：clients、frontend、storage disks。逻辑 API 7 个操作：重名用 **version 号**解决，删除 = 打标 + 异步 GC。
- **Manager node**（中央协调器）：发唯一 ID、切 chunk、定 placement、编排复制。读路径：frontend 从 manager 取 metadata → 返回 **chunk map** → **client 直读 data node**，manager 不在读热路径上；client 缓存 metadata。
- Trade-off：中央 manager 简化一切，但 **SPOF + 瓶颈**。

## 83. 设计考量（可直接复用的词汇表）
- 分层：account → container → blob，各有 ID/metadata/shard 映射。
- **Chunking**：如 128MB 切 2×64MB chunk，**3 副本/chunk**；chunk 大小 = metadata 开销 vs I/O 效率的 trade-off。
- **按完整路径分区**（account+container+blob）：同一用户的 blob co-locate；随机 ID 分区会导致 listing 瓶颈。
- 索引：用户自定义 KV tag 建可搜索索引；分页用前缀匹配 + continuation token。
- **三级复制**：同 DC（防 rack/drive 坏）/ 同 region 异 DC（防机房灾难）/ 异 region（防区域性灾难）。
- 删除 = 打标 + 异步 GC（延迟 vs 空间）；流式 = 按 byte offset 顺序 range 读。
- 缓存分层：client（chunk metadata）、frontend（partition map）、manager（热 chunk）、CDN（公开 blob + Cache-Control TTL）。

## 84. 评估
- 4 副本/blob 跨 DC/region；**集群内同步复制（强一致）**，写完后再**异步跨 region**（用区域新鲜度换写延迟）。
- Manager 瓶颈约 **10,000 QPS**，靠定期 snapshot + standby 恢复；durability 靠 peer 恢复 + 磁盘健康监控。
- 答题模式：**每个非功能需求映射到一个具体机制**，最后点名瓶颈并说怎么修（面试官会揪副本数不一致：设计章 3 副本 vs 评估章 4 副本，要能自圆其说）。

## 自测 4 问（带答案要点）
1. Blob store 的两个定义性特征？
   → flat namespace（container 不可嵌套）+ immutable/WORM（更新=传新版本）。
2. 为什么按完整路径分区，而不是随机 ID？
   → 同一用户的 blob co-locate；随机 ID 分区会导致 listing 瓶颈。
3. 中央 manager 的 trade-off？读路径怎么避开它？
   → 简化 placement/复制协调，但 SPOF + 瓶颈（约 10,000 QPS）；读时 frontend 返回 chunk map，client 直读 data node。
4. 三级复制分别防什么？一致性怎么处理？
   → 同 DC（rack/drive 故障）/ 同 region 异 DC（机房灾难）/ 异 region（区域性灾难）；集群内同步复制保强一致，跨 region 异步。
