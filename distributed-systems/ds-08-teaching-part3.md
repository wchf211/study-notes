# Teaching Lens Part 3：Modules 9–10 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 9–10（Part 3 已读部分）。Modules 11–31 的讲法等后续再补。

## 1. Module 9：先给 Puzzle，再分两层，最后引出替代品

先给人类 scheduling puzzle（两个学生，2 点/3 点，一块表快）说明 wall-clock agreement 并不 trivial，再分 practical（真实时钟漂移，NTP 帮但不完美）vs theoretical（相对论：同时性取决于观察者；消息延迟让“同时”无定义），最后引出 logical clock 当 wall time 的替代。视觉是 timeline + 时钟面；基调是概念，不是公式堆。

值得偷的：**Puzzle 先行**。一个生活 puzzle 让“同时没有定义”变得直观，比先讲相对论好。**分层收尾**：practical 和 theoretical 分开讲，观众知道哪部分是工程现实、哪部分是理论边界。

## 2. Module 10：工具箱顺序 + 同一张图反复用

按工具箱顺序建：排序词汇（全序 vs 偏序）→ 因果定义（happened-before）→ Lamport（便宜、弱）→ vector（精确、贵）→ version vector（per-item 变体）→ dotted version vector（可扩展变体）→ snapshot（全局一致割）→ Chandy-Lamport 解法 → 物理-逻辑合成。每课时钟都复用同一张 3 节点执行图，读者可以跨方案比较 timestamp。最密的是时钟条件和向量大小下界。

值得偷的：**同一张执行图反复用**。换一个时钟方案，不换图，观众直接比 timestamp。讲时钟系列时，不要每课换图。**便宜-弱 vs 精确-贵对照**：每个新时钟先说它比上一个贵在哪、强在哪，观众不用背定义，先记住 trade-off。

## 3. 容易讲砸的坑

1. **先讲公式，再讲问题**：时钟条件（clock condition）先给，观众不知道它在解决什么。讲的时候要先给“wall 时间戳判因果会错”的例子，再给条件。
2. **Lamport 和 vector 的区分讲不清**：必须用一句话钉住 —— Lamport 只能单向（因果 → 时间戳小），vector 是双向（当且仅当）。这句话要反复说。
3. **Snapshot 只讲算法，不讲“为什么要一致”**：先给孤儿接收的例子（B 显示收到、A 显示没发），再讲 marker，观众才知道算法在防什么。

## 自测 4 问（带答案要点）
1. Module 9 的讲解顺序？
   → Puzzle → practical vs theoretical → logical clock 当替代品。
2. Module 10 的讲解顺序？
   → 词汇 → 因果 → Lamport → vector → version → dotted → snapshot → Chandy-Lamport → 合成。
3. 讲时钟系列时，同一张图反复用的好处？
   → 换方案不换图，观众直接比 timestamp。
4. Lamport vs vector 必须钉住的一句话？
   → Lamport 单向（a→b 则 C(a)<C(b)）；vector 双向（V(a)<V(b) 当且仅当 a→b）。
