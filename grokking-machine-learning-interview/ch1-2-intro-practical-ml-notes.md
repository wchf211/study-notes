# ML 面试 I：Intro + 实用 ML 概念 — 提炼笔记（Ch1–Ch2，2026-10-06）

> Grokking the Machine Learning Interview。MLE 面试视角：考官考的是 applied judgment，不是背定义。

## L1. 这门课帮你过什么样的面试
- 大厂 ML 面试有专门的 ML system design 轮：把 ML 基础（决策树、transformer、XGBoost）用到真实问题上。
- 策略（meta 但重要）：先澄清问题再设计；**出声讲推理过程**；根据面试官反馈迭代。

## L2. 搭一个 ML 系统
- 管线：原始数据收集 → 预处理 → 特征工程 → 训练 → serving。
- 数据源：众包标注、用户行为日志、公开数据集。
- 预处理：scaling/归一化、one-hot、feature hashing、缺失值、去离群点。特征选择：filter / wrapper / embedded。**类别不平衡**：oversample 少数类 / undersample 多数类 / SMOTE（少数类 <\~1% 时要处理）。
  - 直觉 **feature hashing**：把任意字符串特征（如 user_id、URL）用 hash 函数直接映射到固定长度的向量槽位，不用事先建词表。为什么需要：线上随时会来没见过的 ID，词表会爆炸还跟不上；hash 让维度固定、内存可控。小例子：user_id="u12345" → hash % 10000 落到槽位 7319；代价是偶尔碰撞（两个 ID 挤进同一槽），实践中可接受。
  - 直觉 **SMOTE**：对少数类样本做"插值"造新样本——在特征空间里取一个少数类点和它的近邻，连线中间随机取点当新样本。为什么需要：纯复制少数类（oversample）会让模型死记那几个点；SMOTE 造出"附近的新点"，泛化更好。小例子：欺诈样本只有 100 条，在两条欺诈样本之间插值造出第 101 条"长得像但不完全一样"的欺诈。
  - 直觉 **filter / wrapper / embedded**：特征选择的三档自动化程度。filter 先算统计量（如相关系数）筛一遍，和模型无关、最快；wrapper 把特征子集一个个拿去训模型看效果，最准但最贵；embedded 是训练自带选择（如 L1 把不重要的权重压成 0）。小例子：1000 个特征先用 filter 砍到 200，再用 L1 训出最终 50 个。
- 切分：train/val/test + 交叉验证。指标目录：precision/recall/F1、PR 曲线、ROC AUC、**NDCG**、mAP（排序）；RMSE、MAPE、adj R²（回归）；Rand/silhouette/FID/BLEU（niche）。
  - 直觉 **NDCG**：排序指标，核心思想是"越相关的文档排得越靠前越好，且位置越靠后折扣越狠"（分母 log(位置+1)），再除以"理想排序"的得分做归一化，跨 query 可比。为什么需要：accuracy 只管分对分错不管顺序，搜索里第 1 名和第 10 名天差地别。小例子：两份排序都召回 3 篇相关文档，A 放 1、2、3 位，B 放 8、9、10 位——NDCG 给 A 高分，precision 给它俩打平。
- 三个必讲的 trade-off：
  - **业务对齐**：先澄清产品目标，选能推动业务目标的指标，别在真空中优化 accuracy；
  - **成本 vs 精度**：深网 99% accuracy 的 serving 成本可能是 XGBoost 96% 的 **4–5 倍**——按 latency/budget 选，不按 leaderboard 选；
  - **train/serve skew**：两边特征逻辑必须同一份代码。

## L3. 性能与容量
- 三种 serving 模式：**实时**（同步 request-response）、**近实时**（流式/事件驱动）、**异步/离线**（micro-batch）。
- 延迟看 **p99/p999** 不看均值；SLA 例：搜索 \~300ms 其中 rerank 占 \~100ms。
  - 直觉 **p99/p999**：p99 是"99% 的请求比它快"，也就是最慢的那 1%。为什么需要：均值会被大多数快请求稀释，藏起长尾——用户记住的恰恰是那几次卡顿。小例子：均值 50ms 看着很好，但 p99=800ms 意味着每 100 次搜索就有 1 次转圈近 1 秒，SLA 就卡这个。
- 容量公式：**QPS × 单查询耗时 ≈ 所需服务器数**（500 QPS × 0.4s ≈ 200 台）——面试官必考的心算。
- 存储：SQL 管结构化查询，NoSQL KV/document 管在线特征快查。CAP：分区容忍是必选项，真实系统选 AP 或 CP。算力：CPU 通用 serving，GPU/TPU 给 DL 推理/训练。
- 口径：batching 提吞吐但伤延迟；重排序这种重活要拆出独立的 latency slice。

## L4. 训练数据怎么来（"没有数据怎么办"必考）
- 四路 often 组合：**人工标注**（in-house/外包/众包；annotation guideline、**inter-annotator agreement**、Cohen's kappa、**active learning** 只标信息量最大的、**data programming/weak supervision** Snorkel 式启发函数投票）；**用户行为数据**（implicit 信号，免费但 popularity skew + feedback loop）；**自监督**（BERT mask、GPT next-word、simCLR 对比、VAE）；**公开数据集**（ImageNet \~1400 万）。
  - 直觉 **inter-annotator agreement / Cohen's kappa**：两个标注员对同一批数据打标的一致程度，kappa 还扣掉了"瞎猜也能蒙对"的部分。为什么需要：人都不一致的任务，模型不可能学好——IAA 低说明 guideline 写得烂，先修 guideline 再谈模型。小例子：10 条评论里两人有 8 条一致，kappa \~0.7 算可用；kappa 掉到 0.4 以下就别训了，回去改标注手册。
  - 直觉 **active learning**：不随机抽数据去标，而是让模型挑"它最拿不准"的样本给人标。为什么需要：标注预算有限，标 10 万条简单样本不如标 1 万条边界样本。小例子：分类器对概率 0.5 附近的样本最纠结，优先把这批送去标注，标 1 万条顶随机标 5 万条。
  - 直觉 **weak supervision（Snorkel 式）**：写一堆粗糙的启发式规则（labeling function）给数据投票打"弱标签"，再用统计方法去噪合并。为什么需要：一条规则 10 分钟写完，1 小时能标 100 万条——拿精度换量。小例子：规则 1"含'免费赚钱'判 spam"，规则 2"发件人在通讯录判 ham"，两条规则投票，Snorkel 学出每条规则的可信度再合并。
  - 直觉 **reservoir sampling**：流式数据里等概率抽 k 个样本，一遍过、内存只留 k 个。为什么需要：日志流无限长不可能全存，又想要无偏样本。小例子：要留 1 万条，第 100 万条到来时以 1万/100万 的概率替换掉蓄水池里随机一条——最终每条被留下的概率都相等。
- 配套手段：GAN/diffusion/LLM 合成数据、迁移学习、augmentation（几何/颜色/crop、mixup/cutmix）、**reservoir sampling**（流式采样，第 i 个保留概率 k/i）、存储（blob、Parquet 列式）。
- Unfold 里的定义：**parameters**（训练学到的，如神经网络权重）vs **hyperparameters**（训练前定的，如学习率、epoch 数）。
  - 直觉 **parameters vs hyperparameters**：parameters 是模型从数据里"学"出来的（权重），hyperparameters 是你开训前"定"的（学习率、层数）。为什么要区分：前者靠梯度下降自动调，后者得你手动或搜索调——"调参"调的是后者。小例子：权重矩阵是 parameters；你设学习率 0.001、训 10 个 epoch，这些是 hyperparameters。
- Trade-off：众包便宜但噪声大，专家准但慢贵；weak supervision 拿精度换量。

## L5. 在线实验（最高频考区之一）
- A/B：control vs treatment，随机化单元通常是 user；小心 **novelty/primacy effect**。Bucket：hash(user_id, salt) % 100。**AA 测试**验 harness 本身。Interference（网络效应）破坏独立性假设。
  - 直觉 **novelty / primacy effect**：新功能上线初期，用户因为"新鲜"多点两下，指标虚高；跑久了新鲜感过去又掉回去。为什么需要知道：A/B 只跑 3 天可能全是新鲜感红利，结论不可信。小例子：按钮改了个颜色，前 3 天 CTR +15%，两周后回到 +2%——前者是 novelty，后者才是真实 lift。
  - 直觉 **AA 测试**：两组都不给新功能（A vs A），按理指标应该没差。为什么需要：先验证分流系统本身没 bug——如果 AA 都显著，说明 bucket、hash 或指标口径有问题，AB 结果全不可信。
- 假设检验：p < 0.05、95% CI、**power 80%**、MDE 定样本量。**Interleaving**（team-draft）给排序问题省流量。**Bandits**（epsilon-greedy、UCB、Thompson sampling）持续探索-利用平衡。
  - 直觉 **power 80% / MDE**：power 是"真有效果时能检测出来的概率"，MDE（最小可检测效应）是你想抓住的最小提升。为什么需要：先定"我要抓住 2% 的提升"，才能反推要多少样本、跑多少天——否则跑完了才发现流量根本不够。小例子：日活 100 万、基线转化 5%，想抓 +2% 相对提升，power 80% 下大概要跑两周；流量减半就得跑四周。
  - 直觉 **interleaving**：把 A、B 两个排序结果像洗牌一样交错成一份展示，看用户点谁的多（team-draft：两队轮流从各自榜单"选秀"拼一份）。为什么需要：传统 A/B 要两拨用户各看一份榜单，排序实验极费流量；interleaving 让同一批用户同时贡献信号，省一个量级。小例子：A 榜单 [a1,a2,a3]、B 榜单 [b1,b2,b3]，交错成 [a1,b1,a2,b2…]，用户点了 b1 和 a2——B 的首位赢了。
  - 直觉 **bandits（epsilon-greedy / UCB / Thompson sampling）**：在线"边试边学"的分流策略。epsilon-greedy 最直白：90% 时间用当前最优，10% 随机试；UCB 给"试得少"的选项加探索分，逼自己别过早下结论；Thompson sampling 按每个选项"是最优的后验概率"去抽，天然平衡探索利用。为什么需要：A/B 是先试后定，bandit 试的同时就把流量倾斜给赢家，亏得少。小例子：三个推荐策略，Thompson sampling 每天按"各自最优的概率"分流量，烂策略自动饿死，不用等两周才下线。
- Guardrail 指标防回归；**Simpson's paradox**（整体和分群结论反转）必查分群。
  - 直觉 **Simpson's paradox**：整体看 A 赢 B，分群看 B 处处赢 A——因为流量配比在捣乱。为什么需要：只看整体会被"哪个组人多"带偏，必须分群验。小例子：新策略在新用户上 +5%（人少）、老用户上 −1%（人多），整体一平均变成 −0.5%——其实它对新用户是好的，分群才看得出来。
- 口径模板："怎么在线验证模型改动"→ 定 success + guardrail 指标 → 选随机化单元 → 按方差/MDE 定样本量 → 跑完不许偷看 → 查分群。排序题点名 interleaving，自适应分配点名 bandit。

## L6. Embeddings
- 稠密低维向量表语义相似，替代稀疏 one-hot。Word2Vec（CBOW 上下文猜词 / skip-gram 词猜上下文）、GloVe、fastText（subword，抗拼错/生僻）。维度 **100–300**，BERT 768。相似度：cosine / dot product。
- **Two-tower**：user 和 item 各一塔，dot product + **ANN**（FAISS、ScaNN、HNSW）做候选召回 → **retrieve-then-rank** 两阶段是标准答案。
  - 直觉 **two-tower**：user 塔和 item 塔各学一个向量，线上用内积算相似度。为什么需要：user 和 item 的向量可以**离线各自算好**，线上只做内积 + 最近邻检索——几亿 item 也扛得住。小例子：用户向量 [0.2,0.8,…]、视频向量 [0.3,0.7,…]，内积高就推；item 向量离线算好存 FAISS，请求来了只算 user 塔。
  - 直觉 **ANN（近似最近邻）**：在几亿向量里找最像的，精确搜索太慢，ANN 用"近似"换速度（HNSW 建多层小世界图、FAISS 做聚类/量化）。为什么需要：召回要在几十毫秒内从全库捞几百个候选，暴力精确算根本来不及。小例子：1 亿个视频向量，HNSW 从顶层"高速公路"几跳就定位到相似簇，几毫秒返回 top-500，召回率 95%+。
- 训练技巧：**negative sampling**、**triplet loss** L = max(d(a,p) − d(a,n) + margin, 0)。冷启动靠 content-based embedding。存 vector DB + ANN 索引。
  - 直觉 **negative sampling**：正样本（点了）天然有，负样本（没点）无穷多——训练时随机抽几个当负例。为什么需要：全量负样本算不动，抽样让训练可行；但抽样方式决定模型学到什么。小例子：用户点了视频 A，随机抽 B、C、D 当"不喜欢"，模型学"离 A 近、离 B/C/D 远"。
  - 直觉 **triplet loss**：三元组（anchor, 正例, 负例），目标是 d(a,p) + margin < d(a,n)——正例至少比负例近一个 margin。为什么需要：直接规定"近/远"的几何关系，学出的 embedding 空间可解释。小例子：anchor 是用户，p 是他看完的视频，n 是随机视频；loss 逼着"用户–看完"距离比"用户–随机"小 0.2 以上。
- Trade-off：维度大表达强但吃内存拖 ANN；预训练省数据但可能 domain mismatch。

## L7. 迁移学习（"标注数据很少怎么办"的标准答案）
- 两模式：**feature extraction**（冻 backbone 只训分类头——便宜、稳）vs **fine-tuning**（解冻部分/全部层继续训，学习率通常小 \~10 倍）。
  - 直觉 **feature extraction vs fine-tuning**：feature extraction 把预训练模型当"特征提取器"，只训最后的分类头——快、稳、不易过拟合；fine-tuning 把后面几层也放开一起训，学习率调小（约小 10 倍）防止把预训练知识冲掉。怎么选：数据少/任务像 → extraction；数据多/任务差异大 → fine-tuning。小例子：1000 张猫狗图，冻住 ResNet 只训最后一层，1 小时搞定；10 万张医学影像，就把后几层解冻、小学习率继续训。
- 实践：冻 early layers（学的是通用特征），fine-tune 后面层；**discriminative LR**（逐层衰减）；gradual unfreezing。
  - 直觉 **discriminative LR / gradual unfreezing**：越靠前的层越通用、越不该大动，所以学习率逐层衰减（前面小、后面大）；gradual unfreezing 更保守——先只训顶层，稳定了再一层层往下解冻。为什么需要：预训练知识是资产，一把梭全量大学习率等于推倒重来。
- 风险点名：**catastrophic forgetting**（洗掉预训练知识）、**negative transfer**（domain 差太远越迁越差）。相关：domain adaptation、multi-task、few/zero-shot。
  - 直觉 **catastrophic forgetting**：新任务大学习率一训，模型把原来学的东西忘了。为什么：梯度更新不分"旧知识还是新知识"，全冲着新 loss 去。小例子：BERT 在医疗文本上大学习率训 10 个 epoch，通用英语理解能力反而掉了——小学习率 + 只调顶层就是防这个。
  - 直觉 **negative transfer**：源 domain 和目标差太远，迁移反而帮倒忙。为什么：预训练的"先验"错了，还不如从零学。小例子：用自然图像预训练的模型去搞显微镜细胞图，学到的纹理先验全是错的，不如直接训。

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
