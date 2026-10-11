# 应用（Ch4）：计算机视觉 + NLP

> 两大旗舰应用。CV=理解图像，NLP=理解语言。共同实操：**别从零训大网，fine-tune 预训练的**。

## 1. 计算机视觉

**图像处理 vs 计算机视觉**：前者把图变好（去噪/增强），后者理解图（里面有啥、在哪）。

数字图像：**pixel**（最小单位，一个颜色值）网格；灰度=每像素一强度，彩色=RGB 三通道；对计算机图像就是矩阵（彩色是 3D tensor）。

**任务分类**（各是不同的活）：
- **Image classification**：图里有啥（一图一标签）；
- **Object detection**：有啥+在哪（bounding box；R-CNN 系、**YOLO**"you only look once"实时级）；
- **Semantic segmentation**：每个 pixel 按类着色（所有"路"一个色）；
- **Instance segmentation**：分开每个个体（车 1 vs 车 2）；
- **Facial recognition**：先 detect 脸再认谁；
- **OCR**：从图里读字；
- **Image generation**：生成侧。

**CNN 管视觉**：卷积层学空间层级（边缘→纹理→部件→物体），pooling 降采样拿平移不变性+省算力。
**Transfer learning 实操**：拿 ImageNet（百万级标注图，VGG/ResNet 架构）预训练网，在自己小数据集上微调——小数据从零训必死。

古典技术（前深度时代，仍值得知道）：Sobel/Canny 边缘、Hough 找线圆、**SIFT/SURF** 手工关键点、Haar cascades 快检脸。

**数据现实**：视觉模型饿标注，标注贵。**Data augmentation**（旋转/翻转/裁剪/调亮度造新样本）是标准解——还让模型对真实变化鲁棒。

实操：**detection（在哪）比 classification（是啥）贵**；小数据永远 fine-tune 不从零训；augment 往死里加。

## 2. NLP

**为什么难**：歧义（"bank" 河岸还是银行）、context 依赖、讽刺习语俚语。

**NLP 管线**：拿文本→清洗（小写、去标点、去 **stopwords**"the"这种没信号的高频词）→ **tokenize**（切 token：词/子词/字；子词对生僻词友好）→ **stem/lemmatize**（还原词根：stemming 粗暴砍后缀 running→runn，lemmatization 查字典 running→run）→ POS 标注→句法 parse→向量化。

**文本转数字**（老到新）：
- one-hot：每词一巨稀疏向量——没语义；
- **Bag of Words**：每文档词频——简单但丢词序；
- **TF-IDF**=词频×逆文档频率——本文高频+全局稀有的词（如 quasar）权重高，"the" 沉底；
- **word embeddings**：稠密向量，语义近的坐得近（Word2Vec/GloVe；king−man+woman≈queen 名场面）；
- **contextual embeddings**：BERT 式——同词不同句不同向量。

**经典任务**：sentiment（褒贬）、**NER**（named entity recognition，找人名地名机构）、机器翻译、摘要（**extractive** 摘句子 vs **abstractive** 重写）、问答、聊天、**LDA**（Latent Dirichlet Allocation，挖文档集的隐藏主题）。

**语言模型**：n-gram（看前 n−1 个词猜下个——简单但稀疏饿死）；神经 LM；**Transformer+self-attention**（每个词掂量所有词的分量——可并行，RNN 不行）。**GPT**=生成式 decoder（逐 token 往后写）；**BERT**=双向 encoder（mask 词训练，理解/分类强）。

**评估看任务**：分类看 accuracy/F1；翻译看 **BLEU**（n-gram overlap，precision 味）；摘要看 **ROUGE**（overlap，recall 味——reference 内容抓到没）；语言模型看 **perplexity**（对真实文本多"惊讶"，越低越好）。

实操：今天多数文本分类=**微调预训练 transformer**，别从 TF-IDF 往上造；tokenize 和清洗认真做（垃圾 token 进垃圾向量出）；指标配任务（翻译 BLEU、摘要 ROUGE、分类 F1）。

## 自测 4 问（带答案要点）
1. 图像处理 vs CV？detection 比 classification 贵在哪？
   → 变好 vs 理解；detection 还要定位（box）。
2. Transfer learning 为什么是视觉实操铁律？augmentation 干嘛？
   → 小数据从零训必死，拿 ImageNet 预训练微调；造样本+抗真实变化。
3. TF-IDF 白话？word embedding 相对 BoW 强在哪？
   → 本文高频×全局稀有加权；稠密+语义近坐得近（king−man+woman≈queen）。
4. GPT vs BERT？BLEU vs ROUGE vs perplexity 各看什么？
   → 生成 decoder vs 理解 encoder；翻译/摘要/语言模型困惑度。
