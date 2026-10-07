# YouTube — 提炼笔记（109–113，2026-10-06）


## 109. 开场
- 规模锚点：约 **25 亿 MAU**、每分钟 **69.4 万小时**播放、每分钟 **500 小时**上传。
- 口径：任何流媒体题先分两条 workload——**上传/处理 vs 播出**，再报规模数字。

## 110. 需求与估算
- 功能：上传、播放、搜标题、赞/踩、评论、缩略图。
- 非功能：HA 99%+、可扩展、低延迟播放、durability；**一致性分档**：播放量/推荐可最终一致，**上传内容绝不能丢**。
- 估算（假设：15 亿用户、5 亿 DAU、平均 5 分钟、压缩后 6MB/分钟）：
  - 存储：500 小时/分钟 × 60 × 6MB ≈ **180GB/分钟**基线（再 ×分辨率档数，实际数倍）。
  - 带宽：upload:view = **1:300**；上传约 480Gbps，播出按比例放大（大头）。
  - 服务器：峰值按 DAU 代理 → 64K RPS/台 → 约 **8000 台**（展示线性关系即可）。
- Building blocks：DB（元数据）、blob（视频）、CDN、LB、应用服务器、encoders。

## 111. 设计：pipeline
- 主链路：**上传 → encoders（多分辨率转码）→ blob 存储**；元数据进 DB/Bigtable；热门推 CDN/colo；用户从最近源播。
- REST API：uploadVideo、streamVideo（带 **resolution/bitrate/device chipset** 参数做自适应）、searchVideo、viewThumbnails、likeDislike、commentVideo。streamVideo 异步；视频切 sequential packets，服务端暂存部分状态支持续播。
- 组件：全局+本地 LB → web（轻量静态）→ 应用服务器（业务逻辑+多层缓存）→ 用户/元数据存储解耦；缩略图元数据放 Bigtable（低延迟 KV），缩略图本身放 blob。
- Trade-off：长上传/转码用异步 API；服务端留 chunk 状态换可续播（花内存）；**CDN 放热、colo 放温、origin 放冷**（成本 vs 延迟分层）。

## 112. 评估：Vitess 是分水岭
- MySQL 是扩展瓶颈：分片增加应用复杂度、威胁 ACID、反范式化伤写。
- **Vitess**：中间件把分片 MySQL 集群包装成**逻辑单库**，处理分片和查询路由，**保持 MySQL 兼容 + ACID**——关系型规模化问题的标准答案。
- Trade-off：feed/计数最终一致够用，用户数据强一致（解耦分开）；**public vs private CDN**：低流量地区用公有 CDN 省 CAPEX、扛病毒式突发；规模大了自建更划算、可内部优化。
- 去重：假设 500 小时/分钟里 50 小时是重复 × 6MB/分钟 = 18GB/分钟 → 年省 **约 9.5PB**；检测用 LSH/block matching/ML（顺带做版权）。

## 113. 现实更复杂：编码→部署→播出
- **Encode smarter**：per-shot（按段自适应编码）——按内容复杂度给每段不同压缩率（动态炫技画面给足码率，简单画面压狠），比整片固定 N 档**文件更小**；音频也多格式（Netflix 多语言）；小文件 → 部署和播出都省带宽、CDN 命中率更高。
- **Deploy closer**：热门 chunk 进 CDN/ISP PoP；谈不下 ISP 就用 IXP（还能回填 ISP 缓存）；origin 分层：**flash 服务器（热/温，低延迟）+ storage 服务器（冷，海量）**；非高峰向 ISP 推内容。
- **Deliver adaptively**：**ABR**——客户端测带宽，动态切换 segment 清晰度；取决于端到端带宽、设备能力、编码方式、客户端 buffer。
- 推荐两阶段：candidate generation（百万→数百，看用户历史/上下文）→ ranking（特征→数十），两阶段都用 ML。
- 收尾口径：encode→deploy→deliver 三段论，讲的是**最终用户体验**，不止组件。

## 自测 4 问（带答案要点）
1. YouTube 上传和播放两条链路分别怎么走？
   → 上传：web→app→storage→encode→缩略图/元数据；播放：就近（CDN/colo/origin）+ ABR 自适应。
2. 500 小时/分钟上传、压缩后 6MB/分钟，存储和带宽怎么估？
   → 500×60×6MB ≈ 180GB/分钟基线（×分辨率档数）；upload:view 1:300，上传约 480Gbps，播出按比例放大。
3. MySQL 分片后 ACID 怎么办？
   → Vitess：中间件把分片 MySQL 包装成逻辑单库，保持 MySQL 兼容和 ACID。
4. Per-shot encoding 解决什么问题？
   → 按内容复杂度分段自适应压缩，比整片固定 N 档文件更小；小文件省带宽、CDN 命中率更高。
