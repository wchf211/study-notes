# 三种范式 + 传统 vs 深度（Ch3）

> 本章是全课技术心脏：监督/无监督/强化的算法全家桶 + 评估指标 + 神经网络怎么训。术语第一次出现都给了白话。

## 1. Supervised：学映射，赌泛化

有标签数据学输入→输出映射。整场游戏叫 **generalization**（没见过的数据上表现好），夹在两种死法之间：**underfitting**（太简单，训练新数据都烂）vs **overfitting**（背下训练噪声，训练好新数据烂）。

标准 workflow：收数据→预处理→特征→切分（train/val/test）→训→val 上调 **hyperparameters**（训练前你定的设置，如树深/学习率；区别于模型学到的 parameters）→test 评估→部署。

**Regression vs classification**：
- Regression：连续数（房价）。指标：**MSE**（均方误差——大错罚更狠）、**MAE**（平均绝对误差——对离群鲁棒）、**R²**（解释了目标方差的几分之几，1=完美）。
- Classification：离散类（垃圾/非垃圾）。分 binary / multiclass / multilabel（一张照片同时"海滩"+"日落")。指标：
  - **Accuracy**=答对/总数（类别不平衡时骗人）；
  - **Precision**= flagged 里真对的（TP/(TP+FP)）——**误报贵时优化它**（正常邮件进垃圾箱）；
  - **Recall**= 真对的里抓到的（TP/(TP+FN)）——**漏报贵时优化它**（体检漏肿瘤）；
  - **F1**= precision/recall 调和平均，两边都要时看它；
  - **ROC-AUC**= 全阈值上的类别分离能力，0.5=瞎猜，1=完美。

**算法记分卡**：
- Linear regression：y=b0+b1x1+… 最小二乘拟合；简单可解释；假设线性、怕离群。
- Logistic regression（名字骗人，是分类器）：sigmoid 把线性分数压成 0–1 概率，0.5 切；简单高效给概率；线性假设。
- Decision trees：好读、数字类别通吃、预处理少；易过拟合、树大没法看。
- **SVM**：高维强，**kernel trick** 处理非线性；大数据慢、kernel 难选。
- **KNN**：巨简单、新数据不用重训；预测慢（扫全集）、k 难选。
- **Naive Bayes**：极快、高维文本好；假设特征独立（"naive"但常 work）。

实操：**先上简单模型当 baseline**；指标按"错的代价"选不按习惯；类别偏先看 precision/recall。

## 2. Unsupervised：没答案，自己找结构

**Clustering（聚类）**：
- **K-Means**：定 k→随机撒 k 个 **centroids**（簇心）→"每点归最近心→心移到簇均值"循环到稳。简单可 scale；k 得事先定、结果看初始化、假设簇近似球形、离群点拽心。
- **DBSCAN**：按密度长簇（邻域半径+最小点数），稀疏的标 noise。簇形状任意、不用 k、自带离群检测；密度差异大的簇搞不定。
- **Hierarchical**：自底向上合并（或自顶向下分），**dendrogram**（树图）事后选粒度；不用 k 但大数据贵。

**Association mining**（"买 X 的人买 Y"）：**support**（这组合多常见）、**confidence**（有 X 时 Y 多可靠）、**lift**（比 baseline 强多少——排除"俩都热门"的假关联）。**Apriori** 高效挖（剪掉太稀有的）。

**Dimension reduction**：**PCA**（投影到方差最大方向）、**t-SNE**（非线性、保局部邻域、2D 可视化神器但慢+随机+不适合进管线）、**autoencoders**（神经网络经窄 bottleneck 重构自己——bottleneck 自动学会压缩表示）。

没标签评估难：**silhouette score**（点离自己簇 vs 最近别簇，−1\~1 越高越干净）、**elbow method**（簇内散布 vs k 画曲线找拐点）、领域专家肉眼。

实操：先聚类探索再标注；k 未知/有噪声→DBSCAN 优先 K-Means；**距离类算法先 scale**；验证分数当 hint 不当 verdict。

## 3. Reinforcement：做中学

Agent 在环境里行动→拿 reward→多轮收敛到长期 reward 最大的 **policy**。

反馈环：observe **state**（局面描述，如棋盘）→ 选 **action** → 环境转到新 state → 拿 reward → 更新 policy/**value function**（"处在这个 state 有多好"的期望未来 reward）→ 重复。

**Exploration vs exploitation**（核心 tension）：探索试新的 vs 利用已知的薅。**Epsilon-greedy**：ε 概率随机乱试，否则贪最优——教科书折中。

**Model-based vs model-free**：前者学/拿环境模型做规划（省样本但模型错则全错）；后者直接从经验学（简单无偏但巨能吃数据）。

算法巡礼：
- **Q-learning**：学每个 state-action 的 Q 值（期望未来 reward），按 Q(s,a)←Q+α[r+γ·maxQ(s′,a′)−Q] 修正——白话：往"刚拿到的 reward + 下个 state 最好的未来"方向 nudged；α 学习率、γ **discount factor**（0=只看眼前，≈1=看长远）。
- **SARSA**：on-policy 版（按实际走的 action 更新，不按最优的）。
- **DQN**：Q 表换神经网络，吃得下游戏像素级大状态。
- **Policy gradient**（REINFORCE）：直接优化 policy，连续动作（方向盘角度）天然合适。
- **Actor–critic**：actor 提动作、critic  judge。
- **PPO**：clip 每次 policy 走多远，训练稳。

**Reward 设计是实战最难**：sparse reward（只终局 win/lose）指导少；dense reward 好学但被 game；**credit assignment**（现在的 reward 该算哪步的功）是长程任务难的根。**Sim-to-real gap**：仿真里完美、真机上拉胯。

实操：RL 适合**序列决策+清晰反馈**，不适合单次预测；预算海量算力/样本；reward 写错→agent 聪明地干错事。

## 4. 传统 ML vs Deep Learning

DL 不是对立范式——是**神经网络自己从原始数据学特征**的 ML，省掉人工特征工程那步。

| | 传统 | 深度 |
|---|---|---|
| 算法 | 线性/树/SVM | 多层神经网络 |
| 特征 | 专家手工 | 自动（浅层学边缘纹理，深层组合成物体概念） |
| 数据 | 结构化表格、小数据 | 非结构化（图/音/文本）、要大数据 |
| 硬件/时间 | CPU、分钟级 | GPU、小时/天级 |
| 可解释 | 高（看系数/树分裂） | 低（黑盒倾向） |

**神经网络白话**：神经元分层排；每 neuron 算加权和+bias，过 **activation function**（非线性——没它多层塌成一层线性）。常用：sigmoid（压 0–1，二分类输出）、tanh（压 −1\~1）、**ReLU**（max(0,x)，隐藏层默认，便宜抗梯度消失）、**softmax**（转概率分布，多分类末层标配）。

**训练白话**：forward（数据推过去得预测）→ 算 **loss**（错得多离谱的一个数）→ **backpropagation**（链式法则逐层分锅）→ **gradient descent**（每个权重往 loss 减小的方向挪一小步；**learning rate** 步长——太大发散太小爬）→ **epochs**（全数据刷多遍）重复。

**架构动物园**：**CNN**（卷积扫局部空间 pattern+pooling 缩减——图像 pipeline 之王）；**RNN**（带环跨步记记忆——时序/文本；**LSTM/GRU** 门控版治长程失忆）；**Transformer**（**self-attention**：每个位置掂量其他位置的分量——可并行，现代 LLM 地基；GPT=生成式 decoder，BERT=双向 encoder 理解强）。

选型：中小结构化数据+要可解释+算力紧→传统；大非结构化+准头优先+有 GPU→深度。Hybrid 合法：深度网当特征提取器喂经典分类器。
**别在 1 万行 spreadsheet 上掏深度学习**——梯度提升树更快更便宜准头还高。真上深度：**transfer learning**（大网预训练+自己任务微调）省数据省算力。

## 自测 4 问（带答案要点）
1. Underfitting vs overfitting？generalization 是什么？
   → 太简单啥都烂 vs 背下噪声新数据烂；没见过的数据上好。
2. Precision vs recall 各保什么？F1/ROC-AUC？
   → 误报贵保 precision，漏报贵保 recall；F1 调和平均，AUC 全阈值分离能力。
3. Epsilon-greedy 解决什么 tension？discount factor γ 干嘛？
   → 探索/利用；γ=0 只看眼前，≈1 看长远。
4. Backprop+梯度下降白话？ReLU 为什么是隐藏层默认？
   → 分锅+挪权重减 loss；便宜、抗梯度消失。
