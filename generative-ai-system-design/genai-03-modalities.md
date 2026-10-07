# 六大 Modality 实战（Ch5–10）：每个都是"训练 + 部署"两套账

> 同一套 SCALED 骨架，套在文本/图像/语音/视频/字幕/ASR 六个 modality 上。看的时候抓三样：**选了什么模型（为什么）、关键数字、部署的子系统**。

## 1. Text-to-Text（Llama 3.2 3B）

- **训练**：3B × 6 × 1000 epochs × 2 亿行 / 156 TFLOPS ≈ 267 天（单卡）→ 32 卡 \~8.35 天。数据 pipeline：2 亿 × 2.1ms ≈ 117h（16 台机器 \~7.3h）。评估：perplexity、BLEU、ROUGE。
- **部署**：6GB 模型；\~9.6ms/请求 → 104 QPS → 112 台 A100；ingress 185 Mbps / egress 926 Mbps。
- 子系统：prompt 处理（embedding + **LRU 语义缓存** + Pinecone 向量检索）→ long-term memory（cosine 检索 + Redis 会话 + Neo4j 知识图谱）→ 模型 host + moderation（版本管理、GDPR/HIPAA 审核，**审核不能加用户可感知的延迟**）。
- 面试画法：API 网关 → LB → embedding（缓存）→ 检索 → 模型 → 审核 → 用户。

## 2. Text-to-Image（Stable Diffusion 3.5 Large, 8.1B）

- **为什么 diffusion 赢了 GAN**：GAN 有 mode collapse（只会画几种图）且训练不稳定；diffusion 是"学去噪"，稳定、可控、多样。直觉：从纯噪声开始，一步步"猜"出更清晰的版本，像雕塑家从大理石里凿出形状。
- **省钱的关键**：**冻结 CLIP 文本编码器**（不训它，省巨量算力）+ latent diffusion（在压缩空间去噪，不在像素空间）。
- **训练**：1000 个去噪步（C 乘子大）；1 卡 5,688 天 → 512 张 H100 \~11.11 天。评估：FID（分布真实度）、prompt 对齐精度、多样性召回。
- **部署**：16.2GB 模型；**存储是重头**：1000 TB/天 + 25% → 37.5 PB/月；每张图 \~5.2 秒 GPU（100 步）；61 台 A100。推理优化：DDIM/DPM-Solver 把步数砍到 20–50（见第 5 篇），prompt 先过滤再进贵推理。
- 面试口径：先讲存储数学，再讲步数预算。

## 3. Text-to-Speech（Fish-Speech Dual-AR, \~2B）

- **架构直觉**：Dual-AR = 慢 transformer 管语言（韵律/停顿）+ 快 transformer 管声音细节，像"先想好怎么念，再控制嗓子"。对比：Tacotron 2（MOS 4.5 但闭源）vs Fish Speech（MOS \~4.05，开源多语言）。
- **训练**：C=1000（10 秒音频 × 100）；单卡 890 天 → 64 卡 \~13.9 天。评估：**必须有人工 MOS**（自动指标听不出自然度）+ WER（用 Whisper 转写回来验）。
- **部署**：4GB 模型；**egress-heavy**（音频重）：250 TB/天 → 7.5 PB/月，egress 27.8 Gbps；148 台 A100。子系统：文本归一化 → 音素转换 → 声学模型 → vocoder（HiFi-GAN）→ 波形。
- 面试口径：先算带宽再算服务器；vocoder 选型决定延迟。

## 4. Text-to-Video（Mochi 1, 10B + T5-XXL 4.7B）

- **为什么最吃算力**：视频 = 图片 × 时间维。靠 **latent 压缩**（8×8 空间 + 6× 时间 → 12 通道）把 44,520 个 token 的 3D attention 变得可算；文本编码器冻结省钱。
- **训练**：C=15,000（5 秒视频）；5000 张 GPU \~21 天。评估：LPIPS（感知损失）、光流时间一致性、FVD、MOS。
- **部署**：29.4GB 模型；1562.5 TB/天（\~47 PB/月）；naive 算法 8,208 台 A100 —— **面试时要主动说"这个数太天真"，然后讲怎么砍**（量化、蒸馏步数、TensorRT、并行）。
- 面试口径：先讲压缩策略 + 步数数学，这是视频题的命门。

## 5. Image Captioning（BLIP-2 式 VLM）

- **Q-Former 直觉**：ViT 看图 → Q-Former 当"翻译"把视觉 token 压缩成 LLM 能懂的 embedding → 冻结的 LLM（FLAN-T5）直接生成描述。**只训翻译层（\~1B），不训 LLM**——便宜又好用，这是标准"桥接"范式。
- **训练**：单卡 \~9 天 → 8 卡 \~1.1 天。评估：BLEU、CIDEr（TF-IDF 加权）、ROUGE。
- **部署**：8GB 模型；**ingress-heavy**（图进、文本出）：312.5 TB/天，ingress 23 Gbps；30 台 A100。Beam search 换 caption 质量（拿延迟换）。
- 面试口径：先算网络（ingress），再说预计算 embedding + TensorRT。

## 6. ASR（Whisper v3 式，\~1.5B）

- **架构直觉**：log-mel 频谱图 → encoder → 带 cross-attention 的 decoder 自回归吐词。**multitask token**（`<transcribe>`/`<translate>`/语言 token）让一个模型干转写+翻译+说话人分离多件事。
- **为什么现代 ASR 强**：大规模自监督预训练（不依赖标注），抗噪音/口音/小语种。
- **训练**：2M 行（Common Voice 5 万小时）；单卡 \~44.5 天 → 8 卡 \~5.6 天。评估：**WER**（词错率，增删替，越小越好）是 headline 指标。
- **部署**：3GB 模型；1.25 PB/天；30 秒音频 \~50ms → 579 台 + 20–30% buffer → 700–750 GPU。子系统：音频接入（30 秒分块）→ 模型 host（greedy 快 vs beam 准）→ 后处理（标点、置信度、幻觉 flag）。
- 面试口径：Whisper 原生 99 语言 + 自动检测是加分点；50ms/30s 是算服务器数的基准。

## 自测 4 问（带答案要点）
1. Diffusion 为什么取代 GAN？哪两个省钱手段最关键？
   → 稳定/多样/可控；冻结 CLIP + latent diffusion。
2. 六个 modality 里，哪个是存储重头、哪个是 egress 重头、哪个是 ingress 重头？
   → 存储：文生图（PB/天）；egress：TTS/视频；ingress：captioning（图进文本出）。
3. Q-Former 的"桥接"范式是什么？为什么便宜？
   → 只训视觉→语言的翻译层，LLM 冻结复用。
4. 视频 serving 算出 8,208 台 A100，面试时怎么接？
   → 主动说 naive，然后讲量化/蒸馏步数/TensorRT/并行怎么砍。
