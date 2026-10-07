# AI Product Engineering — 笔记

短课（21 lessons，\~3h）。LLM 功能的产品工程：从 scoping、prompt contract、eval 体系，到按需取用 RAG/tool calling/fine-tuning，再到生产防御与漂移监控。Practitioner 视角，每篇带 4 道自测题。

- [生命周期 + Scoping](ai-product-engineering/aipe-01-lifecycle-scoping.md) —— 五个职责、闭环 lifecycle、三个门禁问题、failure boundary
- [Prompt 即 Contract](ai-product-engineering/aipe-02-prompt-contract.md) —— contract 五件套、四种技巧、结构化输出的代码校验
- [评估体系](ai-product-engineering/aipe-03-evaluation.md) —— 真实 eval set、打分阶梯、五问诊断法、防回归
- [按需取用能力](ai-product-engineering/aipe-04-capabilities.md) —— retrieval gap、tool 安全、bounded agent、fine-tuning 时机
- [生产运维](ai-product-engineering/aipe-05-production.md) —— prompt injection 防御、两类故障、drift 监控
