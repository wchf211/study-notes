# QLoRA（M3 L9–L10 + 收尾）：4-bit base + 高精度 adapter

> **QLoRA = LoRA + 量化**。巨型冻住的 base 权重存 4-bit（内存 tiny）；只有小 LoRA adapter 用高精度训。**单卡微调大模型成为可能**。

## 1. 三组件

**4-bit NormalFloat（NF4）**：为正态分布数据设计的 4-bit 格式（模型权重经验上就是钟形、聚在 0 附近）。先把权重缩到 [-1,1]，再用 **quantile quantization**（选 16 个可表示值让每个 bin 分到等量权重——对正态数据理论最优）。32-bit→4-bit 存，精度损失 minimal。

**Double quantization（双重量化）**：把量化常数本身也量化。背景：量化要按 tensor 的绝对最大值 rescale（**quantization constant** c；X_int8=round(127/absmax(X)·X)），存 c 才能反推回来。**Outliers**（极少数大权重，带着重要信息）会浪费 bin，所以按 block 分别量化、各 block 有自己的 c——但 c 存多了也吃内存，于是 c 自己也被量化。

**Paged optimizers**：内存安全机制——NVIDIA unified memory 在 CPU/GPU 间倒数据，训练时的临时内存 spike 不 OOM。

**端到端**：base 4-bit NF4 躺着 → 计算前 double dequant 到 **bfloat16**（BrainFloat16，ML 圈流行的 16-bit 浮点）→ LoRA adapter 在高精度下训，4-bit base 永不动。课里 callout 强调的 subtle 点：**量化 rounding 是永久信息丢失——dequant 找不回来**。

## 2. Hands-on：同样 GSM8K，4-bit 版

和 L8 完全对照实验，只换 base 精度：`BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_use_double_quant=True, bnb_4bit_quant_type="nf4", bnb_4bit_compute_dtype=torch.bfloat16)`。其余全同：400 行 GSM8K、r=16/alpha=16/q_proj+v_proj/dropout 0.1、lr 3e-4、5 epochs、SFTTrainer、adapter 存 ./qlora。

实测：
- 内存 **5,591,548,160 字节（\~5.2 GB）**——8-bit 的 \~8.45 GB 再往下，unquantized \~30 GB 更不用说。dtype 显示 torch.uint8（4-bit 值的存储容器）+ torch.float16。
- 训前 4-bit 模型自然语言答题；训后吐 `<<48/2=24>>` / `#### 72` 数学式——adaptation 成功，同样有 repetition artifact（400 行+5 epochs 的 demo 债）。

**内存梯子（Llama 3.1 8B，课内实测）**：\~30 GB（float32）→ \~8.5 GB（8-bit）→ **\~5.2 GB（4-bit NF4）**。QLoRA 质量和 8-bit LoRA 相当——多压的几乎白送。

## 3. 收尾 + 跨课实操手册

课里 recap：微调基础与训练参数 → 技术 taxonomy 与选型 → 量化类型/工具/动手 → LoRA 机制/动手 → QLoRA 内部/动手。学完能动手：summarization、文本生成、虚拟助手、chatbot 的微调。

趋势（课里的 framing）：微调和 NLP/量化一起高速进化；会高效 PEFT 的工程师需求在涨。
建议的 next：diffusion（图像生成）、LangChain（模型+工具链）、vector DB（多模态 app）、prompt engineering。

**跨课实操手册**（读者一定会问的）：
- **LoRA vs 全微调**：默认 LoRA/QLoRA——便宜、快、无灾难性遗忘、adapter 小可换（serving 一 base 多 adapter）。全微调只在**数据大 AND 分布很 alien**（新语言、医疗/法律重度 domain）时考虑。需求本质是"查文档回答"→RAG 优先于一切微调。
- **数据集**：杠杆最高的一步。干净、结构化、任务代表性强，**格式=推理时想要的格式**（GSM8K 例子训的就是目标记法）；切 train/test；tokenize 截断+pad 定长；tokenizer 没 pad token 就拿 eos 顶。400 行是 demo，真干要多得多。
- **超参起手**：rank 16–32、alpha=rank、dropout 0.1（≤13B；33B+ 调低）、Llama 系 target q_proj/v_proj、lr 3e-4、auto batch、5+ epochs。任务远离预训练→加 rank；想往自己数据推得更狠→alpha 相对 rank 调大。
- **评估**：代表性 prompt 看训前训后对比；repetition 这类 artifact=欠训信号；fine-tune→evaluate→refine 循环——held-out test 上看任务精度，迭代数据量/epoch/rank/alpha。
- **Serving adapter**：只存 adapter（save_pretrained），base 冻住+量化，AutoPeftModelForCausalLM + BitsAndBytesConfig 按需 load——**一个 base 便宜 serve 多任务**。
- **内存预算**（8B 实测）：float32 \~30G → 8-bit \~8.5G → 4-bit NF4 \~5.2G。**选你质量线能忍的最低精度**。

## 自测 4 问（带答案要点）
1. QLoRA 三组件各解决什么？
   → NF4（4-bit 存）、double quant（常数也压）、paged optimizers（防 OOM spike）。
2. 8B 内存梯子三档？QLoRA 相对 8-bit LoRA 质量？
   → 30→8.5→5.2 GB；相当，多压几乎白送。
3. Dequant 为什么找不回信息？实操含义？
   → rounding 永久丢失；精度选型是单向门，质量线说了算。
4. 默认选型链（RAG/LoRA/full）？
   → 查文档→RAG；一般微调→LoRA/QLoRA；大数据+alien 分布→full。
