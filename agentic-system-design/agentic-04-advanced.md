# 进阶（Ch2 L8–L10）：自主奖励学习 + 多模态生成

> 三个硬核 case：让 LLM 自己写 RL 的奖励函数（Eureka）、用 Google ADK 把它工程化、多物体图像生成的 agent 化。共同 pattern：**目标难描述但便宜可测 → 生成器外面套 evaluate–reflect–improve 循环**。

## 1. 自主奖励学习（Eureka）

背景：**RL（reinforcement learning）**= trial-and-error 学，试动作、好的加固。一切 hinge 在 **reward function**（奖励函数：什么算"好"的打分规则）上。手写奖励是瓶颈：要领域专家 + 无限调参。

RL 词汇表：
- **policy**：学出来的决策规则（状态→动作），被训练的东西；
- **RLHF**：用人类偏好标签代替手写分数的训练；
- **reward hacking**：学习者钻字面指标的空子（经典：奖励"前进"的机器人学会原地振动骗传感器）——**任何分数都是 proxy，优化器专吃 proxy**，本课中心故障。

**Eureka 循环**（NVIDIA 研究，进化式 agent）：
1. LLM 生成一批候选 reward function；
2. 每个候选在仿真里训 policy；
3. 按任务指标评估 policy；
4. 最好的留下；
5. LLM **reflect**（赢家为啥赢）→ 定向 mutate 下一代；
6. 无人值守重复。

为什么 work：LLM 带物理/任务先验（第一稿就 sensible，不瞎搜）+ 进化用真实训练结果做 selection pressure（再好听的想法，训出来不行就死）+ reflection 把原始结果变成定向改进（不是盲 mutate）。
战绩：多数任务打败手写奖励，83% 达到人类水平或更好；**灵巧操作**上最亮眼——正是人类直觉最弱、手写"好"最难的地方。

工程 takeaway：reflection 和 generation 一样重要（没 critic 的生成器只会产自己盲区的变体）；评估必须多维（高分≠行为对——**看 rollout 视频，别只看数**，reward hacking 在视频里现形）；成本真实（每轮 N 个 LLM 调用 + GPU 训练——仿真便宜才划算）；pattern 通用化：**目标难描述但便宜可测 → 生成器套 evaluate–reflect–improve**。

## 2. 用 Google ADK 工程化

**Google ADK**（Agent Development Kit）：建多 agent 应用的框架——typed agents、tools、session state、sequential/loop 等编排原语。这课讲架构编排不讲语法：把 Eureka 每个职责映射成显式 agent，整个 loop 可观察可 debug。

Agent 名单：
- RewardDesigner（LLM 写候选 reward）；
- CandidateEvaluator（训+评估，出指标+rollout 可视化）；
- Selector（选赢家——**和评估故意分开，judge 和 meter 不能混**）；
- RewardReflector（诊断哪个 reward 组件起了作用、定向修——critic）；
- HumanReflection（可选 HITL checkpoint，贵循环的便宜保险）；
- ExitChecker（停机标准：达阈值 / 提升 < floor / 到 max iter）；
- IncrementIteration（循环计数）。

编排：setup agent 跑一次 → RewardEvolutionLoop（ADK loop pattern）循环 designer→evaluator→selector→reflector→（可选人工）→exit check→increment。

工程 lesson：
- loop 控制做成显式 agent，别裸 while——显式阶段可观察、可 debug、可中断；
- selection 和 evaluation 分开；
- human checkpoint 可选但留着；
- **每轮产物（指标+rollout 视频）是主要 debug 面**。

环境：Brax HalfCheetah 仿真，Colab T4。（只读了课，没跑代码。）

## 3. 多物体扩散的 Multimodal-LLM Agent（PMG）

问题：**Diffusion models**（从纯噪声一步步 denoise 成图）画单物体强，多物体场景翻车——**attribute leakage**（属性串色："红方块蓝圆球"画成发红的球）、丢物体、空间关系错。根因：单个文本 prompt 欠指定布局，**cross-attention**（prompt 词和图像区域的关联机制）把属性抹匀了。

**PMG（Progressive Multi-Modal Generation）**：别一次要整张图——agent 规划、分块建、再验。

五阶段：
1. **Parse**：LLM 把 prompt 转结构化 plan（物体清单+属性+空间关系）；
2. **Divide**：按前后排序生成（先背景后前景，遮挡自然对）；
3. **Conquer**：每个物体单独生成，配自己的 sub-prompt + **spatial mask**（指定画布区域），attention 锁局部，属性串不出去；
4. **Combine**：拼到一张画布，位置遮挡摆对；
5. **Refine**：**VLM**（vision-language model，图文都懂的模型）critique 成品（少物体？颜色错？布局烂？）→ 只重画问题区域；到质量线或 iteration budget 用完停。

为什么这是 agentic：规划（布局分解）+ tool use（diffusion 当画图工具、VLM 当 critic 工具）+ verify/reflect 循环 = L2 的 perceive–reason–act。
代价：多跑 diffusion + VLM critique，**quality 换 cost/latency 的干净 trade**。
故障：LLM 一开始空间关系 parse 错（plan 烂则全烂）、mask 重叠 blending artifact、分别生成的物体风格打架、refine 没硬 cap 会 stall。

**可移植 pattern**：单模型调用欠指定结构化输出时 → plan → 分块生成 → verify → repair。= plan-and-execute + critic，画在像素上。

## 自测 4 问（带答案要点）
1. Reward hacking 是什么？Eureka 循环五步？
   → 钻 proxy 空子；生成→仿真训→评估→选→reflect 定向变异。
2. 为什么 reflection 和 generation 一样重要？怎么发现 reward hacking？
   → 没 critic 只产盲区变体；看 rollout 视频，高分≠行为对。
3. ADK 实现里为什么 Selector 和 Evaluator 分开？loop 为什么做成显式 agent？
   → judge 和 meter 不混；可观察可 debug 可中断。
4. PMG 五阶段？attribute leakage 根因？
   → parse/divide/conquer/combine/refine；单 prompt 欠指定+cross-attention 抹匀。
