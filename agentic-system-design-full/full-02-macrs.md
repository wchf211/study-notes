# MACRS（Module 2）：多 Agent 对话推荐

> 这一章的教学价值不在“推荐系统”，而在“一个任务捆了几种互相干扰的技能时，怎么拆”。MACRS 是全课第一个完整多 agent 系统，也是 planner / executor 分离 + reflection 的标准例子。

## 1. 设计挑战：多目标对话

**传统推荐器**：固定策略，不能自然自适应对话。**单个 LLM**：能聊，但容易丢目标、漂移、没法同时平衡几个互相竞争的工作。

多目标对话要同时做到：推断用户真实偏好、问有用的澄清问题、推荐相关 item、解释推荐 —— 还要在偏好浮现时保持目标、持续适应。

MACRS 的解法：把工作拆成专科 agent 团队，由 planner 协调 + reflection loop（反思循环），没有一个 agent 需要样样都行。什么时候拆：一个任务捆了几种不同技能（理解、规划、推荐、表达），硬塞进一个 prompt 会互相干扰。什么时候别拆：简单 one-shot 推荐。例：音乐推荐先学到用户要“ upbeat 90 年代健身歌”，再缩小选择，而不是丢一个通用 top-10。失败模式：单个 agent 推荐太早、从不修正。Fix：planning 和 generation 分开 + 反馈驱动修正。

## 2. Act Planning Framework：对话是 Act，不是自由回复

**Dialogue act（对话行为）**：小动作，如问偏好、确认、推荐、解释、收尾 —— 不是自由回复。**Planner agent**：管策略，跟踪 goal、对话状态、用户画像，选下一个 act。**Responder（推荐 + 生成）**：执行 act。**Information Seeker**：另取画像 / item 知识。

Responder 自己再拆三个 role-specific sub-agent（角色专属子 agent）：
- Dialogue agent：理解意图。
- Recommendation agent：打分 / 调 retrieval 工具。
- Response agent：把选定的 act + 证据变成自然语言。

每个 agent 角色窄、推理风格自有、工具自有 —— planning 用 chain-of-thought，recommendation 用 retrieval 工具。什么时候用：对话既要计划又要 grounded 内容。别让 generator 一步同时定策略和内容。例：Planner 定“先问 genre 再推荐”；Recommender 按 stated taste 检索候选；Response 写成一句短问题。失败模式：act 互相矛盾 / 重复提问。Fix：Planner 独自拥有 act 选择权，其余只执行。

## 3. Reflection：反思改策略，不只是道歉

**Reflection（反思）**：系统检查自己最近的 turn，判断哪里错了，改未来行为 —— 不只是道歉。两个层级：

- **Turn-level reflection（单轮反思）**：每轮之后，检查 act 选对没、回复有没有帮助，用用户显式和隐式反馈（纠正、拒绝、follow-up、语气）。
- **Trajectory-level reflection（轨迹反思）**：整段对话之后，看全程，找出哪些 act 级选择反复帮 / 害最终结果，存成 compact lesson（紧凑经验）给未来对话。

Reflection 改的是策略，不只是下一句话。什么时候用：反馈在多轮对话里陆续到，同一个错可能重犯。别把每次用户纠正都当 one-off 回复修复。例：用户拒绝推荐说“太 mainstream（主流）”；reflection 记下该用户偏 niche（小众），之后倾向更冷门的匹配。失败模式：只改表面措辞，重复同样的坏推荐模式。Fix：诊断哪个 act / 假设失败了，更新 plan / policy。

## 4. 评估：任务成功 + 人评，不是流利度

评估三件套：offline data（离线数据）+ simulated user（模拟用户）+ human judgment（人工判断），因为没有任何单一指标能抓住对话质量。Offline 指标看是否达成正确推荐 + 用了多少轮；task success 用模拟用户看推荐是否真被接受；人工评 overall quality（整体质量）、explainability（可解释性）、coherence（连贯性）、user-friendliness（友好度）。

结果：MACRS 在 task success、recommendation success、turn-level act-choice accuracy 上都超过传统对话推荐器和 plain LLM baseline，尤其 reflection 打开后。Trajectory-level reflection 是最可靠、最一致的改进；turn-level reflection 也有帮助但更不稳定。Case study：baseline 只凭模糊线索就跳去推荐，MACRS 先探询、反思，再落到更相关的选择。Takeaway：planning、act 级反馈、整段对话学习分开，评估用 task success + 人评 —— 不只文本相似度。失败模式：优化流利回复，却从不转化成被接受的推荐。Fix：测 success / 轮数 + 人类接受度，不只流利度。

## 这门课怎么教的（Module 2）

先给失败对比（旧推荐器 vs 单个 LLM）引出拆分动机，再按系统自己的 pipeline 顺序：问题 → 架构/act → 反思 → 证据。单个 running domain（电影/音乐推荐）贯穿，配一段 traced dialogue（追踪对话）对比 baseline vs MACRS。心智模型：dialogue act 当小而显式的动作词汇 + planner / executor 分离。每种机制都配 before / after 例子，好跟。难处：Responder 三个 sub-agent vs Planner vs Seeker，五六个角色名第一遍容易混。最值得偷的：**同一段对话讲两遍 —— 单 agent 一遍、团队一遍 —— 标注到底哪个 act 改变了结果**。要避免：先介绍五个 agent 名，再给一段端到端 turn。

## 自测 4 问（带答案要点）
1. MACRS 为什么拆？什么时候别拆？
   → 一个任务捆几种互相干扰的技能（理解/规划/推荐/表达）；简单 one-shot 推荐别拆。
2. Planner / Responder / Seeker 各干什么？Responder 为什么再拆三个？
   → Planner 选 act；Responder 执行；Seeker 取知识；Responder 拆 dialogue / recommendation / response，各自推理风格 + 工具不同。
3. 两级 reflection 各是什么？改的是什么？
   → turn-level：每轮后检查 act + 回复；trajectory-level：整段后存 compact lesson；改策略，不是下一句话措辞。
4. 为什么评估不能只看流利度？三件套是什么？
   → 流利不等于被接受；offline + simulated user + human judgment；看 success / 轮数 + 人评。
