# TinyURL + Web Crawler + WhatsApp — 提炼笔记（153–159，2026-10-06）

> 只留关键、长远有用的点。自己的话总结，非原文搬运。

## TinyURL（153）
- 长链 → 短别名；**写读比 1:100**（读多）。
- **Sequencer** 发 64 位 ID → **Base-58** 编码（去掉 0/O、I/l，易读）；5.85 bits/字符 → 64 位约 **11 字符**；从 10 亿起发保证 ≥6 字符。
- 自定义别名：把别名解回十进制 ID，在 DB 标"已用"。
- MongoDB + 主从复制；Memcached 缓热点；按 api_dev_key 的 fixed-window 限流。
- 估算：月 2 亿新 URL、500B/条、5 年 retention → 120 亿条 × 500B = **6TB**；写 76/s、读 7.6K/s；入 304Kbps、出 30.4Mbps；缓存按 80/20 对日 6.6 亿重定向 → **66GB**；约 1600 台。
- **安全**：顺序 ID 可猜、可遍历 → 从未用 ID 池**随机取**，保证不可预测。
- API：shortURL、redirectURL、deleteURL（可自定义别名+过期，默认 5 年）。

## Web Crawler（154）
- 循环：fetch → process → store。Crawl manager（URL 来源 + frontier）、fetcher、document processor。
- **Frontier**：FIFO + **priority queue**，后面是 **politeness scheduler**（每域名/host 一个队列）→ **按 hostname consistent hashing** 分给 fetcher 线程，限单域抓取速率。
- 抓之前三查：**robots.txt 缓存**（经典坑）、DNS 缓存、URL seen-hash。抓后：去文本 → **去重**（先 URL hash，再内容指纹）→ parser（标题/正文/链接）→ 唯一进索引，新链接回 frontier。
- 存储按多 hash ring 分片（冗余+并行）；**TTL + priority queue 保新鲜度**；checkpoint + 复制做容错。
- 估算：月 10 亿页、500KB/页 → **500TB/月**；约 185Mbps；单 fetcher 15 页/秒 → 约 **25 台**。
- Trade-off：并行度 vs politeness；新鲜度 vs 重爬成本；去重分两段（URL 级早，内容级后）。

## WhatsApp（155–159）
- 规模：20 亿+用户、日 **1000 亿**消息；"55 个工程师"故事讲的是**单机连接效率**，不是人效。
- **Ack 生命周期是骨架**：sent → delivered → read，8 步交换（发→服务端 ack→落盘→投递→已送达 ack→已读→已读 ack）。
- 功能：1:1+群聊、ack、媒体、离线排队、push。非功能：低延迟、**严格有序**（多设备一致）、可用性权衡、**端到端加密**、可扩展。
- 估算：100B/条 → **10TB/天**（30 天 300TB）；出入对称约 **926Mbps**；**1000 万连接/台** → 200 台 chat server。
- 媒体走旁路：传 blob，只把 media ID 发给对方，对方经 **CDN** 下——大文件不上 chat server。
- 组件：connection manager（用户在哪台上）、message handler、**message sequencer**（给 1:1 分配顺序 ID 保有序）、group handler（**服务端按成员 fan-out**）、user/group service。
- 数据：user DB、group DB、message DB 用 **Mnesia**（内存分布式，实时有序队列）；Redis cluster 缓存元数据/头像。
- 评估：**CP over AP**——分区时保序优先，选一致性；延迟 vs 安全——接受端到端加密的 CPU 开销（大媒体在端上加解密最明显），安全优先。

## 自测 4 问（带答案要点）
1. TinyURL 为什么用 Base-58 不用 Base-64？64 位 ID 要几个字符？
   → 去掉易混淆字符（0/O、I/l）保可读；5.85 bits/字符 → 约 11 字符。
2. 顺序 ID 有什么安全问题？怎么修？
   → 可猜测、可遍历；从未用 ID 池随机取，保证不可预测。
3. 爬虫 frontier 为什么要有 priority queue？politeness 怎么实现？
   → 新鲜度 vs 重爬成本；每域名/host 一个队列 + 按 hostname consistent hashing 分给 fetcher 线程，限单域速率。
4. WhatsApp 群聊 fan-out 在哪做？1:1 有序靠什么？
   → 服务端 group handler 按成员 fan-out；message sequencer 给 1:1 分配顺序 ID。
