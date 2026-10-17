# Applied AI Engineer — 笔记

39 lessons，11 modules，\~5h。主线是把 LLM 包成一个能跑、能测、能运维的生产系统：prompt → RAG → agent → fine-tune → eval → deploy → safety。阅读说明：这门课的 digest 是 high-level 版 —— 基于每课标题、开头、结构和可见内容提炼，不是逐段全文深读；如需逐段细节，仍需一次 full-text 深读。重叠说明：不少内容是通用 LLM / eval 知识，和已有笔记重叠；本笔记把笔墨放在非重叠部分（API 微服务、成本/延迟预算、决策树、RAG 分阶段排障、eval 进 CI/CD）。每篇带 4 道自测题。

- [Foundations + Prompts](applied-ai-engineer/aae-01-foundations-prompts.md) —— 角色定位、LLM 直觉、选型、成本/延迟预算、生产级 prompt
- [RAG + Agents + Fine-tuning](applied-ai-engineer/aae-02-rag-agents-finetune.md) —— RAG pipeline 与排障、agent、prompt vs RAG vs 微调决策
- [Eval + Deploy + Multimodal + Safety](applied-ai-engineer/aae-03-eval-deploy-multimodal-safety.md) —— 评估、API 包装、缓存/限流/可观测、CI/CD、多模态、Responsible AI
- [Teaching Lens](applied-ai-engineer/aae-04-teaching.md) —— 这门课的讲解组织方式 + 值得偷的讲法
