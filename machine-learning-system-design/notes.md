# Machine Learning System Design（2h 课）— 提炼笔记（2026-10-07）

> 短课，一篇全收：Ch1 方法论 + 5 个实战（视频推荐、Feed、广告点击、民宿搜索、外卖 ETA）。

## Ch1. 方法论（6 步框架）

**答题框架**：problem statement → metrics → requirements → train/evaluate → high-level design → **scale**（scale 讲不好是区分强弱的分水岭）。
**先问澄清问题**再设计（如 LinkedIn feed： chronological 还是 relevance？广告和 organic 怎么配？看哪些 action？）。
**指标**：offline（log loss、AUC；RMSE/MAPE）vs online（CTR、engagement、revenue lift）。
  - 直觉 **log loss**：对"说错且很自信"的重罚——预测 0.99 结果是 0，loss 爆炸；预测 0.51 错了，罚得轻。为什么概率场景用它：它逼模型输出"诚实的概率"，不只是排对顺序。小例子：两个模型都把正样本排前面（AUC 一样），A 说 0.9、B 说 0.6——log loss 更喜欢"更接近真实概率"的那个。
  - 直觉 **MAPE**：平均百分比误差 |预测−真实|/真实。为什么 ETA/销量场景爱用：误差 5 分钟在 10 分钟单子上是 50%、在 60 分钟单子上是 8%——MAPE 自动按单量级归一化，跨城市可比。小例子：预测 30 分钟实际 33 分钟，MAPE=10%。
**Training 需求**（数据、特征、loss、不平衡、重训节奏）vs **Inference 需求**（\~100ms、百万用户、高可用）；**accuracy–speed–cost 三角**。
第一版架构要**小、可解释、可扩展**（candidate gen + ranking 两服务）。

**特征工程三阶段**：brainstorm → 训前筛选 → 训后剪枝。训前：相关性分析（Spearman 去冗余）、L1、树 importance、PCA/LDA、autoencoder；训后：**permutation importance**、**SHAP**。
  - 直觉 **permutation importance**：把某一列特征的值随机打乱，看模型指标掉多少——掉得多说明这特征重要。为什么需要：模型训完后回答"到底靠哪几个特征在干活"，简单、模型无关。小例子：打乱"用户年龄"列，AUC 从 0.8 掉到 0.65 → 年龄是核心特征；打乱"注册星期几"，AUC 纹丝不动 → 可以删。
  - 直觉 **SHAP**：permutation importance 告诉你"全局谁重要"，SHAP 告诉你"这一次预测是谁的功劳"——基于 Shapley 值的公平分账。为什么需要：debug 单个 bad case、给业务解释"为什么给这个用户推这个"。小例子：某用户被拒贷，SHAP 显示"负债比 +0.35、逾期记录 +0.3"——客服能解释，模型也顺带被审计了。
  - 直觉 **Spearman 相关**：看两个特征是不是"同涨同跌"的单调关系（不要求直线，曲线也算）。为什么去冗余：两个特征 Spearman 0.95，留一个就行——模型同时吃两个，权重打架还浪费容量。小例子："月薪"和"年薪/12"的 Spearman 接近 1，删一个。
  - 直觉 **PCA / autoencoder**：都是"降维"——把 1000 维特征压成 50 维还尽量保留信息。PCA 是线性的（找方差最大的方向投影），autoencoder 是神经网络版（encoder 压、decoder 恢复，学非线性压缩）。为什么需要：维度灾难——维度越高越需要指数级数据，降维后模型训得动、还不易过拟合。
编码口诀：one-hot 简单但高基数爆炸 → **embedding**；geo 别用 raw 经纬度（clustering/geohash/S2）；时间用 **sin/cos 周期编码** + 分桶；缺失值 impute + **missingness indicator**；feature crossing 抓交互（防组合爆炸）。
  - 直觉 **geohash / S2**：把地球切成格子，用格子 ID 代替经纬度。为什么：raw 经纬度是两个连续小数，模型很难学出"附近"这个概念；格子化后"同格=附近"天然成立，还能按需放大缩小（geohash 长度越长格子越小）。小例子：外卖 ETA 里，用户和商家落在同一个 6 位 geohash 格 → "很近"这个信号直接可用。
  - 直觉 **sin/cos 周期编码**：把"小时"变成圆上的点 (sin(2πh/24), cos(2πh/24))。为什么：23 点和 0 点在数字上差 23，在现实中只差 1 小时——raw 数字会骗模型；搬到圆上，距离就是真实的时间距离。小例子：不用这个，模型以为"23 点"和"0 点"是两个极端；用了之后，它知道这是相邻的两小时。
**口径**：每个编码都要能讲出为什么；新来的高基数 ID 默认 embedding。

**训练管线**：data → preprocessing → training → evaluation → serving。采样（random/stratified/**time-based**）；不平衡（under/over/SMOTE）；**防泄漏**；\~70/15/15 切分；holdout/k-fold/time-split 验证。Batch/mini-batch/SGD；超参（lr、batch、正则、dropout、树深）；数据少用迁移学习。**过拟合**：正则 + early stopping。上线：A/B + canary + 监控。
**口径**：为你的切分和验证策略辩护；train/serve skew 和 stale model 怎么发现怎么修。

**Inference**：实时（新鲜、吃延迟预算）vs 批量（便宜、stale）。多模型聚合：bagging、boosting、stacking、voting、blending、**Thompson sampling**（bandit，按后验采样做 explore/exploit）。扩展：batching、并行、预测缓存、**量化/剪枝/蒸馏**、加速器。
  - 直觉 **bagging vs boosting**：bagging 是"并行民主"——同样模型在数据的不同随机子集上各训一个，投票平均，降方差（随机森林就是 bagging + 树）；boosting 是"sequential 纠错"——一个接一个训，每个专治上一个的错，降偏差（GBDT/XGBoost）。小例子：数据噪声大 → bagging 求稳；模型太弱欠拟合 → boosting 加把劲。
  - 直觉 **stacking / blending / voting**：voting 最糙（少数服从多数/取平均）；stacking 用"元模型"学怎么组合基模型的输出（比如 LR 学到"树模型在 A 场景准、NN 在 B 场景准"）；blending 是 stacking 的偷懒版（留一份 holdout 定权重，不做交叉验证）。小例子：5 个差异大的模型 + stacking，单模型 0.80，叠完 0.83。
  - 直觉 **量化 / 剪枝 / 蒸馏**：模型压缩三件套，都是"拿一点精度换 serving 成本"。量化：float32 改 int8，模型小 4 倍、快 2–4 倍；剪枝：把接近 0 的权重直接删掉（神经网络天然稀疏）；蒸馏：大模型当老师，教小模型模仿它的输出分布。小例子：BERT-base 量化 + 蒸馏到 6 层，体积剩 1/7、延迟剩 1/4，精度掉不到 1 个点——线上全靠这套。
**口径**：推荐/广告必谈 explore/exploit；batch 够不够用、尾延迟怎么削。

**指标选择**：对齐业务目标。不平衡数据上 accuracy 骗人（看 precision/recall/AUC）；**RMSE 比 MAE 更罚大错**（大错贵时用 RMSE）。Offline 指导迭代，online + A/B 决定上线；**proxy 优化过头会被 gaming**。

## Ch2. 视频推荐（YouTube 式）

**定义问题**：目标是 watch time/engagement 还是 clicks？库存几百万视频（heavy tail），用户几亿，冷启动多。指标：offline AUC/log loss + rank-aware；online watch time、session 时长、CTR、retention + **diversity/freshness guardrail**（防 filter bubble）。Watch time 会养 clickbait。

**两阶段 funnel**：便宜的 candidate gen（协同过滤、content-based、**two-tower + ANN**）→ 贵的精排（几百候选）。Sampled softmax/负采样；**position bias、feedback loop**。
  - 直觉 **two-tower + ANN**：user 塔和 item 塔解耦，item 向量离线算好灌进 ANN 索引，线上只算 user 向量做一次检索。为什么是生产标准：几亿视频全量打分不可能，两阶段把"全库粗筛"和"几百个精排"分开。小例子：10 亿视频 → ANN 几毫秒捞 500 个 → 精排模型只打这 500 个的分。
  - 直觉 **sampled softmax**：输出是几百万个视频，softmax 全算要命——每次只算正样本 + 抽样的几千个负样本来近似分母。为什么需要：分类头太大，全量 softmax 会让训练从几小时变成几天。小例子：100 万视频，正样本 1 个 + 采样 5000 个负样本，loss 只在这 5001 个上算，梯度方向基本是对的。

**系统设计**：请求链 candidate gen → ranking → blending/业务规则 → 返回；feature store 喂离线+在线。估算：QPS、embedding 存储、带宽。**ANN 索引分片**、ranker 复制挂 LB、热门结果缓存、批量特征和实时特征分离。Follow-up：冷启动、freshness vs relevance、**fallback**（索引 stale、ranker 挂了怎么办）。

## Ch3. Feed 排序（LinkedIn/FB 式）

**定义问题**：chronological 还是 relevance？organic 和 sponsored 怎么配？哪些 action 算数。几亿用户、几百上千亿 posts。指标：offline AUC/log loss + rank-aware；online session 时长、DAU、meaningful interactions、广告收入 + guardrail（别靠 outrage bait 刷 engagement）。

**模型**：GBDT 强基线 → deep（wide&deep、two-tower hybrid）。Impression 日志训练；负采样；**按时间切**；debias（position/presentation）。Pointwise/pairwise/listwise；multi-task（多 engagement 目标）；实时特征 vs 批量特征。
  - 直觉 **wide & deep**：wide 部分是线性模型，记"见过的组合"（memorization：北京用户×火锅店就是爱点）；deep 部分是神经网络，学"泛化"（没见过的组合也能推）。为什么都要：纯 deep 会忘了高频组合的精确记忆，纯 wide 泛化不了。小例子：wide 记住"用户123×商家456=高频"，deep 推出"喜欢川菜的用户可能也喜欢湘菜"。

**系统设计**：社交图谱取候选 → 特征组装（feature store 批量预计算 + 实时计数器）→ ranker → 业务规则 + 广告插入。估算：QPS、feature store 读量、ranker 在 \~几百 ms 页面预算里的 slice。**按用户缓存候选集**、feature store 分片、ranker 无状态复制、**优雅降级**（挂了给缓存版/简化版 feed）。

## Ch4. 广告点击预测（pCTR）

**定义问题**：给 user+ad+context 估 P(click)。搜索/信息流/展示；**RTB 几十 ms**。十亿用户、几十亿曝光/天。指标：offline **log loss（calibration！概率进拍卖）** + AUC；online CTR、revenue/eCPM、广告主 ROI + calibration 检查。**这是校准概率问题，不止排序**。

**模型**：LR + hashed/crossed 特征（可解释基线）→ **FM / wide&deep / DeepFM**（高基数 ID 学 embedding）→ GBDT（备选强基线）。Feature hashing 控维度；crossing（如 用户分群 × 广告类目）；**校准**（Platt/isotonic）。Log loss；delayed conversion；高频重训。
  - 直觉 **FM / DeepFM**：FM 给每个特征学隐向量，交叉权重 = 向量内积，自动发现"年轻×游戏广告"这种组合；DeepFM = FM（记低阶交叉）+ DNN（学高阶组合），端到端一起训。为什么 CTR 标配：特征全是稀疏 ID，人工造交叉造不过来。
  - 直觉 **Platt / isotonic 校准**：模型排得对但概率不准？后处理来修。Platt 用 sigmoid 把分数映射到真实概率（假设单调 S 形）；isotonic 更灵活，分段拟合"分数→真实概率"的单调曲线。为什么需要：出价要真概率，重训模型太贵，校准层几分钟搞定。小例子：模型输出 0.7 的那批真实点击只有 0.5——isotonic 学出"0.7→0.5"的映射，上线前统一修正。

**系统设计**：ad request → targeting/过滤 → 特征组装 → CTR 服务 → 算 bid → 拍卖，全在几十 ms。预测服务**无状态复制**；热特征 co-locate/缓存；**超时走 default-bid fallback**（模型慢也不能丢拍卖）。估算：bid QPS、feature store 吞吐、模型容量。冷启动广告主、**反作弊/IVT 过滤**、新广告 exploration。

## Ch5. 民宿搜索排序（Airbnb 式）

**定义问题**：query（地点/日期/人数/过滤）→ 按 P(booking) 排序。**硬约束**（availability/容量/价格）vs 学出来的排序；**双边市场**（房客满意 vs 房东成功）。漏斗：search → click → booking request → confirmed。指标：offline AUC/log loss + NDCG；online booking 转化、revenue/search、满意度、**取消率 guardrail**。优化点击会推好看但订不到的房；优化 booking 更接近 revenue 但 label 稀疏+延迟。

**模型**：**GBDT workhorse**（表格数据强、稳、快）；残差拟合（residual = actual − predicted，shrinkage 步长）；校准；**按时间切**（季节性）。GBDT vs 深网：富文本/图像信号才值回 serving 成本。
  - 直觉 **GBDT 的残差拟合**：第一棵树随便猜，剩下每棵树都去拟合"真实值 − 当前总预测"（残差），一棵棵把误差啃掉；shrinkage（学习率）让每棵树只迈小步，防止某棵树带偏。小例子：房价预测，第一棵树猜均值 500 万，残差 +50 万 → 第二棵树专学这 +50 万的规律，100 棵树后残差接近 0。

**系统设计**：query 理解 + 硬过滤 → listing 索引召回（**geo 特殊**：空间索引、地图视窗查询）→ 特征组装 → ranker → 业务规则（diversity、促销、房东公平）。估算：搜索 QPS、索引大小、ranker 延迟。**Geo 索引分片**、热门搜索缓存、ranker 复制、listing 特征离线预计算 + **实时查 availability**。Follow-up：新房源冷启动、季节 spike、**availability 和排序索引的一致性**。

## Ch6. 外卖 ETA 预测

**定义问题**：回归——下单到送达几分钟。ETA 驱动用户预期 + 调度 + 骑手派单。**代价不对称**：迟到比早到伤。指标：offline MAE/RMSE + 看尾部分布；online 准时率、满意度、退款率。**Loss 要反映不对称**（RMSE 重罚大错；或直接不对称 loss）。点估计简单但藏不确定性；分位数/区间估计更实用但难评难 serve。

**模型**：**GBDT**（异构特征、非线性）；特征：距离/路线、备餐时间、骑手可用性/并单、时段、天气、路况。**按时间切**（小时/周/季节漂移）；离群值（极端延迟）；slice 分析（城市/餐厅/高峰）。

**系统设计**：订单事件 → 特征组装（静态 + 实时路况/运力）→ ETA 服务 → 调度/路由 → 用户 ETA，骑手移动持续更新。离线：完单日志重训、feature store、slice 验证。估算：高峰 throughput、feature store QPS、dispatch 窗口里的延迟预算。无状态副本、流处理实时特征、**区域部署**降延迟、**降级走启发式 ETA**。Follow-up：新区冷启动、surge  interact、**feedback loop**（ETA 会改变骑手行为）。

## Ch7. 总结

6 步框架套 5 个 case 的** recurring patterns**：推荐/搜索都是 **retrieve-then-rank 两阶段**；输出进拍卖/出价的都要**校准概率**；表格数据 **GBDT** 是 workhorse；feature store 桥接离线训练和在线 serving；扩展靠无状态复制 + 缓存 + 优雅降级。
面试口诀：澄清问题 + 指标 → 估算量化规模 → 点名 trade-off（accuracy/latency/cost）→ 收尾讲**瓶颈和故障处理**。

## 自测 4 问（带答案要点）
1. 这门课的 6 步答题框架？
   → problem → metrics → requirements → train/evaluate → high-level design → scale；先问澄清问题。
2. 哪两个场景必须输出校准概率、为什么？
   → 广告 pCTR（进拍卖算 bid）、推荐/定价分数进业务逻辑；AUC 只看排序。
3. 表格数据为什么 GBDT 是 workhorse？
   → 强、稳、快；残差拟合 + shrinkage；按时间切防季节泄漏；深网只在富文本/图像信号时才值回 serving 成本。
4. ETA 回归里 RMSE vs MAE 怎么选？
   → 迟到代价不对称，大错更贵 → RMSE 重罚大错；或直接不对称 loss；online 看准时率。
