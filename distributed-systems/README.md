# Distributed Systems for Practitioners — 笔记（Parts 1–2）

31 modules，167 项，\~9.5h。已读完 Modules 1–20（累计 115 teaching lessons），Modules 21–31 仍在读，读完会续 Part 9。跳过的练习类：quizzes + AI exercise（各 Part 内）。风格：SD 类保持精简 —— 有背景的读者提概念就能想起来，细节花在课程独有的直觉、实现坑、trade-off 上。每篇带 4 道自测题。

- [Course Outline（全课地图 + 续读点）](distributed-systems/ds-00-outline.md) —— 31 modules 全清单 + 停在哪、从哪续
- [Foundations](distributed-systems/ds-01-foundations.md) —— Modules 1–2：为什么分布、八大谬误、部分失败、safety/liveness、系统模型、失败分类、exactly-once、无状态/有状态
- [Basic Concepts + Theorems](distributed-systems/ds-02-concepts-theorems.md) —— Module 3：partitioning、replication、quorum、safety、ACID、CAP、consistency、isolation、模型层级
- [Teaching Lens](distributed-systems/ds-03-teaching.md) —— Modules 1–3 的讲解组织方式 + 值得偷的讲法
- [Distributed Transactions](distributed-systems/ds-04-transactions.md) —— Modules 4–7：serializability、PCC/OCC/MVCC/SSI、2PC/3PC/quorum commit、组合 + Saga
- [Consensus](distributed-systems/ds-05-consensus.md) —— Module 8：FLP、Paxos、Multi-Paxos、RSM、经共识的事务、Raft、用成熟实现
- [Teaching Lens Part 2](distributed-systems/ds-06-teaching-part2.md) —— Modules 4–8 的讲解组织方式 + 值得偷的讲法
- [Time + Order](distributed-systems/ds-07-time-order.md) —— Modules 9–10：物理 vs 逻辑时间、因果、Lamport / vector / version / dotted、快照 + Chandy-Lamport
- [Teaching Lens Part 3](distributed-systems/ds-08-teaching-part3.md) —— Modules 9–10 的讲解组织方式 + 值得偷的讲法
- [Networking + Security](distributed-systems/ds-09-networking-security.md) —— Modules 11–12：分层 + end-to-end、认证/机密/完整、对称/非对称/签名
- [Teaching Lens Part 4](distributed-systems/ds-10-teaching-part4.md) —— Modules 11–12 的讲解组织方式 + 值得偷的讲法
- [Security Protocols + GFS](distributed-systems/ds-11-security-protocols-case1.md) —— Modules 13–15：TLS / PKI / Web of Trust / OAuth、桥接、GFS + HDFS + GFS 一致性
- [Teaching Lens Part 5](distributed-systems/ds-12-teaching-part5.md) —— Modules 13–15 的讲解组织方式 + 值得偷的讲法
- [Coordination Service](distributed-systems/ds-13-coordination.md) —— Module 16：ZooKeeper、znode + watch、guarantee、ZAB、无惊群锁
- [Teaching Lens Part 6](distributed-systems/ds-14-teaching-part6.md) —— Module 16 的讲解组织方式 + 值得偷的讲法
- [Distributed Data Stores](distributed-systems/ds-15-data-stores.md) —— Module 17：HBase / Cassandra / Spanner / FaunaDB，QUORUM ≠ linearizable、TrueTime、determinism
- [Teaching Lens Part 7](distributed-systems/ds-16-teaching-part7.md) —— Module 17 的讲解组织方式 + 值得偷的讲法
- [Messaging + K8s + Ledger](distributed-systems/ds-17-messaging-k8s-ledger.md) —— Modules 18–20：Kafka lever + 投递语义、Kubernetes reconciliation、 Corda need-to-know 账本
- [Teaching Lens Part 8](distributed-systems/ds-18-teaching-part8.md) —— Modules 18–20 的讲解组织方式 + 值得偷的讲法
