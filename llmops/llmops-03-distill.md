# Distill：Prompt 生命周期 + 输出评估（Ch4）

> Prompt 是行为的最大决定因素，也是最难 debug 的——因为它静默漂移。所以：当代码管。

## 1. Prompt 当代码管

- **版本化**：进 git、review、每次输出可追溯到 prompt 版本。
- 解剖：system prompt（角色/规则）+ 任务指令 + 检索到的 context + **few-shot examples**（输入输出示范对）+ 输出格式 spec（如 JSON）。
- 技巧：zero-shot（纯指令）、few-shot、**chain-of-thought**（让模型逐步推理，多步任务准，但烧 token + 延迟）、structured outputs、delimiter（指令和不可信内容划界）。
- **Temperature**：采样随机性旋钮——事实/抽取任务 \~0，创意任务调高。和 top-p 一起，**是版本化配置，不是随手拧的**。
- 生命周期：起草 → golden 集上评估 → 带版本号上线 → 监控 → 迭代。
- **Guardrails 放代码里**（输入输出校验、schema 检查、blocklist），别只写 prompt 里。

铁律：**A/B 测 prompt 改动；生产 prompt 不改不跑回归 eval；每次 completion 记 prompt 版本**。

## 2. 输出评估：分层，不是抽查

随机抽查是 theater（演给自己看）。四层：
1. **确定性单元测试**：格式合规、拒绝行为、PII 不泄露；
2. **Golden 数据集**：每次 prompt/模型改动重跑，当回归 suite；
3. **LLM-as-judge**：强模型按 rubric 打分——快、可扩展，但**judge 有 bias，要用人校准**；
4. **人工评估**：管 nuance 和安全。

指标双轨：质量（faithfulness、relevance、correctness、toxicity）+ 运维（latency、单 query 成本）。
**先建 eval harness，再出事故**——每次事故的 failing case 变成永久回归测试。生产流量采样送人工看，**向低置信/高风险倾斜**。
最便宜的标注数据：**用户反馈**（顶/踩、纠正）回填 eval 集。
**No eval suite, no deploy**——评估是每次发布的门禁。

## 自测 4 问（带答案要点）
1. Prompt 为什么当代码管？哪三件事？
   → 最大行为决定因素+静默漂移；版本化/review/输出可追溯。
2. Temperature 是什么？为什么版本化？
   → 采样随机性；0=事实，越高越发散；随手拧=不可复现。
3. 评估四层？LLM-as-judge 的坑？
   → 单元测试/golden/LLM judge/人工；judge bias，要用人校准。
4. 事故和 eval 的关系？
   → 每个 failing case 变成永久回归测试；用户反馈回填 eval 集。
