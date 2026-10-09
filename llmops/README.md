# LLMOps — 笔记

16 lessons，\~3h。把 MLOps 的生产纪律搬到 LLM 上：4D 框架（Discover/Distill/Deploy/Deliver），以 RAG 应用为 running example，覆盖数据工程、prompt 生命周期、推理 infra 经济学、serving 扩缩容、治理。每篇带 4 道自测题。

- [是什么 + 4D 框架](llmops/llmops-01-what-and-4d.md) —— stochastic 的代价、AI gateway、canary/shadow/blue-green
- [Discover + 数据工程](llmops/llmops-02-discover-data.md) —— RAG 管线、scoping 三约束、chunking、检索评估指标
- [Distill：Prompt + 评估](llmops/llmops-03-distill.md) —— prompt 当代码管、四层评估、LLM-as-judge
- [Deploy：安全 + Infra](llmops/llmops-04-deploy.md) —— prompt injection、TTFT/TPS、VRAM 数学、serving 栈
- [Deliver：编排 + 治理](llmops/llmops-05-deliver.md) —— orchestration 铁律、HITL、kill switch、agent/多模态/蒸馏前沿
