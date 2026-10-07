# ML 面试 II：Search Ranking + Feed — 提炼笔记（Ch3–Ch4，2026-10-06）

> 自己的话总结。MLE 视角：排序问题的标准答题骨架（funnel + 指标 + 训练数据 + 实验）。

## Ch3. Search Ranking

**问题定义**：把"返回最相关的文档"转成 **learning-to-rank** 问题，不止关键词匹配，要语义。先澄清 relevance 的定义再设计（precision vs semantic recall）。

**指标**：人工 graded label 0–4（不相关→完美）。**NDCG** 是 headline metric：位置 log 折扣奖励排得靠前的高相关文档，再用理想排序归一化，跨 query 可比。业务指标：CTR、**zero-click sessions**（搜了没点=bad sign）。口径：NDCG 打败 precision 的原因是 **position bias**；north-star + guardrail 成对出现。

**架构**：经典两阶段 funnel——**document selection**（\~10 万候选，优化 recall，保证 top 5–10 相关文档不丢）→ **ranking**（优化 precision）。分开是因为两阶段目标不同；trade-off：单阶段简单 vs 多阶段（延迟换质量）。

**Document Selection**：单 query \~10 万候选、低延迟；keyword + 语义混合召回；breadth（候选越多 recall 越高）vs ranking 阶段的延迟/算力。用 **recall@K** 衡量召回层。

**特征工程**：两大家族——**interaction**（历史点击/engagement 信号）vs **content**（query-doc 文本匹配）；embeddings 把文本压成稠密向量补语义。口径：小心 **feature leakage**（serving 时用未来信号）；特征必须在微秒级可算。

**训练数据**：人工标注 0–4（贵但干净）vs **implicit**（点击/停留=正，曝光未点=负；噪声大、有偏）。**Pointwise**（预测绝对分，简单、可校准）vs **pairwise**（学相对顺序 A>B，贴近排序目标、对 noisy 绝对标签鲁棒）。深挖点：position bias、click 数据的 selection bias。

**Ranking**：LTR 直接优化 NDCG；pointwise vs pairwise/**listwise**。树模型+神经网络在 held-out 上调 NDCG。**优化你汇报的那个指标**，别拿 accuracy 当 proxy；重打分要卡 latency budget。

**Filtering**：排序后的安全层，**独立分类问题**（不混进 relevance ranking，安全 precision 单独调）。太激进伤 recall，太松伤用户信任。

## Ch4. Feed（Twitter timeline）

**问题定义**：reverse-chron 把高 engagement 的 tweet 埋了 → 按 **P(engagement)** 排序。规模：5 亿 DAU × 100 关注 × 日 10 拉 ≈ **50 亿次排序/天**。

**指标**：**加权 engagement** Σ(w_i × count_i)，report 给负权重，**按活跃用户归一化**（不同用户量的实验才可比）。**Counter-metric**：人均负向行为——防 engagement bait。权重是业务选择。

**架构**：tweet selection（取上次登录以来的候选池）→ 训练数据生成（engagement 行为变 label）→ ranker。**Per-action 多模型**（like/comment/share/hide/report 各一，权重合并）vs 单模型：多模型赢在**业务可控 + 可调试**。

**Tweet Selection**（5 招，edge case 最爱考）：上次登录后的新 tweet；老 tweet 新获大 engagement 则**重捞**；9 点排 450 名的未见 tweet 下次混回来；久未登录用户**截断**（2 周没来只给 500 条）；**out-of-network** 按兴趣/趋势推——解新用户冷启动 + 促发现。

**特征工程**（4 个 actor：用户、tweet、作者、context）：
- **user-author affinity**：如作者近 12 条赞了 6 条 = 0.5，**按用户总活跃归一化**（消掉习惯差异）；
- 相似度：共同关注、TF-IDF、embedding dot product、社交图 embedding；
- **作者权威**：verified、PageRank 式 social rank、**follower/following 比**（spam vs influencer 信号）；
- engagement 率多时间窗；**时间衰减 1/(t+1)**（一月前 500 万 engagement 打不过 1 小时前的热帖）；
- context：星期/时段/地点/季节/热搜/节假日。Sparse：unigram/bigram、user_id、tweet_id（走 embedding）。

**训练数据**：engaged=正，曝光未点=负（**按 action 分 label**：评了但没赞的 tweet 在 Like 模型里是负样本）。5% engagement → 日 1 亿曝光 = 500 万正/9500 万负；cap 1000 万采样 → 负采样到 500 万:500 万。**关键：downsample 破坏 calibration**（模型输出 50%，真实 5%）——排序无所谓（只看相对顺序），广告不行（要出价）。**必须按时间切 train/test**：随机切忽略 weekday/weekend 分布变化；拿前面几周训、后面几周验，模拟"预测未来"。量级：\~7000 万行/周。

**Ranking**：分类问题 P(like)/P(comment)/P(retweet)，AUC。模型 ladder：**LR**（快、可解释，要手造交叉特征）→ **MART/树**（非线性，几百万样本 sweet spot）→ **深网**（1 亿用户规模才划算，两阶段 serving：简单模型先筛、复杂模型精排）。**Multi-task**：共享层 + per-action heads，total_loss = 各 loss 相加（联合正则、训得快）。**Stacking**：树/NN 输出（leaf ID 当 bool 特征、最后一层 hidden 当特征）喂 LR——**支持 online learning**（每次用户行为都更新）、吃 sparse ID 特征做 memorization、分布式实时刷新。

**Diversity**：纯 engagement 排序会扎堆（连 5 条同一作者、连 4 个视频）——单条都对，整体无聊。修法：**author diversity**（重复作者 -0.1）、**content diversity**（同类内容降 3 位）。"模型完美但用户抱怨"是经典 follow-up，考你跳出目标函数思考。

**Online Experimentation**（上线管线）：训 \~15 个候选（换特征/算法/超参）→ 离线验 held-out → **A/B 1% 用户**（5 亿的 1% = 500 万，250 万/250 万），**先拿 withheld 的近期数据重训**再测 → 决策：例 18 万 vs 15 万 = **20% 提升**，要显著性（p-value），权衡复杂度——**不显著的复杂度不许上线**。

## 自测 4 问（带答案要点）
1. Pointwise vs pairwise 怎么选？
   → pointwise 简单、可校准；pairwise 贴近排序目标、对 noisy 绝对标签鲁棒；排序问题优先 pairwise/listwise，直接优化 NDCG。
2. 负采样破坏 calibration，为什么排序没事、广告有事？
   → 排序只关心相对顺序；广告要出价，需要真实概率。
3. 为什么 train/test 必须按时间切？
   → 随机切忽略时间维度和 weekday/weekend 分布变化；按时间切模拟"预测未来"。
4. 模型指标完美但用户抱怨无聊，怎么修？
   → author diversity（重复作者 -0.1）、content diversity（同类内容降 3 位）。
