# MuLan + OpenClaw（Modules 9 + 11）

> 这两章是“同一套 agent 思想，两个不同领域”：MuLan 把生成图像变成 agentic 任务（规划 → 逐步生成 → 每步验证）；OpenClaw 把个人助理变成操作系统（gateway + daemon + memory + skills + safety）。共同点：**把一次性的生成/回答，变成带循环、记忆、验证的系统**。

（注：Module 10 只有 1 个 thought exercise“Design a Multi-Agent Medical Diagnosis System”，按跳过规则未读，没有 teaching lesson。）

## 1. MuLan：多物体生成为什么 One-Shot 不行（Module 9.1）

**One-shot text-to-image diffusion（一镜式文本生图扩散）**：一次生成整张图。Prompt 里几个物体时，要一次把每个物体、属性、空间关系都做对 —— 结果常丢物体、swap 颜色/属性（属性串位）、放错位置。

**MuLan** 把生成当 agentic 任务：LLM agent 规划场景，再逐步生成，一次一个物体，每步检查。架构：LLM planner（规划器）+ diffusion generator（扩散生成器）+ VLM critic（视觉语言模型批评者）+ 可选 human feedback（人工反馈）。

Prompt 捆很多物体/关系时用 progressive generation（逐步生成）。复杂多物体场景别用 one-shot。例：“红车在蓝房子左边，狗在车前面”—— one-shot 可能丢狗或换颜色；MuLan 先建车、再建房、再建狗，每步检查。失败模式：attribute leakage（属性泄漏，如红色涂到房子上）。Fix：一次生成验证一个物体。

## 2. MuLan 规划 + 逐步生成：Plan 是关键产物（Module 9.2）

LLM 先写 global plan（全局计划）：ordered sub-prompt（有序子提示）列表，每个物体一个，写明物体、属性、和已放物体的空间关系。生成是顺序的：先生成第一个物体，然后每个下一个物体都 condition（条件）于当前 canvas（画布）+ 新 sub-prompt + plan，让后放的物体相对早放的定位。Overlapping / occlusion（重叠/遮挡）靠生成顺序处理（按深度 back-to-front，后到前）和 masking / preserving（遮罩/保护）已正确的区域。

Plan 是关键产物 —— 它把含糊段落变成可执行序列。空间关系重要时用显式规划。别让扩散模型自己从段落里推断布局。例：plan = [右边蓝房子]、[房子左边的红车]、[车前面的狗]，按该顺序生成。失败模式：一次全生成，指望布局自己出现。Fix：有序 sub-prompt + canvas conditioning。

## 3. VLM-Feedback + Human-in-the-Loop：每步验证，别等最后（Module 9.3）

每个物体生成后，**VLM critic** 检查当前 canvas：物体在吗、属性对吗、位置/关系对吗、该改什么？检查失败，LLM 修改 sub-prompt 或重生成该物体（带 retry cap，重试上限），再往下走 —— 错误早抓，不等整张图做完。

**Human-in-the-loop（人工在环）** 暂停让人看中间 canvas、编辑 plan、或重定向下一个物体，对 VLM 判不了的风格和意图尤其有价值。多步生成用 per-step verification（每步验证）。别等最后才检查。例：VLM 看到车是蓝色不是红色 → 用更强颜色约束重生成车，再加狗。失败模式：早期错物体毒害后面所有物体。Fix：每步验证 + cap 重试 + 允许人工重定向。

## 4. 评估 MuLan：按物体正确性度量，不只看整体好看（Module 9.4）

评估：multi-object prompt（多物体提示，如 T2I-CompBench 式属性/关系任务），对 one-shot 扩散 baseline 和其它组合方法，用自动指标（物体/属性/关系准确率）+ 人工评分（整体质量 + prompt 对齐）。报告模式：MuLan 提升物体存在、属性绑定、空间关系，尤其 3+ 物体复杂 prompt，代价是更多推理步数和延迟。Ablation（消融）显示 planning 和 VLM feedback 各有贡献 —— 去掉 feedback，早错会 cascade（级联）。

设计 takeaway：复杂生成要分解、增量验证、人工在环管意图、接受 latency 当正确性的代价。质量比速度重要时用这个 pattern。别只按单张图美学判断。例：3 物体 prompt，one-shot 对 1/3，MuLan 对 3/3 且颜色对。失败模式：为速度优化，发一张漂亮但错的图。Fix：按每物体正确性度量，不只整体 appeal（吸引力）。

## 5. OpenClaw：个人助理当 OS，不是更大的 Prompt（Module 11）

**OpenClaw** 是 persistent、personal assistant（持久、个人助理），不是 one-shot chatbot。设计把 LLM 当更大系统里的一个组件：gateway（网关，连聊天 app、语音、web）+ daemon（守护进程，常驻，保状态、调度工作、恢复）+ canvas（共享记忆/工作区）+ tools、skills、安全控制。

**Persistence（持久性）** 来自：durable memory（持久记忆：笔记、偏好、历史）、scheduled work（定时工作：cron job、heartbeat 定期检查工作）、proactive behavior（主动行为：在边界内不被问也能行动）。**Skills（技能）**：小而可复用的能力包（如搜索、日历、文件操作），不用 retrain（重训）就能加。**Safety（安全）**：sandboxing（沙箱）、least-privilege tool（最小权限工具）、consequential action（有后果的动作）要 approval（批准）。

助理必须记住、随时间行动、跨 app 工作时用这个架构。别靠把全部历史塞进一个 prompt 来伪造持久性。例：助理记得用户项目，按 schedule 查邮件，每天早上 draft（起草）总结 —— 发送前批准。失败模式：stateless chatbot（无状态聊天机器人），session 之间全忘。Fix：持久记忆 + 定时循环 + 显式批准门。

## 这门课怎么教的（Modules 9 + 11）

Module 9 先给生动失败（one-shot 图丢物体 / 错颜色），再按 MuLan 自己的顺序：plan → 逐步生成 → 验证 → 证明。同一多物体 prompt 反复出现，before / after 图是核心证据。心智模型：LLM 当 director（导演）、diffusion 当 painter（画家）、VLM 当 critic（批评者）。每种机制映射到一次可见的图像变化，好跟。难处：planning（什么顺序）和 conditioning（画布怎么引导下一个物体）的区分会模糊。最值得偷的：**展示每个物体之后的中间画布，不只最终图 —— 进展本身就是证明**。要避免：不展示哪个物体在哪一步被修好，就给最终图质量分。

Module 11 先给对比（chatbot vs always-on assistant），再按层走系统：gateway → daemon → memory → skills → safety。心智模型：LLM 当操作系统式外壳里的引擎。好跟，因为它在更大规模上复用了 Module 1 的解剖（model / tools / instructions / memory）。最值得偷的：**把个人助理框成 OS（daemon + memory + tools），而不是更大的 prompt —— 持久性变得具体**。要避免：不展示一次完整 day-in-the-life（一天生活）循环就列功能。

## 自测 4 问（带答案要点）
1. MuLan 三个角色各是什么？为什么逐步生成？
   → LLM planner + diffusion generator + VLM critic（+ 可选人反馈）；一次一个物体、每步验证，防止早错 cascade。
2. Plan 为什么是关键产物？Overlapping 怎么处理？
   → 含糊段落变成可执行序列；按深度 back-to-front 顺序 + masking / preserving 已正确区域。
3. VLM 反馈和人反馈各管什么？为什么每步验证？
   → VLM 管存在/属性/位置；人管风格/意图；早错会毒害后面物体。
4. OpenClaw 的持久性来自哪三样？为什么不是塞历史进 prompt？
   → durable memory + scheduled work + proactive behavior；塞历史进 prompt 贵、会丢、不可审计。
