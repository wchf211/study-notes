# 估算 + SCALED 框架（Ch3–4）：面试数学和答题骨架

> GenAI 面试的两大基本功：back-of-the-envelope 算数（训多久、要多少卡、多少钱），和 SCALED 六步答题框架。

## 1. 训练估算：万能公式

**T(秒) = (N_params × 6 × E × D × C) / (R_ops × N_GPUs)**

直觉拆解：
- **6 FLOPs/参数**：forward 约 2×N，backward 约 4×N（反向传播要算梯度，更贵），加起来训练一个样本约 6×N。
- **C（复杂度乘子）**：文本训练 C=1；GAN C=1；**diffusion C=20–100**（去噪要迭代很多步）——这就是 diffusion 训起来贵的原因。
- **R_ops**：A100 FP16 ≈ 312 TFLOPS；H100 TF32 ≈ 989 TFLOPS。

例：Llama 2-70B ≈ 170 万 A100 GPU-hours（单卡 \~195 年，1 万卡 \~1 周），训练成本 \~$3.9M。
面试口径：**背下 6 FLOPs/参数和公式**，面试官会让你现场算"给定时间窗口要多少卡"。

## 2. 部署估算：存储 → 服务器 → 带宽

直觉：按 **存储、服务器、带宽** 这个顺序算，假设先摆明（FP16 还是 INT8 结果差一倍）。

- **模型大小** = 参数量 × 每参数字节数（FP16=2B）：3B 模型=6GB，8.1B=16.2GB。
- **100M DAU 例**：TRPS ≈ 11,574（1 亿 × 10 请求 / 86400 秒）；3B 模型 500 tokens ≈ 9.6ms → 104 QPS/GPU → \~112 台 A100。
- **带宽不对称**：文本场景 ingress 185 Mbps vs egress 926 Mbps（输出比输入大）；**先算带宽再算服务器**的场景：音频/视频（egress-heavy）。
- 优化能砍服务器数：量化/batching/sharding 大约能砍一半。
- 面试口径：永远先说假设（精度、DAU、请求量），再按存储→服务器→带宽算。

## 3. SCALED：六步答题框架

每道 GenAI 设计题都按这个骨架走：

- **S = System requirements**：功能需求 + 非功能需求（延迟、可用性、安全……）。
- **C = Choose model**：选模型是**最关键的决定**——开源 vs 闭源（省 vendor 依赖 vs 省运维负担）、大 vs 小（质量 vs serving 成本）。
- **A = Acquire/prepare data**：数据从哪来、怎么清洗、pipeline 多久。
- **L = Leverage model**：分布式训练 + 用对的指标评估（BLEU/FID……）。
- **E = Estimate resources**：模型大小、存储、服务器、带宽（就是上面的数学）。
- **D = Design system**：自底向上模块化画系统，**训练和部署分开画**。

## 4. 面试评分点

考的不是背架构/背 loss，而是：**表达清晰、trade-off 意识、应变能力**。经典流程：需求 → 选模型 → 训练 infra → 推理管线 → 演进。两大坑：过度设计、一头扎进 ML 理论。每个选择都要带上 cost 和约束。

## 自测 4 问（带答案要点）
1. 训练时间公式是什么？6 从哪来？
   → T=(N×6×E×D×C)/(R×GPUs)；forward 2×N + backward 4×N。
2. Diffusion 训练为什么比文本贵？
   → C=20–100（迭代去噪步数乘子）；推理时可降到 20–100 步（DDIM/DPM-Solver）。
3. 部署估算的顺序和第一个要声明的假设？
   → 存储→服务器→带宽；先声明精度（FP16/INT8）、DAU、请求量。
4. SCALED 里哪一步最关键？为什么？
   → C（选模型）：决定质量/成本/运维负担的全局 trade-off。
