# 分析框架（Ch2 L4）：Problem → Evaluation 六步

> 可复用的六阶段框架，面试和真实架构评审都用得上。防两种经典翻车：没框架地东拉西扯（漏关键故障点）、过早开药方（"上 GPT-4+LangChain"之前没搞懂问题）。

## 六阶段（每阶段附"跳过会怎样"）

**1. Problem**：用自己的话重述问题；点名用户、目标、约束。**显式约束**（latency budget、cost ceiling、scale）和**隐式约束**（safety/reliability/trust——没人说但人人默认）分开列。跳过=给错误的问题造了个优雅的解。

**2. Decomposition**：拆成子问题，接口干净；每个子问题问一句"LLM 真增值吗，还是确定性代码更便宜"；标出可复用组件。粒度 trade-off：太粗→故障模式藏大 blob 里；太细→coordination overhead 吃掉你。**在自然接缝处切**——输入输出 crisp 的边界。

**3. Architecture**：摆组件、连接、数据流；选 topology。single agent（紧耦合任务）；multi-agent（manager–worker 或 decentralized handoff，模块化工作）；**collaborative debate**（多个 agent 互辩互 critique 提质量——高风险决策，准头值得多烧算力）。

**4. Control**：系统怎么跑——interaction pattern（ReAct / plan-and-execute / tool-calling loop）、memory 策略（short/long/external 检索即 **RAG**：回答前把相关文档捞进 context）、error handling（backoff 重试、fallback、graceful degradation）、**human oversight 点**（不可逆/烧钱动作先审批）。

**5. Adaptation**：系统怎么越变越好——真实 outcome 的 feedback loops、**reflection** 自改进（agent 自己 critique 输出再改——内置 inner critic）、fine-tuning（在任务数据上继续训练）或 memory write、**drift monitoring**（用户行为/世界变了，性能悄悄掉，要抓到）。

**6. Evaluation**：先定义成功再开工——task success rate、accuracy、latency、cost per task、user satisfaction。建 **benchmarks**（固定回归测试集）、LLM-as-judge（没 ground truth 时）、人工评（重判断的）、**red-team**（主动攻击找故障模式）、A/B 测改动。跳过=上线一个测不出好坏的系统，之后每次改动都是赌博。

## 元 takeaway

框架逼你 problem-first，评审系统化——每阶段有明确要回答的问题和特征性翻车。六个词背下来：Problem → Decomposition → Architecture → Control → Adaptation → Evaluation。

## 自测 4 问（带答案要点）
1. 框架防的两种翻车？各举一例。
   → 没框架分析（漏故障点）、过早开药方（没懂问题先选型）。
2. Decomposition 的粒度 trade-off？在哪切？
   → 太粗藏故障，太细 overhead；自然接缝（输入输出 crisp）。
3. Control 阶段定哪四件事？为什么 human oversight 点重要？
   → pattern/memory/错误处理/人工点；不可逆动作先审批。
4. Adaptation 有哪几条路？drift monitoring 抓什么？
   → feedback/reflection/fine-tuning/memory write；性能随世界变化悄悄掉。
