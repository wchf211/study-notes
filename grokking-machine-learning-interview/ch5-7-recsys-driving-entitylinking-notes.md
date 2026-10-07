# ML 面试 III：推荐 + 自动驾驶分割 + Entity Linking — 提炼笔记（Ch5–Ch7，2026-10-06）

> MLE 视角：推荐是最高频的 ML 系统设计题，先啃透它。

## Ch5. 推荐系统（Netflix 视角）

**问题定义**：转成 ranking 问题——最大化 P(engagement|user, item, context)，按分排序。规模锚点：1.635 亿订阅（2019）、5300 万国际日活、**80% 播放来自推荐**。
**Implicit vs explicit**：implicit（看了/没看）量大、反映真实行为，赢；explicit 有 **MNAR bias**（只有热情用户才打分、4–5 星为主、没有负信号）且稀疏。另类 formulation：rating prediction（预测打几星）。

**指标**：offline（迭代）vs online（真实行为，A/B 是 ground truth）。RMSE（explicit 场景）、P@k/R@k/F1@k、**NDCG**（首选，graded + 位置折扣）；online：play rate、watch time、sessions；guardrail：retention、churn。口径：ranking 指标打败 accuracy 因为**位置重要**；会手写 NDCG。

**架构**：两阶段 funnel——**candidate generation**（几百候选，管 recall）→ **ranking**（top-N，管 precision）；外加 UI、filtering、logging/impression、feature store、训练/serving、监控、实验。分两阶段是因为重模型不可能全量打分。

**特征**：user（历史、画像、embedding）、item（metadata、embedding）、context（时间/设备/地点）、交叉特征。高基数类别（user/movie ID）→ **学出来的稠密 embedding**，不用 one-hot；sparse 存储、hashing trick、归一化。冷启动要专门特征。

**Candidate Generation**：popularity 基线、content-based（TF-IDF）、协同过滤（user/item-based）、矩阵分解、**two-tower**（user/item 双塔，dot product + ANN）。CF 冷启动差；**two-tower + ANN 是生产答案**（双塔解耦、sub-linear 检索）。

**训练数据**：正样本 engagement，负样本采样/skip；**曝光未点比随机采样更好的负信号**（但带 position bias）；**按时间切分**防未来泄漏；watch 阈值定 label（噪声 vs 量）。深挖：负采样策略、**IPW** 纠 position bias。

**Ranking**：深模型（wide&deep/DNN），pointwise/pairwise/listwise；输出**校准过的** engagement 概率（分数要进业务逻辑，calibration 重要）；深度 vs 延迟；OOV 映射到 zero embedding。

## Ch6. 自动驾驶：图像分割

**问题定义**：逐像素分类（road/lanes/车辆/行人/标志），实时约束；**行人类错误是主要 concern**（安全）。

**指标**：pixel accuracy（**误导**，road 像素主导）、**IoU/mIoU**（标准）、Dice；**per-class IoU** 暴露稀有类翻车；系统指标：车端推理延迟/FPS。

**架构**：车队数据回传 → 标注管线 → 训练基建 → model registry → **车端（edge）部署** → OTA → 监控；**fleet-learning 闭环**（shadow mode / 接管事件挖成训练数据）；仿真。

**训练数据**：像素 mask 标注贵 → **active learning** 挖 hard case、仿真/合成数据、GAN 场景、augmentation、弱监督；多样性（天气/光照/地域）；synthetic-to-real **domain gap**；类别不平衡用采样 + loss 加权。

**建模**：**FCN**（全卷积，任意输入尺寸，upsample + skip connection 恢复边缘）、**U-Net**（对称 encoder-decoder + skip）、**Mask R-CNN**（Faster R-CNN 的 RPN 提 proposal + 三头：分类/框/mask）。Semantic vs instance；精度 vs 实时。**迁移学习分层**：数据少只训顶层 → 中等解冻几层（边训边看 IoU）→ 数据多才全量训。

## Ch7. Entity Linking

**问题定义**：文本 mention → 知识库标准实体。两阶段：**NER**（找 mention）+ **NED**（消歧，如 "Michael Jordan" → 教授还是球员）。核心难点：歧义 + 别名。口径：先把 NER 和 NED 分清楚，面试官会查你混没混。

**指标**：分阶段 + 端到端——NER 用 P/R/F1，NED 用 linking accuracy，端到端 F1。**Pipeline 误差传播**（NER 漏了 NED 直接没戏）；**NIL**（mention 在 KB 里没对应实体）要显式处理。

**架构**：ingestion → NER → **candidate generation**（KB 索引查，调 recall：真实体绝不能丢）→ **disambiguation**（调 precision：选对的）→ KB。Funnel 口诀：**recall then precision**；pipeline vs joint 建模是架构争论点。

**训练数据**：**Wikipedia anchor text** 是免费午餐——(mention → KB 实体) 对白给；anchor 频率算 **prior**（"Harrison Ford" 98% 指美国演员）。Distant supervision 便宜但噪声大；prior 是强特征但会**固化 popularity bias**；NIL 负样本要刻意构造。

**建模**：
- **ELMo vs BERT**（经典考题）：ELMo = char-CNN + 双向 LSTM 前后向**独立**拼起来 → **shallow** 双向；BERT = transformer 自注意力 + **masked LM** 联合建模 → **deep** 双向。BERT 规格：base 12 层/1.1 亿参，large 24 层/3.4 亿，**DistilBERT** 保 97% 质量快 60%，cased 版对 NER 有帮助。
- NER：embedding 当特征 vs fine-tuning（全量还是 top-k，看数据量和预算）。
- 消歧：candidate gen（名字词 + anchor text + 别名/同义词 + embedding k-NN 扩索引）→ DNN linking 分类器（mention、实体类型、整句、候选实体、prior）→ P(候选是真身)。

## 自测 4 问（带答案要点）
1. 推荐里 implicit vs explicit feedback 怎么选？
   → implicit 量大、反映真实行为；explicit 有 MNAR bias（只有效应粉打分、4–5 星为主、无负信号）且稀疏。
2. 候选生成为什么 two-tower + ANN 是生产答案？
   → 双塔解耦、sub-linear 检索；对比 CF（冷启动差）、content-based（依赖好 metadata）。
3. 分割里 pixel accuracy 为什么不能单独用？
   → road 像素主导，均值掩盖行人漏检；用 mIoU + per-class IoU + 安全加权。
4. ELMo vs BERT 双向性的区别？
   → ELMo 前后向独立 LSTM 拼接（shallow）；BERT masked LM 联合建模上下文（deep）。
