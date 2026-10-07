# ML 面试 IV：广告预测 + 反欺诈 — 提炼笔记（Ch8–Ch9，2026-10-06）

> 自己的话总结。MLE 视角：这两个都是"极端不平衡 + 延迟敏感 + 反馈闭环"的代表题。

## Ch8. 广告预测（pCTR）

**问题定义**：预测点击率 pCTR → 按 **expected value ≈ pCTR × bid** 排序进拍卖；计费 CPC/CPM/CPA。两阶段 serving：candidate gen（几千）→ ranking。硬约束：端到端 <\~100ms、打分 <\~50ms、巨大 QPS、**新鲜度**（广告/ campaign 不停换 → 高频重训）。口径：目标不止"预测点击"——revenue vs 广告主 ROI vs 用户体验。

**指标**：AUC-ROC 管排序，但**不够**——预测概率要进拍卖算钱。**Calibration**：预测均值/实际均值 ≈ 1；**AUC 对绝对 miscalibration 是瞎的**（排得对但每单都定错价）。Log loss 看概率拟合；accuracy 在点击不平衡下禁用；lift 看相对基线提升；online 用 CTR/revenue/ROI 做 A/B；永远谈 offline-online gap。

**架构**：训练（LR、FM、GBDT、DNN 吃点击日志）→ 候选召回 → ranker 算 pCTR×bid → serving。**负采样**（留 \~5–10% 负样本或 \~1:5）缩数据，**再 recalibrate**；**online learning**（FTRL）保新鲜；explore/exploit；**feedback loop**（模型只从自己 serve 过的广告学到东西——selection bias）。归因窗口 1–7 天；重训 daily/hourly。复杂度 vs 50ms 打分预算。

**特征**：user、context（页面/设备/时间）、广告属性、**历史 engagement**（past CTR 最强信号）、交叉特征。高基数 ID：hashing trick 或 learned embedding。特征选择：chi-square、MI、permutation importance、drop-column；看 collinearity。**两条铁律**：point-in-time 正确（不许用未来信息）、feature store 保 train/serve 一致。

**训练数据**：点击归因窗口（如 7 天）→ **delayed feedback**。Bias：sample selection（只在 serve 过的广告上训）、position bias（头部天然点击高）。Downsample 后按比例 w 回调（q 由 p 和 w 算）。**Exploration 流量买无偏数据**；按时间切防 temporal leakage。

**Ad Selection**：漏斗——几千 eligible → 轻量召回 → 重 ranker 按 expected value 取 top-K。Latency 分段预算、frequency capping、广告主 budget pacing、diversity/UX guardrail、exploration（ε-greedy/UCB/Thompson）让新广告有曝光。Relevance vs revenue vs UX，三者在 100ms 里平衡。

**Ad Prediction**：pCTR 模型必须**输出校准概率**（拍卖数学否则全错）、online learning 抗 drift、冷启动（prior + exploration）。监控 calibration drift；serving 又快又稳。面试三连问：**calibration、explore/exploit、feedback loop**。

## Ch9. 反欺诈

**问题定义**：实时抓欺诈交易/账户。类型：支付欺诈、盗号、chargeback、身份盗用、promo 薅羊毛、洗钱。约束：**毫秒打分**、**<0.1% 极端不平衡**、**对抗性**（骗子会变招）、监管。KPI：fraud loss rate、FPR、人工审核率、chargeback 率；precision/recall 直接 = 用户摩擦（误杀正常人）。

**指标**：precision/recall/F1；**AUC-PR 优于 AUC-ROC**（极端不平衡下）；FPR、alert precision、review rate、detection latency、**dollars-prevented**（成本加权的业务指标）；risk score 进下游决策时要 calibration。阈值调 precision/recall/审核工作量三角。**Accuracy 是陷阱**：基线 0.3% 时 99.7% accuracy 可能一只 fraud 没抓到。

**架构**：双路——**实时**（流 ingest → velocity 特征/设备指纹 → 规则层 + ML → 决策：放行/挑战/拒绝）+ **批量**（ retrospective、趋势）。Kafka/Flink；feature store 快查；case management（人工审核）；**analyst label + chargeback 回流训练**；持续监控。支付授权要毫秒级。

**训练数据**：label 来自 chargeback/欺诈举报，**滞后 30–90 天**；point-in-time 特征正确性；**禁止 random shuffle**——按时间切 train/val/test（防 temporal leakage + 尊重 concept drift）；对抗性 drift 让老数据过期；防 post-outcome 信息泄漏。

**建模**：生产是 **portfolio 不是单一模型**，按决策点选型：
- **规则**（if-then guardrail，确定、可解释；有银行十年老规则还管 >50% 欺诈决策，监管信任）；
- **监督 ML**（树 ensemble——random forest/GBM——统治表格欺诈数据；LR 求快）；
- **混合**（规则过滤 obvious case，ML 打 ambiguous；有时规则后置 override ML）；
- **异常检测**（抓新型攻击，FP 高——**预警层不当决策者**）；
- **图模型**（欺诈团伙、mule 网络，shared-device、GNN；强但实时贵）。
- 不平衡处理：SMOTE vs undersample vs class weight，各有代价。阈值 0.5→0.8：precision 升、recall 降。**Drift**：监控 P/R/分数分布，weekly/monthly 重训，ADWIN/DDM/KS 检测器 + 业务反馈闭环。部署：实时 vs 批量、特征缓存、审计日志、**SHAP 可解释性**（给 analyst/监管看）。

## 自测 4 问（带答案要点）
1. AUC 高但 calibration 差，广告系统会出什么问题？
   → pCTR×bid 出价全错；calibration bias = 预测均值/实际均值 ≈ 1；AUC 只看排序不看绝对值。
2. 反欺诈为什么 accuracy 是陷阱？
   → 基线 0.3% fraud 时 99.7% accuracy 可能一只没抓到；用 AUC-PR、precision/recall、dollars-prevented。
3. 广告/反欺诈训练数据的共同深坑？
   → delayed labels（点击 7 天窗口、chargeback 30–90 天）；selection bias（只从 serve 过的学）；必须按时间切，random shuffle 禁止。
4. 反欺诈建模为什么是 portfolio 不是单一模型？
   → 规则（可解释、监管信任）→ 树模型（表格数据之王）→ 异常检测（抓新攻击、高 FP 预警层）→ 图模型（团伙）；按决策点选型。
