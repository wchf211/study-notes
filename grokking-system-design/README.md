# 系统设计面试精华笔记

Grokking (Modern) System Design Interview 全课提炼 —— 自己的话总结（非原文搬运），每章带 4 道自测题（含答案要点）。整理于 2026-10-06。

> 左侧目录点着看。建议面试前按这个顺序过一遍：估算 → 非功能需求 → RESHADED 答题骨架 → 1～2 个大题深挖。

## 一、基础与方法论

- [估算 + DNS / LB / DB / KV](grokking-system-design/backfill-estimation-infra-notes.md) —— 五步估算法、Nines 表、Quorum 公式 W+R>N、Jeff Dean 延迟表
- [非功能需求](grokking-system-design/nfr-modern-notes.md) —— Availability / Reliability / Scalability / Maintainability / Fault Tolerance
- [RESHADED 答题骨架](grokking-system-design/ch22-shardedcounters-reshaded-notes.md) —— 面试回答的标准顺序
- [故障复盘](grokking-system-design/system-failures-notes.md) —— Facebook 2021、Kinesis 2020、AWS 全区域宕机

## 二、Building Blocks

- [Sequencer](grokking-system-design/ch12-sequencer-notes.md) —— Snowflake、Lamport vs vector clocks、TrueTime
- [Monitoring](grokking-system-design/ch13-monitoring-notes.md) —— pull vs push、TSDB
- [Cache](grokking-system-design/ch14-cache-notes.md) —— write-through/back/around、Memcached vs Redis
- [MQ](grokking-system-design/ch15-mq-notes.md) —— delivery 语义、offset vs visibility timeout
- [Pub-Sub](grokking-system-design/ch16-pubsub-notes.md) —— partitioned log、retention
- [Rate Limiter](grokking-system-design/ch17-ratelimiter-notes.md) —— 5 种算法、Redis 计数器
- [Blob Store](grokking-system-design/ch18-blobstore-notes.md) —— flat namespace、chunking、三级复制
- [Search](grokking-system-design/ch19-search-notes.md) —— 倒排索引、term vs document sharding
- [Logging + Scheduler](grokking-system-design/ch20-21-logging-scheduler-notes.md) —— 86TB/天、6 种调度算法

## 三、经典大题

- [YouTube](grokking-system-design/youtube-notes.md) —— Vitess、per-shot 编码、ABR、去重
- [Quora](grokking-system-design/quora-notes.md) —— polyglot 存储、一致性分级
- [Google Maps](grokking-system-design/googlemaps-notes.md) —— 两阶段路由、图切分
- [Yelp](grokking-system-design/yelp-notes.md) —— QuadTree vs 静态网格
- [Uber](grokking-system-design/uber-notes.md) —— 两级位置新鲜度、风控
- [Twitter](grokking-system-design/twitter-notes.md) —— 1:1000 读写比、fanout
- [Newsfeed + Instagram](grokking-system-design/newsfeed-instagram-notes.md) —— pull/push/hybrid fanout
- [TinyURL + Crawler + WhatsApp](grokking-system-design/tinyurl-crawler-whatsapp-notes.md) —— Base-58、ack 生命周期
- [Typeahead](grokking-system-design/typeahead-notes.md) —— trie、离线重建
- [Google Docs](grokking-system-design/googledocs-notes.md) —— OT vs CRDT
- [Code Deploy + Payment + LeetCode](grokking-system-design/deploy-payment-leetcode-notes.md) —— 蓝绿/金丝雀、幂等键、双队列 fan-out

## 四、AI 时代

- [AI 重塑与 ChatGPT](grokking-system-design/ai-era-1-notes.md) —— 语义缓存、TTFT、三桶分类
- [数据基建 / RAG bot / 代码助手](grokking-system-design/ai-era-2-notes.md) —— training-serving skew、RAG、TTFT<300ms

## 附

- [课程大纲](grokking-system-design/course-outline.md)
