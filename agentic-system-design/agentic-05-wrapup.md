# 收尾（Ch3）：从 model-centric 到 system-centric

> 综合，无新料。六阶段逻辑在每个 case 里都复现：对话推荐、自主奖励学习、web agent、多物体扩散。

三句 foundation：agent 感知、推理、行动、适应；架构塑造系统行为；可靠性来自规划+控制+反馈，不是某个单组件。

**Headline mental shift：从 model-centric 转 system-centric。** 每个 case 里可靠表现都来自 coordination——专业分工、行动前规划、内置 critic、迭代 refine、有意义的人机交互——不是选了更大的模型。

行业方向：staged workflows 代替单步执行、interactive refinement 代替黑盒生成、feedback-driven loops 代替静态管线。

带走的五个设计问题（新问题先问这五个）：
1. 这需要多步编排吗？
2. 规划应该住在哪？
3. Feedback loop 在哪介入？
4. 错误怎么早发现？
5. 人的角色是什么？

收官：模型会一直进化，**架构设计是留下的东西**——未来属于会推理、会行动、会适应的系统。

## 自测 4 问（带答案要点）
1. 三句 foundation？
   → 感知推理行动适应；架构塑造行为；可靠性=规划+控制+反馈。
2. Model-centric vs system-centric 的区别？
   → 前者赌大模型，后者靠 coordination（分工/规划/critic/迭代/人机）。
3. 行业三个"代替"？
   → staged workflow / interactive refinement / feedback loops。
4. 新问题的五个先问？
   → 多步编排？规划在哪？feedback 介入点？错误早发现？人的角色？
