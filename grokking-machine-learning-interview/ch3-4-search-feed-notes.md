# ML 面试 II：Search Ranking + Feed — 提炼笔记（Ch3–Ch4，2026-10-06）

> MLE 视角：排序问题的标准答题骨架（funnel + 指标 + 训练数据 + 实验）。

## Ch3. Search Ranking

**问题定义**：把"返回最相关的文档"转成 **learning-to-rank** 问题，不止关键词匹配，要语义。先澄清 relevance 的定义再设计（precision vs semantic recall）。

**指标**：人工 graded label 0–4（不相关→完美）。**NDCG** 是 headline metric：位置 log 折扣奖励排得靠前的高相关文档，再用理想排序归一化，跨 query 可比。业务指标：CTR、**zero-click sessions**（搜了没点=bad sign）。口径：NDCG 打败 precision 的原因是 **position bias**；north-star + guardrail 成对出现。
  - 直觉 **NDCG 的两步**：第一步 DCG——每个位置得分 = 相关度 / log(位置+1)，位置越靠后折扣越狠；第二步除以"理想排序"的 DCG 归一化，这样不同 query 之间可比（0–1 分）。小例子：query A 理想 DCG=10、你的排序做到 8 → NDCG=0.8；query B 理想 DCG=4、你做到 3.2 → 也是 0.8。
  - 直觉 **zero-click sessions**：用户搜了但一个结果都没点。为什么是 bad sign：大概率结果太烂、用户直接放弃（少数情况是答案已在摘要里，但先当坏信号查）。小例子：搜"iPhone 17 电池容量"，前 10 条全是广告，用户关了页面——这一 session 就是 zero-click。
  - 直觉 **position bias**：用户天然更爱点排前面的，不管它是不是真更相关。为什么致命：拿点击当 label 训模型，模型学到的其实是"排前面=好"，自我强化。小例子：把第 10 名的文档挪到第 1 名，点击率立刻翻倍——不是它变好了，是位置红利。

**架构**：经典两阶段 funnel——**document selection**（\~10 万候选，优化 recall，保证 top 5–10 相关文档不丢）→ **ranking**（优化 precision）。分开是因为两阶段目标不同；trade-off：单阶段简单 vs 多阶段（延迟换质量）。

**Document Selection**：单 query \~10 万候选、低延迟；keyword + 语义混合召回；breadth（候选越多 recall 越高）vs ranking 阶段的延迟/算力。用 **recall@K** 衡量召回层。
  - 直觉 **recall@K**：所有相关文档里，有多少被召回进了前 K。为什么召回层看这个不看 precision：召回的任务是"别漏"，漏掉的文档精排再强也救不回来。小例子：10 篇相关文档，召回 10 万候选里捞回 8 篇 → recall@100k=0.8；剩下 2 篇永远没机会被排序。

**特征工程**：两大家族——**interaction**（历史点击/engagement 信号）vs **content**（query-doc 文本匹配）；embeddings 把文本压成稠密向量补语义。口径：小心 **feature leakage**（serving 时用未来信号）；特征必须在微秒级可算。
  - 直觉 **feature leakage**：训练时不小心用了"当时还不可能知道"的信息。为什么致命：离线指标虚高、上线立刻崩。小例子：用"该文档 30 天总点击"当特征预测"用户今天会不会点"——训练时 30 天数据是全的，serving 时今天才刚开始，特征根本算不出来。

**训练数据**：人工标注 0–4（贵但干净）vs **implicit**（点击/停留=正，曝光未点=负；噪声大、有偏）。**Pointwise**（预测绝对分，简单、可校准）vs **pairwise**（学相对顺序 A>B，贴近排序目标、对 noisy 绝对标签鲁棒）。深挖点：position bias、click 数据的 selection bias。
  - 直觉 **pointwise vs pairwise vs listwise**：pointwise 把每篇文档独立打分（"这篇 3 分"），简单、分数可校准，但它不知道"排序"是相对游戏；pairwise 学"A 应该排在 B 前面"，直接对准排序目标，对 noisy 标签更鲁棒；listwise 一次看整个榜单优化。怎么选：要校准概率（如广告出价）用 pointwise；纯排序（搜索/推荐）优先 pairwise/listwise。小例子：标注员给 A 打 3 分、B 打 2 分可能是手抖，但"A>B"这个相对判断通常是对的——pairwise 只信相对关系。

**Ranking**：LTR 直接优化 NDCG；pointwise vs pairwise/**listwise**。树模型+神经网络在 held-out 上调 NDCG。**优化你汇报的那个指标**，别拿 accuracy 当 proxy；重打分要卡 latency budget。

**Filtering**：排序后的安全层，**独立分类问题**（不混进 relevance ranking，安全 precision 单独调）。太激进伤 recall，太松伤用户信任。

## Ch4. Feed（Twitter timeline）

**问题定义**：reverse-chron 把高 engagement 的 tweet 埋了 → 按 **P(engagement)** 排序。规模：5 亿 DAU × 100 关注 × 日 10 拉 ≈ **50 亿次排序/天**。

**指标**：**加权 engagement** Σ(w_i × count_i)，report 给负权重，**按活跃用户归一化**（不同用户量的实验才可比）。**Counter-metric**：人均负向行为——防 engagement bait。权重是业务选择。
  - 直觉 **加权 engagement**：不同行为价值不同，like=1、comment=3、share=5、report=−10，加权求和当总分。为什么需要：只看"互动次数"会逼出标题党——report 给负权重就是告诉模型"被举报的互动是扣分项"。小例子：帖子 A 有 100 个 like（100 分），帖子 B 有 10 个 share 但 2 个 report（50−20=30 分）——模型学会别推招骂的内容。
  - 直觉 **counter-metric**：主指标涨的同时必须盯着的"刹车指标"。为什么需要：优化 engagement 最容易的歪路是推争议/愤怒内容，counter-metric（人均 report/hide）就是防这条歪路的。小例子：新模型 engagement +8%，但人均 hide 也 +15%——主指标再好看也不能上。

**架构**：tweet selection（取上次登录以来的候选池）→ 训练数据生成（engagement 行为变 label）→ ranker。**Per-action 多模型**（like/comment/share/hide/report 各一，权重合并）vs 单模型：多模型赢在**业务可控 + 可调试**。

**Tweet Selection**（5 招，edge case 最爱考）：上次登录后的新 tweet；老 tweet 新获大 engagement 则**重捞**；9 点排 450 名的未见 tweet 下次混回来；久未登录用户**截断**（2 周没来只给 500 条）；**out-of-network** 按兴趣/趋势推——解新用户冷启动 + 促发现。

**特征工程**（4 个 actor：用户、tweet、作者、context）：
- **user-author affinity**：如作者近 12 条赞了 6 条 = 0.5，**按用户总活跃归一化**（消掉习惯差异）；
  - 直觉 **user-author affinity**：你有多喜欢这个作者 = 你赞过他几条 / 你总共赞过几条。为什么要归一化：有人一天赞 100 条、有人一周赞 1 条，不归一化的话"重度用户和谁都亲"。小例子：你近 12 条赞里 6 条给了 @猫片日报 → affinity 0.5，他的新帖子给你加权。
- 相似度：共同关注、TF-IDF、embedding dot product、社交图 embedding；
- **作者权威**：verified、PageRank 式 social rank、**follower/following 比**（spam vs influencer 信号）；
- engagement 率多时间窗；**时间衰减 1/(t+1)**（一月前 500 万 engagement 打不过 1 小时前的热帖）；
  - 直觉 **时间衰减 1/(t+1)**：engagement 要按新鲜度打折，t 是小时数。为什么需要：feed 是"现在"的东西，一个月前的爆款今天推出来就是考古。小例子：帖子 A 一月前 500 万赞 → 500万/720 ≈ 7000 分；帖子 B 一小时前 10 万赞 → 10万/2 = 5 万分——B 赢。
- context：星期/时段/地点/季节/热搜/节假日。Sparse：unigram/bigram、user_id、tweet_id（走 embedding）。

**训练数据**：engaged=正，曝光未点=负（**按 action 分 label**：评了但没赞的 tweet 在 Like 模型里是负样本）。5% engagement → 日 1 亿曝光 = 500 万正/9500 万负；cap 1000 万采样 → 负采样到 500 万:500 万。**关键：downsample 破坏 calibration**（模型输出 50%，真实 5%）——排序无所谓（只看相对顺序），广告不行（要出价）。
  - 直觉 **downsample 为什么破坏 calibration**：真实世界 5% 正样本，你下采样到 1:1，模型学到的世界就是"一半一半"——它输出的 0.5 其实对应真实的 0.05。排序为啥没事：0.5 和 0.05 都是"比 0.3 大"，相对顺序没变；广告为啥有事：出价公式 pCTR×bid 里 pCTR 必须是真概率，0.5 当 0.05 用会贵 10 倍买下垃圾流量。**必须按时间切 train/test**：随机切忽略 weekday/weekend 分布变化；拿前面几周训、后面几周验，模拟"预测未来"。量级：\~7000 万行/周。

**Ranking**：分类问题 P(like)/P(comment)/P(retweet)，AUC。模型 ladder：**LR**（快、可解释，要手造交叉特征）→ **MART/树**（非线性，几百万样本 sweet spot）→ **深网**（1 亿用户规模才划算，两阶段 serving：简单模型先筛、复杂模型精排）。**Multi-task**：共享层 + per-action heads，total_loss = 各 loss 相加（联合正则、训得快）。**Stacking**：树/NN 输出（leaf ID 当 bool 特征、最后一层 hidden 当特征）喂 LR——**支持 online learning**（每次用户行为都更新）、吃 sparse ID 特征做 memorization、分布式实时刷新。
  - 直觉 **模型 ladder**：按数据量和复杂度爬梯子。LR 是"带权重的加法"，快、可解释，但特征交叉得手工造（如"年轻×夜猫子"）；MART/GBDT 自动学非线性，几百万样本时性价比之王；深网要上亿样本才喂得饱，且 serving 贵。小例子：10 万样本用 LR 足够；500 万样本上 GBDT；1 亿用户行为才值得掏深网——梯子别跳级。
  - 直觉 **multi-task**：like/comment/share 各一个输出头，但前面几层网络共享。为什么需要：三个任务都需要"理解这条 tweet"，共享层让数据互相帮忙（comment 样本少，可以蹭 like 的表示），还省了 3 个模型的 serving 成本。小例子：share 的正样本太少，单独训过拟合；和 like 一起训，共享层学到的"什么是好内容"直接复用。
  - 直觉 **stacking**：两层模型——底层（树/NN）先学出高级特征，顶层一个简单的 LR 做最终组合。为什么这么叠：LR 顶层可以 online learning（来一条行为更新一次，毫秒级跟上热点），而底层复杂模型天级更新；leaf ID 当特征等于把树的"划分逻辑"翻译成 LR 能吃的 0/1 信号。小例子：世界杯期间"梅西"突然爆火，顶层 LR 几分钟内就把相关权重调上去，不用等底层大模型重训。

**Diversity**：纯 engagement 排序会扎堆（连 5 条同一作者、连 4 个视频）——单条都对，整体无聊。修法：**author diversity**（重复作者 -0.1）、**content diversity**（同类内容降 3 位）。
  - 直觉 **diversity 修正**：模型只认 engagement 的话，会把"最对你胃口的 10 条"全推成同一个作者——单条都对，整体像复读机。修法就是人工扣分：同一作者出现第二次减 0.1，连续 3 个视频降 3 位。为什么不用模型学：多样性是"全局观"，单条打分时看不见榜单，只能后处理。"模型完美但用户抱怨"是经典 follow-up，考你跳出目标函数思考。

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
