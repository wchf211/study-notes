# 评估进阶（Ch3）：公式指标的边界、prompt 即 infra、guardrail vs evaluator

> 三个"别搞混"：BLEU 别当质量分、prompt 别当字符串、guardrail 别当 evaluator。

## 1. 为什么相似度指标挂了

**BLEU/ROUGE**：数词重叠的公式分，为机器翻译/摘要设计的（输出紧贴原文）。对 LLM 评估失效：
- 测的是**表面相似不是对错**：错误答案抄了参考答案的词→高分；正确答案换个说法→低分（paraphrase 问题）；
- 假设有唯一"标准答案"——开放任务答案本来就多个。

它们便宜、确定、眼熟，诱惑大，但**优化词重叠=优化错的东西**。
合法窄用：关键词抽取、精确 API/JSON 输出、去重、便宜的第一遍 triage。**永远不当开放质量的裁判**。

## 2. Prompt 是 infra，不是字符串

Prompt = 版本化、有 owner、可 review 的 artifact（和代码/配置一个待遇）。
- 每次改动跑 eval suite（**prompt CI**）；
- 每条 trace 记 prompt 版本——失败能归因到改动；
- 三层分开评：system prompt（稳定规则→单元测试）、task prompt（任务指令→judge/人工）、dynamic context（运行时注入→grounding 检查：输出真用了它吗）。

例：support prompt 改了一个词，全对话语气变了——**eval suite 在 prompt edit 上跑，抓到了**。
一句话：prompt 改动没有 eval 门禁，你就永远不知道哪次改动搞坏了什么。

## 3. Guardrail vs Evaluator：别混用

- **Guardrail（护栏）**：实时、in-path，出事前拦。必须**保守**——只拦 unambiguous 的违规（违禁内容、畸形 tool 调用）。每个误拦直接伤害用户。
- **Evaluator（评估器）**：多是离线，量质量、指导开发。要**丰富灵活**，不当裁判。

决策规则：**只有"放行的代价 > 误拦的代价"且失败无歧义时，才进实时路径**；其余全离线。
反例：拿模糊的质量 judge 当实时 blocker——又慢又误杀，用户跑光；同样问题放离线，本来可以无痛调好。
**离线的质量分，单独验证前不许晋升成实时拦截**。

## 自测 4 问（带答案要点）
1. BLEU 高分为什么可能是坏事？合法用处？
   → 测词重叠不测对错；错答案抄词得高分。合法：精确输出/去重/triage。
2. Prompt CI 是什么？trace 为什么要记 prompt 版本？
   → 改动必跑 eval；失败归因到具体改动。
3. Guardrail 和 evaluator 的三区别？
   → 实时拦 vs 离线量；保守 vs 丰富；误拦伤用户 vs 信息指导开发。
4. 什么情况下评估可以进实时路径？
   → 失败无歧义 + 放行代价 > 误拦代价；离线分晋升要单独验证。
