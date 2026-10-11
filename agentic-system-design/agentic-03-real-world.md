# 真实系统设计（Ch2 L5/L7）：Web Agent + 多 Agent 推荐

> 两个完整 case：multimodal web agent（看网页、点鼠标的 agent）和多 agent 对话推荐。共同主题：verifier 和分工是可靠性的来源。

## 1. Multimodal Web Agent

**Web agent**：感知网页、规划多步交互、像人一样执行（点、输、导航）。**Multimodal** = 同时处理多种输入（页面文本/结构 + 视觉截图）。

为什么难：页面动态（JS 渲染、懒加载）、布局无限变、内容个性化、很多信号纯视觉、站点主动反 bot（CAPTCHA、bot 检测）。

**Perception 四选**（fidelity/robustness/cost 三角）：
- raw HTML/**DOM**（浏览器页面元素的结构树——精确完整但啰嗦，JS 重的页面脆）；
- **accessibility tree**（读屏软件用的语义简化树——角色标签干净，噪声少）；
- screenshot + vision model（看到用户看到的，动态渲染鲁棒，但 token 烧钱）；
- hybrid fusion（混搭：截图看布局 + DOM 精确定位）。

**Action 三档**：DOM/API 级控制（快而准，站点一改版就挂）、UI 级控制（像人一样点按——布局变了也活，慢）、API-mediated（直接调站点后端——最快最稳，但外部 agent 通常没这个权限）。

**参考架构五件套**：perception（建多模态观察）→ planner（LLM 拆子任务、定下一步）→ executor（做 UI 动作）→ **verifier**（每步后检查页面真到了预期状态——**反幻觉检查点**）→ memory（历史动作/页面状态/用户偏好）。planner→executor→verifier→planner 的反馈环：失败就重规划，不瞎往下走——20 步 workflow 不在第 3 步 derail 就靠它。

**可靠性安全**：self-correction loop、每步强制 verify、guardrails + 不可逆动作（下单/提交表单）**用户先确认**、高风险 HITL、安全失败=停下上报不猜。

**生产账**：rate-limit + 拼命 cache（一次页面观察几千 token）；独立子任务并行；golden task 评估集（已知的端到端好场景）；站点改版 drift 监控；浏览器 session 隔离 + 最小权限 credential（agent 被黑也只能祸害自己的任务）。

## 2. 多 Agent 对话推荐系统

**Conversational recommender**：跟用户聊天套偏好再推东西（像先问再带看的销售）。为什么拆多 agent：单 agent 碰 context 上限、工具一多协调不过来；拆开=**故障域隔离**，一个组件烂不污染全局。

**六角色**：
- coordinator/orchestrator：对话状态机，决定下一个调谁；
- conversation/intent agent：解析意图、跟踪偏好 slot 缺啥、detect 歧义问澄清；
- recommendation agent：从 catalog 捞候选、按 fit 排序；
- explanation agent：给每个推荐用户化的理由——**解释建信任，也让用户纠正误解**；
- safety/moderation agent：过滤有害、执行政策；
- context/state manager：跨 turn 的共享记忆（偏好+当前短名单）。

**控制流**：顺序阶段（理解→检索→排序→解释）+ 迭代 refinement loop + specialist 间 handoff + 全局错误处理（retry → 降级 → 安全 fallback）。
**Adaptation 双记忆**：session memory（这轮聊出的）+ 长期偏好学习（显式评分 + 隐式点击/停留）+ reflection（LLM judge 打推荐质量分回流排序）。

**Trade-off**：多 agent 多 latency（串行模型调用累加）、多烧钱（每个 specialist 都计费）、coordination 复杂度；换模块化、好 debug（看得出谁挂了）、可并行开发。缓解：依赖允许处并行 tool call、cache 重复计算、轻活（路由/分类）用小便宜模型。

**防五故障**：preference drift（聊着聊着口味变了还在优化旧信号）、cascading errors（一个 specialist 错下游全错）、agent 间输出冲突、**hallucinated items**（推不存在的商品——verifier pattern 同样适用）、privacy leakage（偏好敏感，scope 好保护好）。

这课有真实开源多 agent 对话推荐框架 + 真实 benchmark 验证——pattern 不是纸上谈兵。

## 自测 4 问（带答案要点）
1. Web agent perception 四选？各适合什么？
   → DOM/无障碍树/截图+视觉/混合；fidelity-robustness-cost 三角。
2. Verifier 在架构里起什么作用？没有它会怎样？
   → 每步确认状态，反幻觉；20 步 workflow 第 3 步 derail 都不知道。
3. 多 agent 推荐为什么拆六角色？代价？
   → 故障域隔离+好 debug；代价 latency/成本/coordination。
4. 防哪五种故障？preference drift 怎么解？
   → drift/级联/冲突/幻觉商品/隐私；持续跟踪偏好 slot+隐式信号。
