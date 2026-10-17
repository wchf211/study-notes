# Teaching Lens：Modules 1–3 怎么组织讲解，值得偷什么

> 这篇是讲法审计，不是技术复读。覆盖 Modules 1–3（Part 1 已读部分）；Modules 4–31 的讲法等 Part 2 再补。

## 1. 全课结构：先地基，再理论，再真实系统，再招式

- **Module 1**：短 orientation。明说课程是 theory-then-practice：先基础，再 transactions / consensus / time / order / networking / security，然后真实系统 case study，最后实践 pattern。还提醒：分布式思想不能只看一个来源，要 cross-reading（交叉阅读）。
- **Modules 2–3**：地基 + 形式化。Module 2 从具体请求讲到抽象判断；Module 3 从机制讲到形式化。Part 1 的顺序就是“先会判断，再会读文档”。
- **Modules 4–31（未读，见 outline）**：理论核心 → 配套 → 7 个 case study → practices & patterns。地图本身就是教学设计：别跳过基础直接读 case。

## 2. Module 2 的讲法：同一个请求，层层加机器

- **Hook**：一个具体多服务 web 请求（浏览器 → DNS → LB → app → DB）+ 单机 scaling 类比（计算器）。心智模型一句话：“同样的请求，机器多了，失败模式新了。”
- **顺序**：为什么分布 → 错误假设 → 为什么难（部分失败、ambiguity、设计）→ 怎么判断正确 → 模型 → 失败分类 → exactly-once 悖论 → 检测 → 无状态/有状态。每一步都回答上一步留下的问题。
- **图**：简单的 request-flow 图，不炫技。容易跟，因为每课都在同一张请求图上加东西。

## 3. Module 3 的讲法：先机制，后形式化；同一个例子反复锚定

- **Hook**：一个 toy 例子 —— 100 GB 图书 DB / 15k 订单 —— 先让 partitioning 有动机，再层层加机制（partition → replication → quorum），最后才上形式化（safety、ACID、CAP、consistency、isolation）。
- **反复锚定**：银行转账的小事务时间线在 ACID、isolation、consistency 里反复出现；“更强 vs 更弱”的 ladder（梯子）反复出现。符号和相似模型堆起来时，读者有地方回锚。
- **难处**：符号 + 很多相似模型堆在一起时仍吃力。课程的解法是重复锚定 + ladder，不是减少模型数量 —— 模型数量是内容本身，不能砍，只能帮读者定位。

## 4. 值得偷的讲法

1. **同一个请求 / 同一个例子贯穿**：Module 2 的请求图、Module 3 的转账例子。讲分布式系列时，一个 running example 比每课新例子好。
2. **机制 → 形式化**：先让读者手里有机制（quorum 怎么工作），再给形式化名字（linearizability 是什么）。反过来讲，形式化就是天书。
3. **Ladder（梯子）定位法**：每次讲一个新模型，先说它在 ladder 上比谁强、比谁弱。观众不用记住每个模型的完整定义，先记住相对位置。
4. **“为什么要形式化”单独一课**：不把形式化当理所当然，而是论证“含糊术语藏不同保证”。讲理论系列时，这课是必要的 —— 否则观众会问“背这些名字干什么”。

## 5. 容易讲砸的坑

1. **符号堆积不给锚**：相似模型（sequential / causal / eventual、serializable / snapshot / repeatable）连讲时，必须每讲一个就回 ladder 定位一次，否则观众三分钟就混。
2. **标签当保证**：课程反复提醒别信“consistent”“serializable”标签。讲的时候要把“标签 vs 实际 anomaly 保证”当成固定环节，不是脚注。
3. **跳过地基直奔 case**：课程 Module 1 就警告了。讲系列时，CAP 之前不能讲 Cassandra 的 knob。

## 自测 4 问（带答案要点）
1. Modules 1–3 的讲解顺序是什么？为什么是这个顺序？
   → 地基（判断）→ 机制 → 形式化；先会判断，再会读文档；机制在前，形式化才有地方挂。
2. Module 2 和 Module 3 各用哪个 running example 锚定？
   → Module 2：多服务 web 请求图；Module 3：100 GB 图书 DB + 银行转账时间线。
3. 讲相似模型时，课程用什么办法防混？
   → Ladder 定位（强于谁、弱于谁）+ 同一个例子反复锚定。
4. 为什么“为什么要形式化”值得单独一课？
   → 含糊术语藏不同保证；要论证形式化是用来读厂商文档的，不是背名字。
