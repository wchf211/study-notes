# LoRA（M3 L7–L8）：只训两个小矩阵

> **LoRA（Low-Rank Adaptation）**：微调不动原权重。每个权重矩阵配两个 tiny 矩阵，乘积近似"任务需要的改变量"；只训这两个小矩阵（"adapters"），原权重冻结，训完 merge 回去。**Rank** = 小矩阵的内维——adapter 能装多少信息（精度），也=你要训的参数量。

## 1. 为什么：全微调又慢又贵

B 级参数全训慢且贵；LoRA 只训一小撮（Hugging Face 2023 年出了 **PEFT** 框架统管这类方法），GPU 消耗砍到约 1/3。base 冻结→**天然躲开 catastrophic forgetting**。bonus：一个 base 配多个小 adapter 换着用——serving 的赢（多任务一 base 多 adapter）。

**数学**：权重矩阵 W（m×n），LoRA 把更新分解成 A（m×k）× B（k×n），k=rank，k≪m,n。只训 A、B。训完有效权重 ΔW = W + A′B′（带下面的 scaling）。

省多少（课里的表）：5×5（25 参数）→ adapter 10（40%）；10×10→20/100（20%）；100×100→200/10,000（2%）；1000×1000→2,000/1,000,000（**0.2%**）。**矩阵越大省得越狠——LoRA 在巨模型上最香**。

## 2. 三超参

**Rank**：看四样定——模型大小（大模型 base 本来就懂很多，小 rank 够；小模型要高 rank）、任务复杂度（简单低 rank，复杂高 rank）、算力预算（rank 高=参数多=慢）、要的精度。经验：
- 1–8：通用下游（情感分析、问答）；
- **16–32：多数任务的平衡点**；
- 32+：复杂+远离预训练分布（医疗/法律文档、多语）。

**Alpha**：scaling 旋钮——adapter 更新 merge 前乘 alpha÷rank。alpha>rank 放大微调更新（数据和预训练差异大时用）；alpha<rank 压住它（多靠预训练知识）；**alpha=rank → scale=1，简单数据集的默认**。

**Dropout**：**regularization**（每 batch 随机 zero 掉一些 adapter 参数，逼模型泛化不背诵）。课里 0.1（10%）算"高"、0.05（5%）算"低"；太高 underfit。QLoRA paper 的经验：小模型（≤\~13B）dropout 高点，大模型（33B/65B）低点。

**坑**：rank/alpha 没有银弹，每个任务都得试；reparameterized 方法在复杂数据集可能 underfit；dropout 太高学不动。

**起手式**：rank 16–32、alpha=rank、dropout \~0.1（≤13B）；任务远离预训练分布再加 rank/alpha。target 选 attention 的 q_proj/v_proj（Llama 系标准答案）；各模型模块名查 huggingface/peft 仓库。

## 3. Hands-on：Llama 3.1 8B 学会只吐数学式

目标：Llama 3.1 8B 平时解数学应用题是话痨自然语言；训完只吐纯数学表达式。数据：openai/gsm8k（**GSM8K**=Grade School Math 8K，\~8.5K 小学应用题多步推理；答案有 `<<48/2=24>>` 这种记法，结尾 `#### 72`）。

管线（每步实测）：
1. 装库：transformers 4.44.1、accelerate、bitsandbytes 0.43.3、datasets 2.21.0、trl 0.9.6、peft 0.12.0；HF CLI 登录。
2. 8-bit load base（BitsAndBytesConfig），确认"训前"行为：Natalia 发夹题→分步 prose 回答。
3. 预处理：load gsm8k（main split），tokenizer pad token 设成 eos，question/answer 列 tokenize（截断+pad 到定长 100），**只留 400 行**（demo 快）。
4. LoRA config：r=16、lora_alpha=16（scale=1）、target_modules=["q_proj","v_proj"]（attention 的 query/value 投影，Llama 标准 target）、lora_dropout=0.1（8B 算"小"）、bias="none"（不训 bias）、task_type="CAUSAL_LM"（**causal LM**=预测下一个 token，生成模型标准目标）。
5. TrainingArguments：checkpoint → ./lora，auto_find_batch_size=True（让库自己找塞得下的 batch），learning_rate=3e-4（0.0003，速度稳定平衡），num_train_epochs=5。
6. trl 的 **SFTTrainer**（supervised fine-tuning trainer）+ DataCollatorForLanguageModeling(mlm=False)（不用 masked LM 目标，纯 next-token）。
7. 存 adapter（./lora/lora_model）。
8. peft 的 AutoPeftModelForCausalLM + 8-bit config  reload 推理。

结果：同一道题现在吐 `48/2 = <<48/2=24>>24 … #### 72`——行为改过来了。但也有 **repetition artifact**（`#### 72` 重复 N 遍）——欠训的经典相。

**课里明说**：400 行+5 epochs 是 demo 级——结果怪（repetition）是预期的；真用要加数据、加 epoch、搜超参。

**Serving 笔记**：存下来的 adapter 小而可换——**一个冻住的 base + 按需 load 任务 adapter**。

## 自测 4 问（带答案要点）
1. LoRA 白话+数学？为什么矩阵越大越省？
   → 两个小矩阵乘积近似更新量；省的比例随矩阵变大（1000² 省到 0.2%）。
2. 三超参起手式？alpha=rank 什么意思？
   → rank16–32/alpha=rank/dropout0.1；scale=1 平衡默认。
3. 为什么 target q_proj/v_proj？repetition artifact 说明什么？
   → attention 投影是 Llama 标准 target；欠训（demo 级数据/epoch）。
4. LoRA 的 serving 优势？
   → 一 base 多 adapter，按需换；天然无灾难性遗忘。
