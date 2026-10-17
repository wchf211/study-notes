# Eureka（Module 3）：LLM 自动写 RL 奖励函数

> 这一章是全课最强的“meta-agent”例子：agent 的 action 是写代码，environment 是 RL 训练 run。读的时候盯住两个区分：reward（代理指标，LLM 写的）vs fitness（真实目标，用来评判的）。混了这两个，整章就白读了。

## 1. 奖励设计本身当成 coding 任务

**Reinforcement learning / RL（强化学习）**：agent 靠 reward function（奖励函数）知道自己做得好不好。手写奖励函数是慢、脆、专家活。**Eureka** 把奖励设计本身当成 LLM agent 的 coding 任务：给任务描述 + 环境源代码，LLM 写出可执行的奖励函数；fast simulator（快速模拟器，Isaac Gym）训一个 policy；结果反馈回来修正。

循环：generate（生成）→ train / evaluate（训/评）→ reflect（反思）→ improve（改进）。Eureka 是 meta-agent：它的“action”是写代码，它的“environment”是 RL 训练 run。

什么时候用：奖励 shaping（塑形）是瓶颈，且有快速可重置 simulator + 可观测 state variable（状态变量）。什么时候别用：没有 simulator、没有清晰 fitness 度量、环境代码不透明。例：教模拟机械手转笔 —— 不用手调几十个奖励项，让 LLM 提议、测试、修正。失败模式：reward hacking（奖励作弊）—— 奖励分高，行为却是错的。Fix：用 true fitness 评估，不只看生成的 reward；加 reflection。

## 2. Zero-Shot 生成：第一稿通常能跑，但别直接发货

第一份奖励来自一个结构良好的 prompt，零例子。Prompt 给任务描述 + 环境代码（变量、动力学、observation / action space 观察/动作空间），让 LLM 看清哪些 state variable 存在、物理上什么可能，并要求输出可执行奖励函数 + 设计选择的 reflection（反思）。

输出质量取决于输入：state variable、任务目标、observation / action space、环境代码本身是关键配料。Zero-shot 奖励通常能用但不完美 —— 常钻 loophole（漏洞，如原地不动、抄近路），需要 refine（精炼）。用 zero-shot 快速拿到 plausible 起点。别直接发货第一稿。例：让机器人把球推到目标，第一版奖励可能重距离、忘了掉球惩罚。失败模式：hacking —— 高分、错行为。Fix：把 zero-shot 当草稿，用 fitness 测 + 迭代。

## 3. Evolutionary Search：进化，不是单次修改

改进是进化，不是单次编辑。每代：LLM 产一批候选奖励函数（如 16 个）。每个都在 simulator 里并行训一个 policy，按 true fitness 取最好留下来。下一代从 winner 出发 mutate / improve（变异/改进）。重复几代，奖励稳步提升，对坏第一稿也 robust（鲁棒）。

强大但贵：总成本 \~ population（种群）× iterations（代数）× training time（训练时间），之所以可行，是 GPU 加速模拟让很多并行 RL run 便宜快速。什么时候用：能负担很多并行评估、需要鲁棒性。Evaluator 慢/贵时别用。例：16 个奖励留最好，变异出 16 个新变体，重复 5 次。失败模式：随机重启浪费算力，或收敛到局部最优。Fix：留最好 parent + 加 diversity（多样性）+ cap iterations。

## 4. Reward Reflection：别只回一个分数，要回诊断

单个 fitness 分数只说“好/坏”，不说修哪。**Reward reflection（奖励反思）** 加文本诊断：报告 fitness、列每个 reward component（奖励分量）在训练中的取值、每个分量怎么变的，然后让 LLM 解释哪个分量帮了、哪个被忽略/被利用、下一步改什么。诊断变成下一代 prompt 的一部分，把盲目变异变成 targeted editing（定向编辑）—— 如某分量一直平着，就加权重；某分量主导并导致 hacking，就加 cap。

没有 reflection，性能掉很多（报告里约 28.6% 平均 normalized-score 下降）。Evaluator 回 rich trace（丰富轨迹）时一定要用 reflection。别只回 scalar（标量）。例：reflection 显示“upright”项从不变 → LLM 加权重；“speed”项爆炸 → 加 cap。失败模式：只看总分，改错项。Fix：永远回 per-component trace + 书面诊断。

## 5. Human Feedback + Novel Reward Discovery：人管风格，机器管探索

三种人介入方式：human initialization（人写初始奖励，再改进）、human feedback（人看训出的行为给文本反馈，如“走得更稳，不只更快”）、co-piloting（人机交替）。

Human feedback 最有用的时候：fitness 漏掉了风格 / 安全偏好，而人能看出来。一个醒目发现：Eureka 常发现和人类写的奖励弱相关甚至负相关的奖励，却表现更好 —— 它探索更广的设计空间，找到人类不会试的非显然原理。Human feedback 管风格、安全、偏好对齐；agent 探索管原始性能。别指望人和机器的奖励长得像。例：人说“要稳的步态不要只快”；Eureka 保留速度但加稳定性项，用户更喜欢结果。失败模式：把 human feedback 当一次性 prompt，而不是迭代循环。Fix：展示训出行为 → 收反馈 → 带反馈重跑进化。

## 6. 评估：Hold 住训练配置，只比奖励

29 个 RL 环境、10 种机器人（9 个 Isaac Gym 任务 + 20 个 dexterous manipulation 灵巧操作任务），所有方法用同一 PPO 训练，只比奖励不同。不少 benchmark 在 LLM 知识截止之后，测的是真 zero-shot 推理，不是记忆。Baseline：人类写奖励、sparse fitness-only 奖励、L2R（早期模板化 LLM 方法）。指标：Isaac 任务用 human-normalized score（高于 1 = 超过人类），dexterity 任务用 success rate（成功率）。

结果：平均比人类奖励高 \~52% normalized improvement；迭代稳步提升；奖励新颖；之前不可行的任务（如灵巧转笔）被解决。Reflection 是必需的（去掉它约掉 28.6%）。设计 takeaway：LLM 当推理核心、self-improvement loop + 丰富反馈、用快速工具增强、为泛化设计、加人协作、留出新颖空间。失败模式：不同训练配置比方法，把训练优势算到奖励头上。Fix：RL 算法 / 超参 hold 常量，用 true fitness / success 度量。

## 这门课怎么教的（Module 3）

先给真实瓶颈（手写奖励很痛苦）+ 一张难忘图（机械手学转笔），再按 Eureka 自己的 loop 顺序：generate → evolve → reflect → 加人 → 证明有效。同一任务（灵巧操作）反复出现，ablation 证据（有/无 reflection）当成“为什么重要”的证明。心智模型：LLM 当 meta-agent，action 是代码，进化 + 反思是改进引擎。每课只隔离一个机制 + 给失败模式，好跟。难处：fitness（真实目标）和 reward（代理指标）的区分要反复讲 —— 读者容易混。最值得偷的：**每个机制都给 ablation —— 去掉它会坏什么 —— 而不只是成功故事**。要避免：先给最终 52% 提升，再讲分数到底 normalized（归一化）成什么。

## 自测 4 问（带答案要点）
1. Eureka 的循环是什么？它的 action 和 environment 各是什么？
   → generate → train/evaluate → reflect → improve；action 是写代码，environment 是 RL 训练 run。
2. Reward vs fitness 怎么分？为什么必须用 fitness 评估？
   → reward 是 LLM 写的代理指标，fitness 是真实目标；只看 reward 会被 hacking 骗。
3. Evolutionary search 为什么贵？什么时候别用？
   → 成本 \~ population × iterations × training time；evaluator 慢/贵时别用。
4. Reflection 为什么必需？它回什么，不只回什么？
   → 去掉约掉 28.6%；回 per-component trace + 书面诊断，不只 scalar 分数。
