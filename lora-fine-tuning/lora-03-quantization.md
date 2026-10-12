# Quantization（M2 L4–L5）：把模型压小

> **Quantization（量化）**：每个权重用更少的 bit 存——float32（4 字节/权重，默认）→ int8 → int4。文件小、内存少、每次推理算得少，精度只掉一点点。

## 1. 为什么：模型在膨胀

课里点的名：ChatGPT-4 约 1.7T 参数、Llama 3.1 405B、Gemini 1.0 Ultra 175B、Bloom 176B、Mistral 7B。训和 serve 的内存成瓶颈——**量化是让微调和部署在现实硬件上可行的东西**。

好处：塞进受限设备（手机/手表/医疗设备）；推理快（实时场景）；跨硬件 tier 好 scale；延迟低（自动驾驶/同传）。

## 2. PTQ vs QAT

- **PTQ（post-training quantization）**：训完再压，推理时用。快而简单，掉点精度。适合：快速部署、边缘设备、IoT 摄像头/传感器、嵌入式/医疗。
- **QAT（quantization-aware training）**：训练时就量化，让模型学会在低精度下活。精度好，算力贵。适合：精度不容商量——CV（分类/检测/分割）、紧凑 NLP、自动驾驶。

## 3. 四件套工具

- **GGML**：C 库，专为量化 Llama 让它跑在更广硬件（尤其 CPU）；**k-quants** 系统——按 gradient 大小给不同权重配不同 bit 宽；
- **GGUF**：GGML 继承者，推广到更多模型家族；
- **GPTQ**：逐层 PTQ，把每层权重转 2/3/4/8-bit int（4-bit 最常用），选让"原层输出 vs 量化层输出"平方误差最小的值；Hugging Face AutoGPTQ 可用；
- **bitsandbytes**：**NF4**（4-bit NormalFloat）格式存 4-bit，计算时 dequant 回 16-bit；Hugging Face transformers 直接可用。

课里的分类：GGML/GGUF/GPTQ 算 PTQ 系，bitsandbytes 算 QAT 系（本课就用它做量化）。

## 4. Hands-on：Llama 3.1 8B 全精度 → 8-bit

Setup（照着能复现）：Hugging Face 账号+token（Access Tokens 页建）+ Meta 的 gated 模型 `meta-llama/Meta-Llama-3.1-8B-Instruct` 审批（模型页填表，约 1 小时批）。库：transformers 4.44.1、accelerate、bitsandbytes 0.43.3、huggingface_hub[cli]。`huggingface-cli login`（问存 git credential 答 y；非交互用 `--token` 或 `HF_TOKEN` 环境变量）。

实测数：
- Baseline：全 torch.float32，`model.get_memory_footprint()` = 32,121,053,440 字节（**\~30 GB**）。prompt "Portugal is" → 通顺事实段落。
- `BitsAndBytesConfig(load_in_8bit=True)` 经 `quantization_config` 传给 `from_pretrained`（+ `device_map="auto"` 自动把层铺到可用 GPU/CPU）：参数 dtype 变成 torch.int8 + torch.float16 混合，footprint = 9,081,209,088 字节（**\~8.45 GB**）。同样 prompt，答案质量肉眼无差。

**8-bit 把内存砍到约 1/4，简单生成任务质量无可见损失**。

脚手架（后面课复用）：token 登录 → BitsAndBytesConfig → from_pretrained(quantization_config + device_map="auto") → AutoTokenizer → generate(max_new_tokens) → decode(skip_special_tokens)。

**内存数学记住**：8B 参数模型，4 字节/权重 ≈ 30+ GB；8-bit ≈ 8.5 GB。

## 自测 4 问（带答案要点）
1. Quantization 白话？为什么现在是刚需？
   → 少 bit 存权重；模型膨胀到 T 级，内存成瓶颈。
2. PTQ vs QAT？各适合？
   → 训完压（快，掉精度，边缘部署）vs 训时压（精度好，贵，CV/自动驾驶）。
3. 四工具各干嘛？本课用哪个？
   → GGML/GGUF（Llama 系 CPU 跑）、GPTQ（逐层 PTQ）、bitsandbytes（NF4，本课用）。
4. 8B 模型 float32→8-bit 内存数？质量？
   → \~30GB→\~8.45GB（约 1/4）；简单任务肉眼无差。
