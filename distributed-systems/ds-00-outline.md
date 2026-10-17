# Course Outline：全课地图 + 续读点

> 怎么用这张地图：先读 Part 1（Modules 1–3，打地基），再按需跳。Transactions / Consensus / Time / Order 是理论核心；Networking / Security 是配套；Case Study 1–7 是真实系统（HDFS/GFS、Zookeeper、BigTable/Cassandra/Spanner、Kafka、K8s、Corda、MapReduce/Spark/Flink）；Practices & Patterns 是落地招式。Part 1 停在 Module 3 末尾，续读点是 **Module 4 Introduction to Distributed Transactions**（先跳过 Module 3 末尾的 Quiz on Distributed System Concepts and Theorems）。

**已读（Part 1）**：Modules 1–3，26 teaching lessons。
**未读**：Modules 4–31，见下。计数含 quizzes / exercises；`[Q]` = Quiz（跳过），`[EX]` = Exercise / AI-eval（跳过），其余为 teaching lessons。

1. Before Getting Started（1）：Introduction ✅
2. Introduction to Distributed Systems（10）：Getting Started；Fallacies of Distributed Computing；Difficulties Designing Distributed Systems；Measures of Correctness；System Models；Types of Failures；The Tale of Exactly-Once Semantics；Failure in the World of Distributed Systems；Stateless and Stateful Systems；Quiz [Q] ✅
3. Basic Concepts and Theorems（18）：Partitioning；Algorithms for Horizontal Partitioning；Replication；Primary-Backup Replication；Multi-Primary Replication；Quorums；AI-eval on Replication [EX]；Safety Guarantees；ACID Transactions；The CAP Theorem；Consistency Models；CAP Theorem's Consistency Model；Isolation Levels and Anomalies；Prevention of Anomalies；Consistency and Isolation；Hierarchy of Models；Why All the Formalities?；Quiz [Q] ✅
4. Distributed Transactions（2）：Introduction to Distributed Transactions；Quiz [Q] ⏳ 下一个
5. Achieving Isolation（6）：Achieving Serializability；Pessimistic Concurrency Control (PCC)；Optimistic Concurrency Control (OCC)；Achieving Snapshot Isolation；Achieving Full Serializable Snapshot Isolation；Quiz [Q]
6. Achieving Atomicity（5）：Hard to Guarantee Atomicity；2-Phase Commit (2PC)；3-Phase Commit (3PC)；Quorum-Based Commit Protocol；Quiz [Q]
7. Concluding Distributed Transactions（2）：How It All Fits Together；Long-Lived Transactions and Sagas
8. Consensus（12）：Defining the Consensus Problem；FLP Impossibility；The Paxos Algorithm；Intricacies of Paxos；Paxos in Real Life；Replicated State Machine via Consensus；Distributed Transactions via Consensus；Introduction to Raft；Communication among Raft Nodes；Raft's Implementation；Standing on the Shoulders of Giants；Quiz [Q]
9. Time（5）：What is Different in a Distributed System?；A Practical Perspective；A Theoretical Perspective；Logical Clocks；Quiz [Q]
10. Order（10）：Total and Partial Ordering；The Concept of Causality；Lamport Clocks；Vector Clocks；Version Vectors；Dotted Version Vectors；Distributed Snapshot Problem；Solving the Distributed Snapshot Problem；Physical and Logical Time: Closing Thoughts；Quiz [Q]
11. Networking（9）：Introduction；The Physical Layer；The Link Layer - Services；The Link-Layer Protocols；The Network Layer；The Transport Layer；The Application Layer；Taking a Step Back；Quiz [Q]
12. Security（7）：Introduction；Authentication；Confidentiality；Integrity；A Cryptography Primer；Symmetric/Asymmetric Encryption and Digital Signatures；Quiz [Q]
13. Security Protocols（5）：Transport Layer Security (TLS)；Public-Key Infrastructure (PKI)；Web of Trust (PGP)；OAuth Protocol；Quiz [Q]
14. From Theory to Practice（1）：Introduction
15. Case Study 1: Distributed File Systems（5）：HDFS and GFS；Creating and Reading Files；Writing and Deleting Files；GFS Consistency Model；Quiz [Q]
16. Case Study 2: Distributed Coordination Service（6）：Coordination Service；Zookeeper；Guarantees Provided by Zookeeper；Zookeeper's ZAB Protocol；Examples of Powerful Primitives by Zookeeper API；Quiz [Q]
17. Case Study 3: Distributed Data Stores（16）：Introduction；BigTable/HBase Architecture；Appends and Read Operations in HBase；Guarantees Provided by HBase；Cassandra's Data Model；Cassandra's Cluster Internode Communication；Cassandra's Consistency Levels；Linearizability Violations in Cassandra；Linearizability Guarantees by Cassandra；Cassandra Performing Queries Efficiently；Spanner's Data Model；Spanner's Architecture；Spanner using TrueTime；Spanner Operations；FaunaDB；Quiz [Q]
18. Case Study 4: Distributed Messaging System（5）：Introduction to Kafka；Kafka Levers；Kafka's Messaging Guarantees；Transactions, Storage Layout, and other Guarantees；Quiz [Q]
19. Case Study 5: Distributed Cluster Management（3）：Kubernetes；Components of Manager and Worker Nodes；Quiz [Q]
20. Case Study 6: Distributed Ledger（5）：Introduction to Corda；Corda's Data Model；Corda's Architecture；Backwards Compatibility provided by Corda；Quiz [Q]
21. Case Study 7: Distributed Data Processing Systems（10）：Introduction；Introduction to MapReduce；MapReduce's Manager-Worker Architecture；Introduction to Apache Spark；DAG of Stages in Apache Spark；Perks of Apache Spark；Apache Flink；Time and Watermarks in Flink；Failure Recovery in Flink；Quiz [Q]
22. Practices & Patterns（1）：Introduction
23. Communication Patterns（4）：Creating and Parsing Data；Transfer of Data；Datastores for Asynchronous Communication；Communication Models
24. Coordination Patterns（1）：Coordination Patterns
25. Data Synchronization（3）：Data Synchronisation；Event Sourcing；Change Data Capture (CDC)
26. Shared-nothing Architectures（2）：Sharing Problems and their Solution；Benefits and Drawbacks
27. Distributed Locking（2）：Leases in Distributed Systems；Preventing Safety Risks in Leases
28. Compatibility Patterns（2）：Backwards Compatibility；Maintaining Backwards Compatibility
29. Dealing with Failure（6）：Failure Handling Techniques；Applying Failure Handling Techniques；Retries；Containing Impact of Failure；Backpressure；Reacting to Backpressure
30. Distributed Tracing（1）：Recording Program's Execution
31. Concluding this Course（2）：Recap of the Course；Some More Things to Discover

## 自测 4 问（带答案要点）
1. 为什么 Part 1（Modules 1–3）必须先读，再看 case study？
   → 先学 vocabulary + guarantees，再抄真实设计；没 CAP 就读 Cassandra，它的 knob 看起来是任意的。
2. 想搞懂“为什么重试会 double-charge”，去哪个 module？
   → Module 2（部分失败 + exactly-once）+ Module 3（quorum / safety）。
3. 想搞懂“读到旧数据”，去哪个 module？
   → Module 3（replication、consistency models、isolation）。
4. Part 1 的续读点是什么？导航坑是什么？
   → Module 4 Introduction to Distributed Transactions，先跳过 Module 3 末尾 quiz；不要盲点 Next（可能指向 quiz），用 sidebar 导航。
