# 微调 taxonomy（M2 L3）：non-parametric vs parametric 全家桶

> 定制模型的两种哲学：不动权重（知识走 prompt/外部来）vs 动权重（全动还是巧动）。课里的 punchline：**默认 PEFT/LoRA**，除非数据又大又 alien。

## 1. Non-parametric：不动权重

知识在推理时经 prompt/外部存储进来，模型原封不动。

- **In-context learning**：任务指令+例子直接塞 prompt，模型模仿 pattern。最简单，零技术门槛。**死法**：context window 有限，例子一多挤掉指令和其他内容，信息丢失。
- **RAG**：冻住的模型 + 外部知识库，每 query 现搜——给模型配了个外部记忆。最高效的不动参定制之一。**死法**：知识库一大，每 query 搜索变慢 + 信息过载导致误读出错。

## 2. Parametric：动权重

在你数据上真更新权重，分全动和巧动。

**Full fine-tuning**：每层每个参数全重训。**什么时候要**：数据大且和预训练分布很不一样（如单语模型转多语、医疗/法律 domain）。代价：算力黑洞、catastrophic forgetting、数据小时 overfit。

**PEFT**（只训选中/新增参数，冻结其余；便宜+保知识），三分：

**Partial（selective）微调**：冻结存通用 pattern 的浅层，只训后面任务相关的层。
- **BitFit**：只训 bias 项（每层加的小向量）——小到离谱，但在中小数据上接近全微调精度；
- **LT-SFT**（lottery-ticket sparse）：先全微调一次看哪些权重变化最大，取 top-k 加 binary mask 只训它们——强，但要先付一次全微调的钱才找得到 mask；
- 还有 Diff Pruning、Threshold-Mask。
- Trade-off：省资源大，但复杂任务适应弱、有点 overfit 风险、轻微残留遗忘。

**Additive 微调**：base 全冻，外挂新的可训模块（"adapters"）。
- **prompt tuning**：可学习的 prompt token 拼到输入上，base 不动；
- **prefix tuning**：可学习的"soft prompts"插到 attention 层的 hidden states，经 feed-forward 参数化；
- 还有 P-tuning、LST、MPT、PASTA。
- Trade-off：快、预训练知识全保留；但模型变大变复杂、adapter 可能过拟合到你的任务。

**Reparameterized 微调**：把权重换个更便宜的等价形式，只训换出来的部分。
- **LoRA**：原权重用两个小 low-rank 矩阵近似，只训它们，原权重冻结；
- **QLoRA**：LoRA + 量化；
- KronA（Kronecker-product 分解代替 low-rank）、DyLoRA、AdaLoRA。
- Trade-off：效率+保知识的最佳平衡；但复杂数据集可能 underfit。

## 3. 选型 checklist

看：模型大小、任务复杂度、可用算力、部署约束、预训练知识要保多少。

课里的对照（定义/用例/局限）：
- full：大数据+分布差异大（贵、遗忘、小数据 overfit）；
- partial：任务接近预训练+数据预算中等（适应有限、over/underfit 风险）；
- additive：多任务/快速适应+资源紧（模型变大、overfit 风险）；
- reparameterized：效率+保知识最平衡（复杂 pattern 可能抓不住、underfit 风险）。

**实操**：默认 PEFT/LoRA；数据又大又 alien 才考虑 full；如果需求本质是"查我们文档回答"——**RAG 优先于一切微调**。

## 自测 4 问（带答案要点）
1. Non-parametric 两种？各死在哪？
   → in-context（context 挤爆）、RAG（库大变慢+误读）。
2. PEFT 三分？各举一代表？
   → partial（BitFit）、additive（prompt tuning）、reparameterized（LoRA）。
3. BitFit 为什么神奇？LT-SFT 的代价？
   → 只训 bias 就接近全微调；要先全微调一次找 mask。
4. 什么时候 full fine-tuning？什么时候 RAG 优先？
   → 数据大+分布差异大；需求是"查文档回答"时。
