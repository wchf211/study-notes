# 工具栈（Ch2）：库 + 预处理

> 实战铁律：算法没人手写，都是调库；**预处理是真实 ML 工作的大头**——跳过它，模型训练时看着 fine，上线就挂。

## 1. 库：分层栈

- **NumPy**：数值地基——n 维数组+快速数学；几乎所有 Python ML 库都站它肩上。
- **pandas**：数据摆弄——**DataFrame**（内存里的 2D 带标签表，像 spreadsheet）、**Series**（一列）；清洗合并聚合。
- **scikit-learn**：经典（非深度）ML 的 workhorse——分类回归聚类特征提取评估，一个统一 API。**不是神经网络，先到这**。
- **TensorFlow**（Google）：大规模深度学习，data-flow graph；Lite（手机/边缘）、TF.js（浏览器）。生产级部署强。
- **PyTorch**（Meta）：动态计算图（网络结构能边跑边变），好 debug——研究/CV/NLP 主流。
- **Keras**：高级易用 API，站 TensorFlow 上；快但细粒度控制弱。
- **Spark MLlib**：数据装不进单机时——分布式分类回归聚类协同过滤。
- **H2O**：可扩展开源 + **AutoML**（自动选模型调参）+ 非程序员 Web UI。
- R 系（ggplot2 画图、dplyr 数据操作、caret/mlr 建模）、Julia 系（Flux.jl、MLJ.jl、DataFrames.jl）。

选型：默认 Python（scikit-learn → PyTorch/TensorFlow）；内存装不下→Spark；要快出 baseline→H2O/AutoML。

## 2. 预处理管线：collect → clean → format → reduce → scale

**数据清洗**：修错值/缺失/离群/重复。pandas：`drop_duplicates`、`dropna`（删缺失行）、`fillna`（均值/中位数/众数填）、`isnull`（找洞）；sklearn imputers（SimpleImputer、KNNImputer——拿相似行填）。OpenRefine/DataCleaner：可视化点选清洗。

**Feature selection（特征选择）**：只留相关的。为什么：特征少→模型简单、训得快、**overfitting**（记住训练噪声、新数据拉胯）轻、人类能看懂。三家：filter（统计检验排名，快）、wrapper（子集实测留最好，准而贵）、embedded（训的时候顺手选，如特征重要性）。工具：SelectKBest、**RFE**（递归特征消除——反复训、每次砍最弱）。

**Feature extraction（特征抽取）**：从原始输入造新特征。文本：CountVectorizer/TfidfVectorizer（文档转数字）。**降维**：**PCA**（投影到方差最大的方向，少维度留大信号）打 **curse of dimensionality**（维度一高数据变稀、模型变傻）。

**Feature scaling（特征缩放）——常被跳过但关键**：KNN/SVM/梯度下降是距离/步长敏感的，"几千"量级的特征会淹死"零点几"的。
- **standardization**：零均值单位方差（数据近似钟形时好）；
- **normalization**：压到固定区间如 0–1（有界/偏斜时好）；
- RobustScaler：中位数+四分位距，离群点带不偏。
- 工具：StandardScaler、MinMaxScaler。

**类别编码**：**one-hot**（每类一二进制列——干净但高基数如 ZIP 会炸维度）vs **label encoding**（每类一整数——省但暗示顺序，只给真有序数据）。

**Binning**：连续变量切区间（如年龄→年龄段），更可解释有时更鲁棒。

**数据切分**：**永远别在训练数据上评估**。标准 70–80% 训、余下测；validation 集（或交叉验证）调参。**K-fold cross-validation**：切 k 份，轮流一份验证、k−1 训，分数平均——更稳的真实性能估计，代价 k 次训练。

**Pipelines**：预处理链+模型封成一个对象，保证训练和新数据走**完全相同的变换**——防 **data leakage**（测试集信息漏进训练，如在测试集上 fit scaler，性能虚高）。ColumnTransformer 给不同列配不同变换。

**Imbalanced data（类别不平衡）**：99% 正常 1% 欺诈时，**accuracy 是谎言**——全猜正常也有 99%。修：oversample 少数类（**SMOTE**——在真实少数样本之间插值造新的）、undersample 多数类、class weights（少数类错罚更重）、转异常检测；**看 precision/recall/F1 别看 accuracy**。

## 自测 4 问（带答案要点）
1. 预处理五步？为什么它是"大头"？
   → collect/clean/format/reduce/scale；跳过=训练 fine 上线挂。
2. Standardization vs normalization？什么时候必须 scale？
   → 前者零均值单位方差（钟形），后者压区间（有界/偏斜）；距离/步长敏感算法前必须。
3. Data leakage 是什么？pipeline 怎么防？
   → 测试集信息漏进训练；链式封装保证变换一致。
4. 类别不平衡时为什么 accuracy 是谎言？看什么？
   → 全猜多数类也高分；看 precision/recall/F1；SMOTE/class weights 修。
