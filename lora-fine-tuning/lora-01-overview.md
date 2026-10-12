# 微调入门（M1–M2）：为什么微调、workflow

> 预训练模型是通才，问它 niche 问题就泛泛而谈或 hallucinate。从零重训贵到离谱——微调是中间路：留住通用知识，加上专业化。

## 1. 课程地图

教：拿巨型预训练模型（running example：Meta Llama 3.1，80 亿参数），用 **LoRA**（只训一小撮新增权重，不动原模型）+ **QLoRA**（LoRA + base 模型存 4-bit）+ **quantization**（权重压到更少 bit，塞进小硬件）做便宜 specialization。

前置（课里列的）：Python + PyTorch + Hugging Face Transformers；ML/生成 AI/深度学习常识；transformer 架构；tokenization（切词）/stemming（词根还原）/**embeddings**（词的稠密数字向量）。

**PEFT**（parameter-efficient fine-tuning）：只训参数子集/新增参数，训练便宜、预训练知识不丢——全课的方法论地基。

## 2. 什么是 Fine-Tuning

**Fine-tuning**：拿别人预训练好的模型，在你自己的小任务数据集上继续训，调它的 **weights**（组成模型的几十亿数字），让它专精。

三个必懂词：
- **Catastrophic forgetting（灾难性遗忘）**：新训练把旧技能洗掉——像升级手机丢了所有设置；
- **Underfitting**：没学进去（书翻一遍，啥都不会）；
- **Overfitting**：背下训练答案，新东西全挂。

Transformer 速览（课里铺的背景，微调正好 exploit 它）：input embeddings（文本→数字）→ positional encoding（注入词序，模型本来没顺序感）→ encoder 层（**self-attention**：每个词看别的词定语境；**multi-head**：多个 attention 视角并行）+ feed-forward + add-&-norm 残差（shortcut+归一化保训练稳定）→ decoder 层（masked attention 不能偷看未来 + 对 encoder 输出的 attention）→ linear 映射到词表 logits → softmax 转概率，选最可能的下一个 token。**浅层学通用 pattern，深层学 specifics——微调 exploit 的正是这个分层**。

**医疗 chatbot 场景**（课里的 motivation）：医院要 chatbot 按病情/病史/基因/生活习惯写个性化治疗方案。naive 做法（病历全塞 prompt）几个病人行，几百万病人=GB/TB 级 context，每次 query 全扫一遍，又贵又慢、精度还掉、context 保鲜成全职工作。从零训？贵且洗掉预训练知识。**微调=中间路**。

微调的好处：任务精度高；省（小数据、训得快）；适应新数据；scale 便宜；**knowledge retention**（学新技能不灾难性遗忘）。

**训练参数白话**：
- **Batch size**：一轮训几个例子——大跑得快但吃内存；
- **Epochs**：全数据集刷几遍——少了 underfit，多了 overfit；
- **Iteration**：一 epoch 要几个 batch（=总数/batch size）；
- **Learning rate**：每次权重更新步子多大——小稳但慢（要多 epoch），大快但可能 overshoot。

**微调六步 workflow**：1) 选预训练模型（任务/资源/预算匹配；开源 Llama/BERT/Mistral vs 闭源 ChatGPT/Gemini）；2) 准备任务数据集——**这步决定一切**；3) 预处理：清洗+切 train/test；4) 配训练参数（lr/batch/epochs）；5) 跑微调（学新东西，留住旧知识）；6) 在任务上评估+迭代。

铁律：**数据集质量是杠杆最高的决策**。当 prompt 塞数据在量/延迟上扛不住、但数据又没 alien 到要重训时，微调就是对的工具。

## 自测 4 问（带答案要点）
1. 微调 vs 从零训 vs prompt 塞数据，三者的位置？
   → 塞数据：量小延迟不敏感；微调：中间路，留知识加专精；从零：贵且洗知识。
2. Catastrophic forgetting 是什么？PEFT 怎么躲它？
   → 新训洗掉旧技能；只动子集/新增参数，base 冻结。
3. Underfitting vs overfitting？epochs 怎么调？
   → 没学会 vs 背下训练集；少了前者多了后者。
4. 六步 workflow 里哪步杠杆最高？为什么？
   → 数据集准备；垃圾进垃圾出，参数调不回来。
