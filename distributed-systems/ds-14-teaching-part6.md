# Teaching Lens Part 6：Module 16 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Module 16（Part 6 已读部分）。Modules 17–31 的讲法等后续再补。

## 1. 顺序：动机 → 模型 → Guarantee → 协议 → Recipe

先给动机 —— 每个分布式系统都要对 membership、leadership、configuration 达成 agreement，每个系统自己重实现容易错 —— 再给出 ZooKeeper 当一个小、强一致的服务，让很多系统共享。从 coordination 问题 → ZooKeeper 数据模型（znode）→ guarantee → 协议（ZAB）→ 基于 API 的 recipe。用文件系统式树图 + client / session 演练。

值得偷的：**先给动机，再给模型，最后给 recipe**。Recipe 是这章的产出，不是附带。**“一个小服务，很多人共享”** 是这章的框架 —— 讲 coordination 时先问“为什么不每个 app 自己做”，再讲共享服务。

## 2. Guarantee 绑定 Recipe

每个 guarantee 映射到具体 recipe 时最易懂。讲 guarantee 时要绑定 recipe（哪个 guarantee 让哪个 recipe 成立），否则 guarantee 只是名词。

值得偷的：**讲保证时给用法**。不是“ZooKeeper 保证线性化写”，而是“线性化写让简单锁成立；follower 读可能 stale，所以读配置前要 sync”。保证一绑定用法，观众就知道什么时候该用哪个。

## 3. 容易讲砸的坑

1. **ZAB 讲成 Paxos 重讲**：ZAB 和 Paxos 重叠，但它是单 leader 复制日志特化的。讲的时候要先说“它不是通用 Paxos”，再讲两阶段。
2. **Watch 当订阅**：watch 只触发一次，必须反复强调。讲 recipe 时要把“每次通知后重新注册”写进 recipe 步骤里。
3. **Herd effect 只讲概念**：要给对比 —— 所有 waiter watch 同一个节点 vs 各 watch 前驱 —— 观众才看到 sequential 节点的作用。

## 自测 4 问（带答案要点）
1. Module 16 的讲解顺序？
   → 动机 → 模型 → guarantee → 协议 → recipe。
2. 为什么 recipe 是这章的产出？
   → 观众离开时要带走“怎么拼 lock / membership / config”，不是“ZooKeeper 是什么”。
3. 讲 guarantee 时为什么要绑定 recipe？
   → 否则 guarantee 只是名词；绑定用法，观众知道什么时候用哪个。
4. Watch 的关键性质怎么讲才记得住？
   → 只触发一次；讲 recipe 时把“每次通知后重新注册”写进步骤里。
