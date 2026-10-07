# ML 面试 V：Hate Speech + 动态定价 — 提炼笔记（Ch10–Ch11，2026-10-06）

> 自己的话总结。这门课收工。

## Ch10. Hate Speech 检测

**问题定义**：policy 驱动的二分类（hateful/non-hateful），阈值可调；可扩展多分类。先定 policy 再建模；precision/recall 姿态看产品阶段；实时打分（秒级）+ 海量 +  spike；多语言；**人必须在环**（edge case + appeal）；语言演化 → 定期重训。**全自动化不可能**（context/intent/文化nuance 本质模糊）。

**指标**：<5%（甚至 <1%）hateful → accuracy 无意义（2% hateful 时全判负也有 98%）。**Precision**（少误伤、少 appeal，早期建信任）vs **recall**（多抓 harm，保护弱势群体；代价 FP）。F1 藏了哪种错占主导；**PR-AUC > ROC-AUC**。阈值映射动作：<0.3 放行、0.3–0.6 软标/降权、0.6–0.85 人工审、≥0.85 硬删；按用户历史/地区/话题动态调。人工指标：reviewer agreement、**appeal reversal 率**（10% 翻案=有问题）、解决时效 SLA。

**架构**：流式管线——ingestion（Kafka/Flink/Spark，1 万–100 万+ posts/s）→ 预处理（5–20ms：归一化、emoji、unicode、语言 ID、SimHash 去重）→ **两级 cascade**（蒸馏模型过滤简单 case，全量模型处理 uncertain，p99 <100ms）→ 决策引擎（删/标/shadow ban/封号，阈值 A/B 测）→ 人工审 → 反馈闭环。**Redis 缓存重复内容**（延迟降 60–80%；2–5% 是重复 repost）。**Active learning** 只标 0.4–0.6 uncertainty 带；周/双周重训；drift 监控（"静默退化比 loud failure 更糟"）；对抗防御（homoglyph/零宽字符）；PII 脱敏、30–90 天 retention；**优雅降级**（故障时预设 fail 模式：deny/allow/shadow ban 三选一）。

**训练数据**：数据质量 > 模型选择。公开集 + 平台日志 + 众包 + 合成，**provenance 追踪**防 bias。标注：二分类 vs 多分类 vs context-aware；**\~20% 标注分歧是正常的**，测 IAA。别 over-normalize 洗掉信号；保留 context（回复链、反讽）。不平衡：oversample、分层 undersample、SMOTE（语义验证 caveat）、**weighted loss**（hateful miss 罚 5 倍）。

**建模**：**start simple 再 scale**——LR/SVM/RF（快、可解释）→ LSTM/GRU/CNN → BERT/RoBERTa/DistilBERT（最好但贵；distill/prune/quantize 上线；SHAP/LIME 解释）。生产形态：**轻 cascade + 重二审**。Weighted CE：−w_hate·y·log(ŷ) − w_non·(1−y)·log(1−ŷ)（5% hateful → w_hate=10）；**focal loss** FL = −α(1−p)^γ·log(p)。加 fairness 指标；可 multi-task。

## Ch11. 动态定价

**问题定义**：定价是**不确定性下的决策**，不是预测——模型会污染自己的训练数据。**Feedback loop**（今天的价格 bias 明天的数据；把自身影响误读成 ground truth → overpricing/震荡）。模型只输出**信号**（需求曲线、弹性），**policy 层**定价。Framing 按成熟度：需求预测（好调试，早期）→ 转化率/WTP（个性化，公平性风险）→ 弹性估计（要 identifiability、自然实验、A/B 价格测试）→ contextual bandit/RL（生产 RL 通常是受约束的 bandit）。演进：supervised → bandit → RL。约束放 policy 层：监管（不歧视，含 proxy 变量）、伦理（灾难不 surge）、运营（最低 margin、±10% 涨跌幅、日更一次）。粒度：全局 → 分群 → 个人（理论上最优，稀疏/公平/风险）。

**指标**：**portfolio**，单个都会骗人。Revenue = Σ(P×Q) 骗人（$10→$12 订阅涨价 revenue 涨、retention 掉）；配 RPU、**ASP**（revenue/units；ASP spike 抓定价事故）。Profit = (P−C)×Q；**margin = (P−C)/P** 做硬约束。**Elasticity** = %ΔD/%ΔP（−2：涨价 10% 需求掉 20%；必需品接近 0）。Guardrail：转化率、churn/retention、**价格波动**（日价格标准差、限调价频率、rolling 平均）。探索代价：**regret** = 最优 revenue − 实际（$20 例）。Offline（仿真 revenue、MSE）必要不充分——新价格下行为会变；online：A/B、bandit、**分阶段 rollout**（10% 城市）。

**架构**：control plane，六层——(1) ingestion/feature store（Kafka ms 级、Airflow 批量；Amazon 单品 200+ 特征）；(2) 模型层（GBT/NN/RL；表格数据树模型统治；**输出概率不是价格**）；(3) 业务逻辑（margin cap、公平、区域规则；风险超阈**规则 override ML**）；(4) 定价决策（REST/gRPC、Redis/Memcached 缓存，<100ms）；(5) 监控（Prometheus/Grafana；ADWIN/KS 测 drift；** subtle drift 偷走 2–5% revenue**；自动重训/回滚）；(6) 实验（shadow、A/B、canary；大厂同时跑几十个）。

**训练数据**：每行 = （context, action=price, outcome），**永远只能观测到实际展示的那个价格的结果**。Context 当 contract 记（库存、时间、季节、竞品、分群、设备、促销 + policy metadata，别让模型只模仿老规则）；记**精确展示价**（含折扣/取整）+ 模型建议价 vs 实际执行价；outcome 延迟/截断 → 多 label（购买、件数、revenue、margin、time-to-purchase）+ 观察窗。**Confounding**：高峰高价 ≠ 价格导致需求。**Counterfactual gap**（>90% 价格-context 组合从没观测过）+ selection bias → **刻意 exploration**（价格桶、随机折扣、A/B）——这是跟业务谈判不是调参。冷启动：类目 prior、仿真需求曲线、保守默认。Shadow mode 跑几个月。

**建模**：模型估计**需求对价格的响应**，不预测"正确价格"。树（可解释、噪声大）→ RF（稳、延迟高、外推差；离线 benchmark）→ **GBDT 最常用**（sequential 纠错、**单调约束**"涨价需求不涨"、LightGBM/XGBoost 毫秒打分、SHAP 解释；打候选价，policy 层加规则）。线性/逻辑回归给监管场景。NN 很少当表格核心（难约束难解释）。**RL 谈得多部署少**（探索烧 revenue 和信任，反馈延迟）。

## 自测 4 问（带答案要点）
1. Hate speech 里 precision vs recall 怎么摆？
   → <5% 正样本 accuracy 无意义；早期 precision 建信任（少误伤），安全系统 recall 优先（多抓 harm）；阈值映射动作（0.3/0.6/0.85）。
2. 广告 calibration 和定价 regret 有什么共同点？
   → 都是"模型输出进决策"：pCTR×bid 要校准概率；定价用 regret 衡量探索代价；offline 指标都不够，要 online 验证。
3. 定价的 feedback loop 为什么危险？
   → 今天的价格污染明天的数据；把自身影响误读成 ground truth → overpricing/震荡；模型只输出信号，policy 层定价 + 约束。
4. 反欺诈和 hate speech 的人工环节有何不同？
   → 反欺诈：case management 审可疑交易（毫秒决策 + 事后审）；hate speech：分级 SLA（4h/24h/48h）+ appeal reversal 率监控；都是 human-in-the-loop 但节奏和指标不同。
