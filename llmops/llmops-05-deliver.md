# Deliver：编排 + 质量治理 + 未来（Ch6）

> 上线只是起点。编排（确定性代码管随机部件）、人兜底、治理（出事能分钟级恢复），再看 agent/多模态/效率三个前沿。

## 1. Orchestration：确定性代码链随机部件

**Orchestration** = 检索→组装 prompt→调模型→校验→可能调工具→返回，这条链。
模式：顺序链、**routing**（按 query 类型走不同管线，如 FAQ vs 深度研究）、**fan-out/fan-in**（并行多路检索再合并）、带退出条件的有界循环。

铁律：**控制流全确定性**——LLM 是唯一的非确定组件，这样故障可复现、可 debug。
分布式 hygiene 照单全收：**idempotency**（重试不产生重复副作用）、超时、**circuit breaker**（依赖挂了冷却一阵别 hammer）、fallback（缓存答案/小模型/优雅降级文案）。

结构化输出用 schema 校验，parse 失败重试；管线用 fixture 端到端测，别只测模型。框架（LangChain 类）是方便，**模式才是本体**。

## 2. 质量、HITL、治理

- **Human-in-the-loop（HITL）**：人进工作流审/批输出。高风险决策的兜底——**没有指标能抓住所有故障**，有些错（错退款、错医疗摘要）比人工贵。用置信度阈值+采样，只把高风险切片送人；其余走**quality gates**（展示前的自动检查）。
- **Governance（治理）**：所有 artifact 版本化（prompt/模型/数据集/eval 集）、生产变更要审批、审计日志（谁改了什么、产出什么）。
- **Data drift**：用户 query 的统计特性慢慢变（新话题、新问法），质量静默下滑——盯 query embedding 和反馈率当 drift 信号。
- **Incident**：**kill switch**（一键关功能/切安全模式）+ prompt/模型一键回滚 + 对外沟通预案。

血泪：**每次事故产出一个回归测试**，能加 gate 的加 gate。治理平时像 overhead，出事那天是分钟级恢复的原因。

## 3. 未来三前沿（+ 不变的纪律）

1. **Agents**：Thought → ToolCall → Observation 循环，能动真格（建 ticket、调 API）。新故障：无限重试、token 烧穿、无界副作用——naive agent 能无人值守烧几百刀或污染生产。LLMOps 答案：**受限执行**——max_steps、tool allowlist、沙箱执行（副作用只许在容器/microVM/WASM，不许上宿主机）、可审计 trace。
2. **多模态**：图像/音频/视频输入打碎文本那套 ingestion/embedding/评估/安全（图里能藏 PII、凭证、prompt injection）。防御：OCR/视觉预处理 → PII 脱敏 → 重活用异步队列甩出请求路径 → 检索层保持文本为主（可调试）。
3. **效率**：**Distillation（蒸馏）**——大模型当老师训小学生，大部分质量、零头成本；本身是条 training pipeline，也要版本化数据集+回归 eval+回滚。**Edge AI**：端侧推理，延迟≈0、隐私好（数据不出设备），代价是可观测性差+版本漂移。

不变的纪律：概率系统配确定性护栏；评估先于部署；artifact 版本化；反馈闭环；**复杂度是成本中心**。
收官：**知道什么时候不加能力**——不是每个系统都需要 agent、多模态、前沿大模型。

## 自测 4 问（带答案要点）
1. Orchestration 的铁律是什么？为什么？
   → 控制流全确定性，LLM 是唯一非确定组件；故障可复现可 debug。
2. HITL 什么时候必须有？怎么省人力？
   → 高风险决策；置信度阈值+采样只送高风险切片。
3. Agent 生产化的四个约束？
   → max_steps、tool allowlist、沙箱执行、可审计 trace。
4. 出事时哪三件套决定恢复速度？
   → kill switch、一键回滚、审计日志；每次事故产出回归测试。
