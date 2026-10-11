# 收尾（Ch5）：路线图

> 总结课，无新概念。弧线：ML 地基→数据驱动决策→库与预处理→三范式→传统 vs 深度→视觉+NLP 应用。

课里指的下一站：**Machine Learning with NumPy, pandas, scikit-learn, and More**（\~15h，intermediate）——动手编码深度。

定位：这门课是**词汇+地图**，实现能力去 follow-up 课里建。

**Project（只读了介绍，没跑代码）**：Fake News Detection with scikit-learn——新闻真假二分类。Kaggle 标注数据 + News API 拉的实时新闻 → **TfidfVectorizer** 转数字 → 切分训练测试 → 训分类器 → held-out 评估。教科书式 supervised binary-classification NLP：TF-IDF + sklearn，正好是 L4–L6+L11 教的那套。

## 自测 4 问（带答案要点）
1. 这门课的定位？下一步？
   → 词汇+地图；15h 动手课建实现能力。
2. Fake news 项目的技术栈对应哪几课？
   → TF-IDF（L11）+ sklearn 分类（L4–L6）。
3. 学完这门课，你能给 ML 任务选范式了吗？（开放）
   → 有标签→监督；无标签找结构→无监督；序列决策→强化。
4. 传统 ML 什么时候打败深度学习？（开放）
   → 中小结构化数据+要可解释+算力紧。
