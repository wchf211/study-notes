# ML 面试 I：Intro + 实用 ML 概念 — 提炼笔记（Ch1–Ch2，2026-10-06）

> Grokking the Machine Learning Interview。自己的话总结，非原文搬运。MLE 面试视角：考官考的是 applied judgment，不是背定义。

## L1. 这门课帮你过什么样的面试
- 大厂 ML 面试有专门的 ML system design 轮：把 ML 基础（决策树、transformer、XGBoost）用到真实问题上。
- 策略（meta 但重要）：先澄清问题再设计；**出声讲推理过程**；根据面试官反馈迭代。

## L2. 搭一个 ML 系统
- 管线：原始数据收集 → 预处理 → 特征工程 → 训练 → serving。
- 数据源：众包标注、用户行为日志、公开数据集。
- 预处理：scaling/归一化、one-hot、feature hashing、缺失值、去离群点。特征选择：filter / wrapper / embedded。**类别不平衡**：oversample 少数类 / undersample 多数类 / SMOTE（少数类 <\~1% 时要处理）。
- 切分：train/val/test + 交叉验证。指标目录：precision/recall/F1、PR 曲线、ROC AUC、**NDCG**、mAP（排序）；RMSE、MAPE、adj R²（回归）；Rand/silhouette/FID/BLEU（niche）。
- 三个必讲的 trade-off：
  - **业务对齐**：先澄清产品目标，选能推动业务目标的指标，别在真空中优化 accuracy；
  - **成本 vs 精度**：深网 99% accuracy 的 serving 成本可能是 XGBoost 96% 的 **4–5 倍**——按 latency/budget 选，不按 leaderboard 选；
  - **train/serve skew**：两边特征逻辑必须同一份代码。

## L3. 性能与容量
- 三种 serving 模式：**实时**（同步 request-response）、**近实时**（流式/事件驱动）、**异步/离线**（micro-batch）。
- 延迟看 **p99/p999** 不看均值；SLA 例：搜索 \~300ms 其中 rerank 占 \~100ms。
- 容量公式：**QPS × 单查询耗时 ≈ 所需服务器数**（500 QPS × 0.4s ≈ 200 台）——面试官必考的心算。
- 存储：SQL 管结构化查询，NoSQL KV/document 管在线特征快查。CAP：分区容忍是必选项，真实系统选 AP 或 CP。算力：CPU 通用 serving，GPU/TPU 给 DL 推理/训练。
- 口径：batching 提吞吐但伤延迟；重排序这种重活要拆出独立的 latency slice。

## L4. 训练数据怎么来（"没有数据怎么办"必考）
- 四路 often 组合：**人工标注**（in-house/外包/众包；annotation guideline、**inter-annotator agreement**、Cohen's kappa、**active learning** 只标信息量最大的、**data programming/weak supervision** Snorkel 式启发函数投票）；**用户行为数据**（implicit 信号，免费但 popularity skew + feedback loop）；**自监督**（BERT mask、GPT next-word、simCLR 对比、VAE）；**公开数据集**（ImageNet \~1400 万）。
- 配套手段：GAN/diffusion/LLM 合成数据、迁移学习、augmentation（几何/颜色/crop、mixup/cutmix）、**reservoir sampling**（流式采样，第 i 个保留概率 k/i）、存储（blob、Parquet 列式）。
- Unfold 里的定义：**parameters**（训练学到的，如神经网络权重）vs **hyperparameters**（训练前定的，如学习率、epoch 数）。
- Trade-off：众包便宜但噪声大，专家准但慢贵；weak supervision 拿精度换量。

## L5. 在线实验（最高频考区之一）
- A/B：control vs treatment，随机化单元通常是 user；小心 **novelty/primacy effect**。Bucket：hash(user_id, salt) % 100。**AA 测试**验 harness 本身。Interference（网络效应）破坏独立性假设。
- 假设检验：p < 0.05、95% CI、**power 80%**、MDE 定样本量。**Interleaving**（team-draft）给排序问题省流量。**Bandits**（epsilon-greedy、UCB、Thompson sampling）持续探索-利用平衡。
- Guardrail 指标防回归；**Simpson's paradox**（整体和分群结论反转）必查分群。
- 口径模板："怎么在线验证模型改动"→ 定 success + guardrail 指标 → 选随机化单元 → 按方差/MDE 定样本量 → 跑完不许偷看 → 查分群。排序题点名 interleaving，自适应分配点名 bandit。

## L6. Embeddings
- 稠密低维向量表语义相似，替代稀疏 one-hot。Word2Vec（CBOW 上下文猜词 / skip-gram 词猜上下文）、GloVe、fastText（subword，抗拼错/生僻）。维度 **100–300**，BERT 768。相似度：cosine / dot product。
- **Two-tower**：user 和 item 各一塔，dot product + **ANN**（FAISS、ScaNN、HNSW）做候选召回 → **retrieve-then-rank** 两阶段是标准答案。
- 训练技巧：**negative sampling**、**triplet loss** L = max(d(a,p) − d(a,n) + margin, 0)。冷启动靠 content-based embedding。存 vector DB + ANN 索引。
- Trade-off：维度大表达强但吃内存拖 ANN；预训练省数据但可能 domain mismatch。

## L7. 迁移学习（"标注数据很少怎么办"的标准答案）
- 两模式：**feature extraction**（冻 backbone 只训分类头——便宜、稳）vs **fine-tuning**（解冻部分/全部层继续训，学习率通常小 \~10 倍）。
- 实践：冻 early layers（学的是通用特征），fine-tune 后面层；**discriminative LR**（逐层衰减）；gradual unfreezing。
- 风险点名：**catastrophic forgetting**（洗掉预训练知识）、**negative transfer**（domain 差太远越迁越差）。相关：domain adaptation、multi-task、few/zero-shot。

## L8. 模型调试（"上线后指标掉了"必考）
- 两阶段：**v1 快速上线**——AUC 0.70 打败规则基线 0.68 就发版，用 live traffic 迭代，别在离线调参上磨；然后 failure-case 驱动迭代。
- Debug 清单（按顺序）：**train/serve 特征 skew**（离线 7 天窗口 vs 线上 30 天）→ **分布漂移**（Wikipedia 训的 entity linking 到论文流量上挂；12 月 holiday query 训的搜索 1 月挂——季节性）→ **过拟合**（hidden test 集从不参与调参；test 分布要贴 live 流量）→ **欠拟合**（模型太简单，加高阶特征或上神经网络）。
- 改进：从失败 case 出发，加缺的特征 / 给弱 slice 加训练样本。多组件系统：**先定位哪个组件贡献了大部分失败**（如 80% 搜索失败在候选召回层），再修它。

## 自测 4 问（带答案要点）
1. 99% 的深网 vs 96% 的 XGBoost，怎么选？
   → 看 latency/budget：深网 serving 贵 4–5 倍；先对齐业务指标，不为 leaderboard 优化。
2. 上线后指标掉了，debug 清单顺序？
   → train/serve 特征 skew → 分布漂移（含季节性）→ 过拟合（hidden test 集、分布对齐）→ 欠拟合。
3. 没有标注数据，训练集怎么做？
   → 混合：public/pretrained 冷启动 → weak supervision/众包放量 → active learning 花预算；点名 bias（popularity skew、annotator 分歧）+ 缓解。
4. Triplet loss 公式？Two-tower 用在哪？
   → L = max(d(a,p)−d(a,n)+margin, 0)；user/item 检索场景，双塔 + ANN 召回，走 retrieve-then-rank。
