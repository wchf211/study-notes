# LLMOps 是什么 + 4D 框架（Ch1–2）

> LLMOps = 把 MLOps 那套生产纪律，搬到 LLM 这种"不确定"的模型上。核心矛盾：同样的输入，模型每次输出可能不一样——没法像确定性代码那样测试。

## 1. 为什么需要 LLMOps

- **MLOps**：把机器学习模型从训练送到可靠生产的整套功夫（版本管理、测试、监控、回滚）。
- **LLMOps**：同一套纪律，适配大语言模型。决定性差异：LLM 是 **stochastic（随机性）** 的——同样输入可能产出不同输出。
- **Hallucination（幻觉）**：模型一本正经地编造错误答案。Demo 里是尴尬，生产里（客服/法律/医疗）是 liability（法律责任）——整个学科就是围绕"管住它"建的。
- 全课的 running example 是一个 **RAG 应用**（客服知识库问答）：不靠模型背知识，检索相关文档塞进 prompt，让答案有据可查。

## 2. 怎么走到 LLMOps 的

2017 transformer → 2018 BERT/GPT → 2020 GPT-3 的 **few-shot learning**（prompt 里给几个例子就会干活，不用重训）→ 2022 ChatGPT 出圈 → 2023 GPT-4 多模态+工具。
关键转折：**in-context learning**（靠 prompt 教，不靠改权重）+ API 化 → 团队不用标注数据、不用 GPU 也能发货。这也是 LLMOps 和 MLOps 的分水岭：**MLOps 多是自己训模型，LLMOps 多是调别人 API**——工作重心从训练变成集成、评估、运维。

参考架构：数据层 → embedding 层（文本转向量，存 **vector database**，专为向量相似搜索优化的存储）→ 检索层 → LLM 推理层 → 评估层，prompt 编排串起来。
**第一天就记日志**：prompt、completion、用户反馈——这些是 feedback flywheel（生产流量持续改进 eval 和 prompt）的燃料。

## 3. MLOps → LLMOps：什么变了，什么没变

没变的：数据版本化、CI/CD、实验跟踪、模型 registry、实时监控。
变了的：
- 数据集管理 → **prompt 管理**（prompt 要版本化！）；
- 重训 → prompt engineering；
- 多了 embedding + 向量库；
- **评估变难一个数量级**：accuracy 那套对自由文本不适用；
- **Red-teaming（红队测试）**：故意用对抗输入攻击自己的系统，变成标配。

关键决策：**AI gateway**（网关）——内部统一服务，路由多模型/多供应商，顺带 fallback（A 挂了切 B）、缓存、key 管理、成本统计。多一个组件要运维，但第一次供应商 outage 就回本。

铁律：**retrieval 和 generation 分开评估**，不然挂了都不知道是哪半边的问题。

## 4. 4D 框架：全课的骨架

- **Discover（发现）**：先定业务用例、用户、成功标准——KPI（answer accuracy、工单解决率）+ 约束，再碰数据。
- **Distill（提炼）**：把原始数据变成可靠行为——切块、embedding、检索、prompt、选模型、评估。
- **Deploy（部署）**：加固——安全、推理 infra、测试、**canary（先导 5% 流量）/ shadow traffic（复制流量跑新系统但不返回）/ blue-green（新旧并行一键切换）**。
- **Deliver（交付）**：运维——监控、反馈闭环、治理、迭代。

两个工程概念：
- **Data contracts（数据契约）**：上下游对 schema/新鲜度/质量的显式约定——上游改个列名，别让检索静默中毒。
- **SLA**：可用性/延迟的合同承诺（如 99.9%、p95 < 2s）——每多一个 9 都是真金白银。

血泪：**大多数生产事故是 Discover 事故**（成功标准模糊、约束没想清楚），不是模型事故。

## 自测 4 问（带答案要点）
1. LLMOps 和 MLOps 的决定性差异？
   → LLM 是 stochastic 的，没法像确定性代码测试；hallucination 是标志性故障。
2. AI gateway 解决什么问题？代价？
   → 统一路由/ fallback/缓存/成本；代价是多一个组件运维。
3. 4D 是哪四个？哪个阶段最容易埋雷？
   → Discover/Distill/Deploy/Deliver；Discover（标准模糊=事故之源）。
4. RAG 挂了先查哪半边？
   → 检索和生成分开评估；先查喂给模型的东西（retrieval 挂得最多）。
