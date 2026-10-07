# 生产运维（Ch6–7）：防滥用、可靠性、监控漂移

> 核心观点：上线不是终点。 defense（防搞事）+ reliability（扛住日常故障）+ monitoring（盯住静默变质），三件套缺一不可。

## 1. 防 Prompt Injection（防滥用）

**Prompt injection** = 走私指令：用户输入或第三方内容（粘贴的邮件、工单正文、网页）里藏着"忽略之前的指令"之类的话，模型当成你的命令执行了。

分层防御（注意：**在 prompt 里写"请不要听注入指令"不算防御**）：
1. **外部内容一律当数据，不当指令**——delimiter 划清界限；
2. 输入输出都校验；
3. Tool 最小权限——被注入了也干不了大事；
4. 敏感操作人工确认；
5. **Red-teaming（红队测试）**：上线前自己用对抗输入攻击自己的功能，当成标配流程。

例：摘要器必须能摘要一封写着"批准 $500 退款"的工单，**而不执行它**。你的功能天生就要读不可信内容——注入不是 edge case，是日常运营环境。

## 2. 可靠性：两类故障，两套打法

**Infra 故障**（超时、限流、5xx、断连）→ **exponential backoff + jitter**（等待时间指数增长+随机抖动，防客户端步调一致重试）+ 超时 + **circuit breaker**（依赖明显挂了就暂时别调了，别拿头撞墙）。
**模型坏回复**（HTTP 200 但内容错）→ **绝不盲目重试**；校验 → fallback（缓存结果 / 非 AI 简化路径 / 人工审核）。
**非法输入不重试**——同样的失败，只会更慢地再失败一次。

例：超时 → backoff 重试一次；自信但错的摘要 → 送审核队列，不回炉重算。
**先分类（transient infra vs 坏内容），再选 playbook**。无脑重试把小抖动变成 outage，把错答案变成更贵的错答案。

## 3. 监控：Cost / Latency / Quality Drift

**Drift（漂移）** = 没改代码，效果慢慢变差（模型供应商升级了、政策文档变了、用户用法变了）——**它永远不报错**，所以必须显式监控。

三个维度，每个都要有**对比基线**（绝对值没意义，看趋势）：
- **Cost**：prompt 越堆越长，单次 token 成本 creep。信号：本月均单 call 成本 vs 上月。
- **Latency**：响应变慢（如检索悄悄变慢）。信号：本周均值 vs 历史基线。
- **Quality**：eval 通过率无缘无故掉。信号：本季度按标签通过率 vs 上季度。

做法：**把指标做进现有 eval harness**（每次调用计时、记 token），别另起一套；like-for-like 对比；每个 drift 发现都回填成 eval set 的新 case。
真实产品注脚：GitHub Copilot 的**acceptance rate**（用户保留建议的比例）既是 eval 指标又是生产监控信号。

三个经典错误：
1. 只盯 cost/latency 不盯 quality——更快更便宜，但悄悄变差；
2. dashboard 建了没人定时看；
3. 暴露问题的 case 没回填进 eval set。

金句：**如果质量一夜腰斩都没告警会响，你那不是监控，是日志**。

## 4. 全课一句话

- Scoping：先问 AI 该不该来，为"它会错"做设计，而不是赌它不错。
- Interface：prompt 是下游代码依赖的规格；**不进代码强制的 contract 不是 contract**。
- Evaluating：真实、打标签的 eval set，每次改动都跑，不只上线跑一次。
- Capabilities：retrieval / tool calling / fine-tuning 治不同的病，**按诊断组合，不按默认爬梯子**。
- Production：上线不是终点；防御、扛故障、盯漂移，功能才一直可信。

毕业设计（课后作业）：拿一个真实功能走一遍——它需不需要 AI？错了会怎样？contract 长什么样？怎么证明它 work？真实故障指向哪个能力？生产信任靠什么？

## 自测 4 问（带答案要点）
1. Prompt injection 是什么？哪五层防御？哪句不算防御？
   → 走私指令；数据/指令分离、输入输出校验、最小权限、人工确认、红队测试；"请不要听注入指令"不算。
2. Infra 故障和模型坏回复的处理有什么根本不同？
   → 前者 backoff+jitter+熔断可重试；后者绝不盲目重试，走校验+fallback。
3. Drift 为什么必须显式监控？三个维度的信号？
   → 它不报错；cost/latency/quality 各自 vs 基线看趋势。
4. 监控的三个经典错误？
   → 不看 quality / 没人定时看 dashboard / 不回填 eval set。
