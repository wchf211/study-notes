# Teaching Lens：这门课怎么组织讲解，值得偷什么

> 这篇不是技术内容复读，而是这门课的“讲法审计”：它怎么开场、怎么排序、怎么用例子、哪里好讲、哪里容易讲砸。给未来做 AI agent / harness 技术讲解用。

## 1. 全课结构：一张图 + 一套模板 + 两台整机

- **一张 recurring 图**：reason–act–observe–reflect loop 在 Module 1 出现，之后每个 pattern 都映射回这张图。观众不用每课学一张新图。
- **一套重复模板**（Module 2 每课相同）：白话定义 → 类比 / 图 → 一步步怎么做 → 代码走读 → 好处 / 局限 → when-to-use 清单。模板重复是优点：观众可以把五个 pattern 并排比较。
- **两台整机**（Module 3）：两个端到端 case，每个配 pattern-mapping table（pattern → 负责什么 → 没了它会坏什么）。Pattern 在真实 pipeline 里再出现一次，才算真正学会。
- **一条 build ladder 收尾**（Module 4）：简单 → 测 → 再加复杂。给读者“下次动手照着走”的规则，不是总结复读。

## 2. 值得偷的讲法

1. **What-it-is / when-to-use / failure-mode 三件套**：每讲一个 pattern，都给这三样。观众离开时带走的是判断力，不是名词。
2. **Pattern-mapping table**：case study 里每行写清 pattern 在本系统里的角色 + 缺了它的后果。比“这个系统用了 X、Y、Z”好一个档次。
3. **Build ladder 做结尾**：结尾不复读，而是给一条动手顺序。观众合上笔记就知道下一步干什么。
4. **Recurring diagram**：全课只用一张核心图反复映射。讲 harness 系列时，perceive–reason–act / loop + tools + memory + eval + guardrails 那张图就该承担这个角色。

## 3. 容易讲砸的坑（这门课自己踩过的）

1. **一课一个新例子**：Module 1–2 每课换领域（天气、旅行、家教、客服、研究），每课都具体，但跨课联系要读者自己连。讲系列视频时，应该用一个 running example 贯穿（比如同一个工单分流 agent），新 pattern 只是给同一台机器加零件。
2. **Evaluator-optimizer 当并列项**：它其实是包在任意 pattern 外面的反馈环。按并列第五讲，读者会误归档。讲的时候要明确说“这是 layer，不是 peer”。
3. **Breakout / mock 伪装成课程**：两个 breakout 打开是交互式 mock 面试，没有静态课文；最后一课的 Next 指向 mock interview，看起来像课程没完。做讲解时，练习和正课要在导航上分清，正课结尾就停在 synthesis。

## 4. 和另两门课的讲法对照

- **AI Product Engineering**：一个 running example + failure-driven（先给失败，再给解法）。这门课缺的正是 running example。
- **Agentic Crash Course**：一个可复用框架 + verifier/critic motif 反复出现。这门课的 pattern-mapping table 和它异曲同工。
- **三门合起来**：running example（AIPE）+ framework spine（Crash Course）+ pattern-mapping table（本课）= 一个可以直接用的讲解脚手架。

## 自测 4 问（带答案要点）
1. 这门课全课的讲解骨架是什么？
   → 一张 recurring 图 + Module 2 重复模板 + Module 3 两台整机 + Module 4 build ladder。
2. 最值得偷的两个讲法？
   → what / when / failure 三件套；pattern-mapping table（pattern → 角色 → 缺了会坏什么）。
3. 这门课最大的讲解坑是什么？讲系列视频时怎么避？
   → 一课一个新例子，联系靠读者自己连；改用一个 running example 贯穿。
4. Evaluator-optimizer 为什么不能当并列第五讲？
   → 它是包在任意 pattern 外面的反馈环，是 layer 不是 peer。
