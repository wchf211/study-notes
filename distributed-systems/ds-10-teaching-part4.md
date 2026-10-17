# Teaching Lens Part 4：Modules 11–12 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 11–12（Part 4 已读部分）。Modules 13–31 的讲法等后续再补。

## 1. Module 11：分层 Tour + 同一例子贯穿 + End-to-End 收尾

紧凑的自底向上分层 tour，框架是“每层加了什么、留给上层什么”。先给 service vs protocol vs interface 词汇，再按 physical → link → network → transport → application 走，每层复用同一双 host 消息例子。最后退一步给 end-to-end argument：多数分布式关切活在 edge，不在网络里。视觉是分层图 + header / payload 草图。

值得偷的：**同一例子贯穿分层**。每层换例子，观众不知道层之间怎么接；同一双 host 消息例子走全栈，分层才看得见。**每层绑一个具体后果**：为什么 TCP in-order 投递会 hurt、为什么 UDP 仍需要 app 层 reliability —— 分层不是背诵，是后果。

## 2. Module 12：按目标组织，不按算法

按目标组织，不按算法：先讲分布式里安全是什么意思（攻击者在网络上，没有可信边界），再每目标一课 —— authentication（谁）、confidentiality（谁能读）、integrity（被改过吗）—— 再给实现它们所需的 crypto 工具箱（randomness / hash、对称 vs 非对称）。全程 Alice / Bob / Mallory 式场景。

值得偷的：**按目标讲安全，按目标选工具**。讲 crypto 时先问“它服务哪个目标”，观众不会在原语里迷路。**场景人名反复用**：Alice / Bob / Mallory 当固定角色，攻击场景不用每次重新介绍。

## 3. 容易讲砸的坑

1. **分层讲成 OSI 背诵**：分层课不讲“每层名字”，讲“每层加了什么、留给上层什么、会坏在哪”。背诵式分层观众听完还是不知道怎么 debug。
2. **Crypto 按算法讲**：按算法讲（RSA、DH、AES…）观众只记得名字。按目标讲（认证/机密/完整），观众知道什么时候用哪个。
3. **End-to-end argument 当口号**：要给反例 —— TCP retry 不让分布式事务 atomic —— 观众才知道它在说什么。

## 自测 4 问（带答案要点）
1. Module 11 的讲解顺序？
   → 词汇 → physical → link → network → transport → application → end-to-end 收尾。
2. 分层课为什么同一例子贯穿？
   → 每层换例子，观众不知道层之间怎么接；同一例子走全栈，分层才看得见。
3. Module 12 为什么按目标组织，不按算法？
   → 按目标讲，观众知道什么时候用哪个；按算法讲，只记得名字。
4. End-to-end argument 怎么讲才不是口号？
   → 给反例：TCP retry 不让分布式事务 atomic。
