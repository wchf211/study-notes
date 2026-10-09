# Deploy：安全 + 推理 Infra + 扩缩容（Ch5）

> 上线三件套：别被搞（安全）、跑得动（infra）、挂了能活（扩缩容套路全是分布式老熟人）。

## 1. 安全与合规

新攻击面：
- **Prompt injection**：用户输入/检索文档里藏"忽略之前的指令"劫持模型——RAG 里**你自己的文档也是不可信输入**；
- **Jailbreaking**：精心构造输入绕过安全训练；
- 数据外泄（套话问出机密）、日志里漏 secret。

防御：不可信内容定界+校验、tool 最小权限、记日志/embedding 前 PII 脱敏、短 retention、传输+静态加密、访问控制、全链路审计。
合规：GDPR/HIPAA/SOC 2；数据驻留影响供应商选择；**被遗忘权对 embedding/向量索引是真难题**——早设计（按用户 namespace、重建索引流程）。
高风险动作：**人审批，不自治**。

## 2. 推理 Infra：硬件、经济、延迟

两个关键延迟数：
- **TTFT（time to first token）**：多久用户看到第一个字。>\~2s 感觉坏了（流式能掩盖后面）。
- **TPS（tokens/sec）**：生成速度。decode 是 memory-bandwidth-bound；5–10 tps ≈ 阅读速度，\~20 顺滑，\~3 卡顿。

**VRAM 数学**：显存 ≈ 参数量（B）× 每参数字节数。Llama-3-8B FP16（2B/参数）≈ 16GB；70B ≈ 140GB，单张 80GB A100 装不下。
**Quantization（量化）**：INT8 内存减半、INT4 再减半（8B → \~4GB），小代价换装下。
**KV cache**：存下来的 attention 状态，随对话变长膨胀（4K tokens 约 2–4GB）——经典 OOM 刺客。

**经济学**：API 是可变 OpEx（如 1000 query/天 × 700 tokens，$0.30/M ≈ $0.21/天）；自托管是固定 $/小时（g5.xlarge \~$1/hr，闲着也 \~$700+/月）。**自托管只在高 utilization 时赢**，utilization = 真在生成的时间占比，靠 batching 提。
**Data parallelism**（多副本提吞吐）vs **model parallelism**（一张模型切多卡，装下大模型）。

决策：**先 API，月账单超过 GPU 租金线再自建**。TTFT、TPS、队列深度、单 query 成本当一级指标盯。

## 3. Hosting、Serving、Scaling

标准 serving 栈：FastAPI 守边 → 推理引擎（**vLLM / TGI / TensorRT-LLM**，做 continuous/async batching——把多请求的 decode 步凑一起喂 GPU）→ 容器化 → K8s GPU-aware autoscale。

分布式老套路全适用：
- 健康/就绪探针（模型 load 完再接流量）；
- 请求队列+超时；backoff 重试；
- 网关限流配额；
- **semantic/prompt caching**：support bot 里重复 query 多，缓存=成本和延迟趋近于零。

发布：canary / blue-green / shadow，**回滚一键**。
扩缩容看 **GPU 饱和度 + 队列延迟**，不是 CPU。
**每个请求记日志**：模型版本、prompt 版本、延迟分解、token 数、成本——不记就没法优化和 debug。

## 自测 4 问（带答案要点）
1. RAG 里为什么"自己的文档也是不可信输入"？
   → prompt injection 藏检索文档里；防御：定界+校验+最小权限+PII 脱敏。
2. TTFT 和 TPS 各是什么？多少算"感觉坏了"？
   → 首 token 时间（>\~2s 坏）、生成速度（\~3 tps 卡）；decode 是 memory-bound。
3. VRAM 数学？KV cache 为什么是刺客？
   → 参数量×字节数；KV cache 随对话膨胀，4K tokens 2–4GB。
4. 扩缩容看什么指标？发布怎么才安全？
   → GPU 饱和度+队列延迟；canary/blue-green/shadow+一键回滚。
