# Teaching Lens Part 3：Modules 7–9 + 11–13 怎么组织讲解

> 这篇是讲法审计，不是技术复读。覆盖 Modules 7–9 + 11–13（Part 3 已读部分；Module 10 只有 1 个 thought exercise，已跳过）。全课 42 课至此读完。

## 1. Module 7：同一任务跑两遍，标注分叉点

先给真实世界 messiness（弹窗、登录、变布局）引出多模态动机，再按 WebVoyager 循环顺序：挑战 → 架构 → 并排比较 → 证据。一个 running task type（在真实站上找/提取）贯穿，traced comparison 是核心教具。最值得偷的：**同一任务用两种设计各跑一遍，标注到底哪一步一个失败、一个成功**。要避免：不先讲清什么算成功就给成功率。

## 2. Module 8：先给真实运行的 Log，再讲代码

先给一个 fixed 任务的 live demo 锚定代码，再自底向上走：runtime → tools → assembly。同一 LinkedIn 任务和同一批 artifact 文件反复出现，代码映射到可观察行为。最值得偷的：**讲工具代码前，先给一次真实运行的 artifact / log —— 学习者先看到每个文件为什么存在**。要避免：不标 stage transition 就贴完整 log dump。

## 3. Module 9：展示中间画布，不只最终图

先给生动失败（one-shot 图丢物体 / 错颜色），再按 MuLan 自己的顺序：plan → 逐步生成 → 验证 → 证明。同一多物体 prompt 反复出现，before / after 图是核心证据。最值得偷的：**展示每个物体之后的中间画布，不只最终图 —— 进展本身就是证明**。要避免：不展示哪个物体在哪一步被修好，就给最终图质量分。

## 4. Module 11：把助理框成 OS

先给对比（chatbot vs always-on assistant），再按层走系统：gateway → daemon → memory → skills → safety。最值得偷的：**把个人助理框成 OS（daemon + memory + tools），而不是更大的 prompt —— 持久性变得具体**。要避免：不展示一次完整 day-in-the-life 循环就列功能。

## 5. Modules 12–13：Checklist + 参考架收尾

Module 12 用已见过的例子绑定的可复用 checklist 收尾，不讲新理论。Module 13 当参考架：glossary 查术语、cheatsheet 选 pattern、landscape 选框架，每个都以 worked application question 结尾。最值得偷的：**用一个“选 pattern / 框架”的 worked 练习收尾，复用前面 case**。要避免：把附录当可选 —— 它是真实设计工作里最可复用的部分。

## 6. 全课讲法总结（Modules 1–13）

- **失败对比开场**：每章几乎都先给失败，再给解法。
- **同一任务 / 同一例子贯穿**：假期规划、电影/音乐推荐、转笔、网格导航、LinkedIn、同一多物体 prompt —— 每章一个 running example。
- **Ablation 当证明**：去掉它会坏什么，比“它多厉害”更有说服力。
- **按系统自己的执行顺序讲**：pipeline 怎么跑，课就怎么排。
- **Checklist + 参考架收尾**：附录不是尾巴，是工具书。

## 自测 4 问（带答案要点）
1. Module 7 最值得偷的讲法？
   → 同一任务跑两遍，标注分叉点。
2. Module 8 最值得偷的讲法？
   → 先给真实运行的 log / artifact，再讲代码。
3. Module 9 最值得偷的讲法？
   → 展示中间画布，不只最终图。
4. 全课收尾为什么用 checklist + 参考架？
   → 观众离开时带走可复用的决策工具，不是新理论。
