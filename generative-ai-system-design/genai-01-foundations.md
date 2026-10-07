# GenAI 基础（Ch1–2）：直觉版

> 39 课里的地基层：GenAI 系统设计到底在设计什么、transformer/评估/并行/推理优化。每个概念先给直觉，再给面试口径。

## 1. GenAI 系统设计 = 分布式 ML + 系统设计

直觉：传统 ML 系统设计只管"模型上线服务用户"；GenAI 多了一整块——**几千块 GPU 怎么把巨模型训出来**。所以每道题都要拆成两套架构：**训练架构**（training infra）+ **部署架构**（deployment/serving infra），面试官最爱考这两者的接缝（比如训好的权重怎么进 serving、版本怎么管）。

## 2. Transformer 速通（直觉版）

- **Attention 干嘛的**：让每个词在决定自己意思之前，先看一眼全句。Self-attention 公式 Attention(Q,K,V) = softmax(QKᵀ/√d)·V —— 直觉：Q 是"我在找什么"，K 是"我有什么"，点积算相关度，softmax 转成权重，再按权重把 V（实际内容）加权求和。
- **Multi-head**：8 个头 = 8 个不同视角同时看（有的看语法、有的看指代），512 维 / 8 头 = 每头 64 维。
- **为什么干掉 RNN**：RNN 必须一个词一个词串行（还记不住长距离）；attention 全句并行算，一次看全。代价：**attention 计算量随序列长度平方增长**（序列翻倍，计算 ×4）——这是 KV cache 等优化的根因。
- 流程：tokenize → embedding（512 维）→ positional encoding（告诉模型词序）→ self-attention → cross-attention（decoder 看 encoder）→ feedforward + 层归一化。

## 3. 评估指标（每个 modality 考的不一样）

直觉：没有万能指标，**自动指标便宜但瞎，人工指标准但贵**，永远搭配用。

- **BLEU**（翻译）：n-gram 精确率的几何平均 × brevity penalty（防作弊：只输出一个对的词骗高分）。只看"像不像"，不管意思。
- **ROUGE**（摘要）：F1 = 2PR/(P+R)，ROUGE-L 用最长公共子序列。偏召回：看关键内容漏了没。
- **FID**（生成图）：真实图和生成图在特征空间的分布距离，**越小越好**；对 mode collapse（只会画一种图）敏感。
- **Inception Score**：越大越好，但依赖分类器本身靠谱。
- **Perplexity**：模型对正确答案有多"意外"，\~1 = 极自信，>20 = 瞎猜。越小越好。
- **CLIP score**：文本和图像 embedding 的余弦相似度，0.6–1 算对齐得好。
- **人工**：MOS（1–5 打分）、pairwise preference（两个输出哪个好）。
- 面试口径：先说"这个 modality/任务用什么指标"，再主动说"它的盲区是什么"（如 BLEU 奖励表面重合、FID 看不出语义错误）。

## 4. 并行训练（Parallelism）

直觉：175B 参数的 GPT-3，单张 V100 要训 350+ 年 → 1024 张 A100 降到 34 天。并行是把"不可能"变成"贵但可行"。

- **Data parallelism**（主流）：模型复制 N 份，每份吃不同数据，训完同步梯度（AllReduce）。直觉：N 个学生各做一套卷子，对答案取平均。
- **Model parallelism**：模型太大一张卡装不下，按层/按算子切开分给多张卡。
- **Parameter server vs P2P**：参数服务器简单，但是单点故障 + 带宽瓶颈；P2P（Ring AllReduce）去中心化，是 LLM 训练默认选择。
- 三大坑：**容错**（checkpointing，卡会坏）、**异构**（快的卡分多点活）、**straggler**（最慢的那张卡决定整体速度）。
- 面试口径：报 GPT-3 的数字；解释为什么 data parallel + P2P 是默认；straggler/故障怎么处理。

## 5. 推理优化（Inference Optimization）

直觉：训练是"不惜代价训一次"，推理是"每天亿次调用，能省一分是一分"。所有优化都是 **accuracy ↔ latency/cost** 的 trade-off。

- **Quantization（量化）**：FP32 → INT8，直觉：用更粗的刻度记数，模型小 4 倍、算得更快，精度掉一点。
- **Pruning（剪枝）**：把不重要的权重直接删掉，可剪掉 \~90% 参数。
- **Distillation（蒸馏）**：大模型当老师，教小模型学它的输出分布，适合端侧部署。
- **KV caching**：decode 时每步的 K/V 存下来复用，避免重复算（直觉：做数学题把中间结果记在草稿纸上）。
- **Semantic caching**：语义相同的问题直接返回缓存答案（"怎么退货" = "退货流程是什么"），阈值 \~0.95。
- **Batching**：攒一批请求一起算，batch 16–32 常用；吞吐上去了，单个请求延迟可能涨。
- 面试口径：标准配方 = 量化 + KV cache + batching；每说一个优化，主动说"我牺牲了什么"。

## 6. 设计 GenAI 系统的积木

- **Tokenization**：BPE/subword 直觉——高频词整个记（"the"），低频词拆开（"un"+"believ"+"able"），在词表大小和序列长度之间平衡。
- **Embeddings**：词/句变成向量，存在 vector DB 里做语义检索。
- **Acoustic model + vocoder**（语音）：声学模型预测 mel-spectrogram（时频图），vocoder 把它变成波形。两阶段是经典模块化套路。
- **Scene graph**（图像理解）：节点=物体，边=关系，把像素变成结构化语义。

## 自测 4 问（带答案要点）
1. 为什么 GenAI 系统设计题要拆成训练和部署两套架构？
   → 训练是 GPU 舰队+分布式，部署是亿级用户 serving；面试官考两者接缝（权重/版本/数据流）。
2. Attention 计算量随序列长度怎么增长？带来什么后果？
   → 平方增长；长上下文贵 → KV cache、FlashAttention 等优化的根因。
3. Data parallelism 里 parameter server 和 P2P 各有什么问题？
   → PS：单点故障+带宽瓶颈；P2P：去中心化但需要高效集合通信；三大坑：容错、异构、straggler。
4. 推理优化的标准配方是什么？每个的代价？
   → 量化+KV cache+batching（+剪枝/蒸馏看场景）；代价：精度、内存、单请求延迟。
