# 加餐 14 课（Ch13）：从硬件到 Agent 生产化

> 免费章节，覆盖面最广：硬件选型、训练直觉、音频/视觉基础、采样、diffusion 推理、Agent 架构、语义缓存、tool calling、回归测试。面试金句最多的一篇。

## 1. CPU vs GPU vs TPU

直觉：**CPU = 几个博士**（单核极强，擅长分支判断、任务调度）；**GPU = 几千个小学生**（单核弱，但一起做矩阵乘法无敌，不管分支）；**TPU = 专用计算器**（Google 为张量定制，systolic array，能效最高，但模型支持少、要配套代码）。
- 灵活性 CPU > GPU > TPU；AI 算力 TPU/GPU > CPU。
- 生产分工：**CPU 编排、GPU 训练、TPU 规模化**。
- 面试口径：用"灵活性/性能/成本"三角回答硬件选型。

## 2. LLM 训练直觉（The Training Loop）

LLM 不是背字典，是**学"下一个词最可能是什么"**。自监督：每句话自带标签（输入"Large language models learn from"，目标"data"）。
四步循环：
1. **预测**：输出词表上的概率分布（Paris 60%, London 25%）。
2. **算 loss**：cross-entropy 问"正确答案让你有多意外"——**自信的错误罚最重**。
3. **反向传播**：查是谁的锅（不涉及"理解"）。
4. **更新权重**：学习真正发生的地方（Adam/SGD 控制步子大小）。
- 训练 vs 推理：训练更新权重、算 loss、反向传播、几周几月；推理权重冻结、不算 loss、毫秒级。**上线后的 LLM 不会从用户对话里学习**。

## 3. VAE（一句话版）

经典 autoencoder 把输入压成一个**点**，点之间有空隙 → 采样到空隙就生成乱码。VAE 把输入编码成一个**分布**（一片区域），区域重叠 → latent 空间连续光滑 → 可以采样、插值（猫渐变成狗）。**"分布 vs 点"就是 VAE 能生成的根因**。

## 4. 音频 101

- 采样率 × 位深 = 保真度 vs 存储/算力（语音 16kHz 够，音乐 44.1kHz）。
- **Spectrogram（时频图）**：横轴时间、纵轴频率、颜色深浅是能量。直觉：把声音画成"图"， phoneme 变成形状，CNN 就能学了。
- **两阶段是经典套路**：先预测 spectrogram（好学），再用 vocoder 转波形（Griffin-Lim 有 artifacts，神经 vocoder 如 HiFi-GAN 效果好）。
- Phoneme（音素）："bat"/"pat" 的最小区别单位；G2P 把字转音素，再管重音/语调/时长。

## 5. CLIP：图文之间的语义桥

直觉：文本是离散 token，图像是连续像素，没法直接比。CLIP 用**两个独立编码器**（文本 transformer + 图像 ViT）把图文都映射到同一向量空间，**对比学习**把配对的拉近、不配对的推远 → 语义相似变成几何距离。
- 匹配 = 余弦相似度：≈1 强相关，≈0 无关，负数反义。
- **Zero-shot 分类**：类别写成文字（"a photo of a cat"），embed 后找最近的——**加新类不用重训，加一句话就行**。
- 面试口径：任何要做图文对齐的设计，从 CLIP 开始。

## 6. Diffusion 直觉 + 推理加速

- **直觉**：forward 加噪直到纯噪声；模型学**逆过程（去噪）**。生成 = 从纯噪声开始，一步步"去"出结构。为什么从噪声开始有效？像雕塑家面对大理石——模型学的是"从随机中长出结构"的通用规律，所以能发明新内容。
- **推理太慢怎么办**：naive 要走全部 1000 步（20ms/步 → 20 秒/张，不可用）。
  - **DDIM**：去随机性 → 步数可跳（1000→980→960…），20–50 步，质量小损。
  - **DPM-Solver**：把去噪看成微分方程，用数值解法大步走，**10–20 步**，生产默认，快 5–10 倍。
- 面试背数字：**1000（标准）→ 20–50（DDIM）→ 10–20（DPM-Solver）**，步数直接挂钩 serving 成本。

## 7. VLM（Vision-Language Model）

管线：图片切 patch → ViT 编码 → **projection 层翻译进 LLM 的 embedding 空间** → 插进 token 序列 → 标准 self-attention。**LLM 原样复用**当推理引擎是关键设计选择。Attention 让文本 token 去看视觉 token（"颜色"去看狗毛的 token），回答被图像" grounding "住。

## 8. 采样策略（Sampling）

每步 LM 输出词表概率分布，采样策略决定怎么挑词：
- **Greedy**：永远拿最高的。确定性，适合短事实问答；长文本会复读循环。
- **Beam search**：留 B 个候选序列一起往前走，取总概率最高的。语法稳，适合翻译/摘要；偏保守、贵。
- **Top-k**：留 k 个最高的词，归一化后采样。k=1 就是 greedy；k 大有创意但可能胡话；k 固定太死板。
- **Top-p（nucleus）**：取累积概率 ≥p 的最小词集再采样。**自适应**：模型自信时词集小，不自信时词集大。**对话/创意的默认选择**。
- 面试口径：采样是设计决策——翻译用 beam，聊天用 top-p，抽取用 greedy。

## 9. Agent 生产架构（四层）

生产事故多来自系统设计（状态/护栏/可观测），不是模型不行：
1. **Orchestrator**：空中交通管制，管访问控制、幂等（工具没它不能写 memory）。
2. **Planner**：目标 → DAG 子任务。static（快、可预测、不灵活）vs dynamic（LLM 逐步重规划，灵活但贵）→ **生产用 hybrid**。
3. **Memory**：短期（当前任务/DAG 位置）vs 长期（偏好/历史）；vector DB（语义，10–100ms）vs KV（精确，1–10ms）vs hybrid。**上下文窗口的淘汰/摘要是核心设计**。
4. **Tools**：registry（schema、限流、鉴权）；容错：circuit breaker、指数退避+jitter、超时预算、幂等键。
- **金句**：幂等 + reflection（每步前检查）+ 可观测性，是 demo agent 和 production agent 的分水岭。

## 10. 语义缓存（Semantic Caching）

直觉：精确匹配缓存认不出"怎么退货"和"退货流程是什么"是同一个问题，白白烧 LLM 钱。语义缓存按**意思**匹配：
- 写：LLM 回答的同时 embed 问题，存 (embedding, query, response)。
- 读：embed 新问题，ANN 找相似度 ≥ 阈值的缓存。
- **阈值是最关键的参数**：0.98 准但命中低；0.85 命中高但会答错（"reset my password" vs "reset my router"）。生产从 **0.95** 起步。
- 层级：L1 精确匹配 → L2 语义（相同问题零 embedding 成本短路）。
- 数字：$0.01–0.03/次推理；50% 命中率 → 省 30–70% LLM 成本。
- 多租户要 namespace 隔离（防串数据）；PII 加密。

## 11. Tool Calling 架构

- **Schema 就是 contract**：名字要动词化（search_flights）、description 是最重要的字段（决定选型准度）、参数 typed + 约束。大 registry 按需动态加载（全塞进 context 又贵又降准度）。
- 模式：单轮（延迟最低）→ 串行 ReAct（call→观察→推理→call，延迟线性涨）→ 并行（独立调用一起发）。
- 错误恢复：指数退避重试（小心非幂等副作用）、fallback 工具、LLM 自纠正（喂回 error，小心死循环）、circuit breaker。
- 安全：schema 校验、allowlist、沙箱、高风险人工 gate、per-tool 限流（防幻觉循环烧钱）。
- 面试金句：**"Schema design is the foundation"**——每个 tool call 都是潜在攻击面和故障点。

## 12. GenAI 回归测试

LLM 输出非确定，传统 exact-match 测试失效（升级提了速，单元测试全过，摘要质量悄悄掉 15%）。
- **Golden responses**：专家验证的语义锚点，比相似度不比 exact match。
- 数据集：500–1000 个精心设计的 case > 10,000 个烂 case；覆盖 edge/adversarial/分布代表性；随模型/ prompt 版本化。
- 评分：lexical（BLEU/ROUGE，快但瞎）+ semantic（BERTScore/cosine）+ task-specific（幻觉率）+ LLM-as-judge（要用人校准 bias）。
- 阈值示例：BERTScore 跌 >2% 或幻觉涨 >0.5% 就 block 发布。
- 统计严谨：每输入生成 3–5 次，置信区间 breach 才报警（防误报）。
- CI/CD：smoke（50–100 case，每次提交）+ 全量（1000+， nightly）；embedding 一升级先重 embed 全库。
- 面试一句话：**"语义级评估 + 统计严谨 + drift 感知的 CI/CD"**。

## 13. Agentic vs Generative：光谱不是对立

| | Generative | Agentic |
|---|---|---|
| 控制流 | 单次 prompt→response | 目标→规划→执行→评估循环 |
| 状态 | 无状态 | 持久工作记忆 |
| 工具 | 可选单次 | 动态选择+链式 |
| 故障恢复 | 调用方重试 | 自纠正/重规划 |
| 延迟/成本 | 单次推理 | 多次调用叠加 |

- 选 agentic：多步推理要验证（代码生成+测试）、目标模糊（"帮我规划假期"）、错误代价高（金融/医疗）。
- 选 generative：单次变换（摘要/翻译）、延迟敏感、查询分布重复（缓存毫秒命中）。
- **大多数系统从 generative 起步演进**：把 generative 地基打好（模块化缓存、干净 API、可观测管线），以后加 agentic 不用重写。

## 自测 4 问（带答案要点）
1. 上线后的 LLM 会从用户对话中学习吗？为什么？
   → 不会；推理时权重冻结、不算 loss、不反向传播。
2. Diffusion 推理的三档步数和对应技术？
   → 1000（标准）→ 20–50（DDIM）→ 10–20（DPM-Solver，生产默认）。
3. 语义缓存阈值为什么是最关键的参数？生产起点？
   → 决定"省钱"和"答错"的边界；0.95 起步，按 domain 实测调。
4. 什么时候用 agentic、什么时候用 generative？
   → 多步/模糊目标/高代价错误 → agentic；单次变换/延迟敏感/可缓存 → generative。
