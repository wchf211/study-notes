# Eval + Deploy + Multimodal + Safety（Modules 7–11）

> 这一章是这门课最有增量的部分：评估当回归测试、模型包成 API 微服务、成本控制当产品功能、eval 进 CI/CD。读的时候重点看 Module 8。

## 1. Evaluation：评估即回归测试

**Eval set（评估集）**：小而有代表性的测试集，覆盖 success、edge（边界）、adversarial（对抗）case，配 expected answers / rubrics（预期答案 / 评分标准）。产品变了，eval 集要跟着刷新。失败模式：eval 只镜像训练数据。Fix：覆盖真实分布 + 对抗 case。

**Automated evaluation（自动化评估）**：deterministic checks（确定性检查：schema、exact match、retrieval presence）+ LLM-as-judge（用模型按 rubric 打分，需 calibration 校准）+ 抽样人工 review。失败模式：信一个没校准的 judge。Fix：judge 要校准，人要抽查。

**LLM A/B testing**：线上随机分流比 prompt / 模型变体，配 guardrails + 预先定好的 success metrics；注意 novelty（新奇效应）和 segment effects（分群效应）。失败模式：把噪声读成胜利。Fix：预先定指标 + 足够样本 + 看分群。

**Hallucinations（幻觉）**：流利但无支撑的答案。降法：grounding（基于检索证据）+ evidence 缺失时 abstain（拒绝回答）+ claim verification（声明校验）；按任务测 hallucination rate（幻觉率）。失败模式：流利但无支撑的答案通过 review。Fix：证据 + 拒绝 + 校验，三件套。

## 2. Deployment：模型包成 API 微服务（本课最 distinctive 的想法）

**API wrapper（API 包装）**：模型藏在一个稳定的服务边界后面：request / response schema、timeout、retry、auth、versioning。好处：换模型 / 换 provider 不破坏客户端。失败模式：把 provider 的错误 / 形状直接漏给客户端。Fix：边界处翻译、收敛。

**Caching, limits, costs（缓存、限流、成本）**：cache exact / semantic repeats（精确 / 语义重复）；per-user / per-key rate limit（限流）；per-request + per-user budget（预算）+ alert（告警）；graceful degradation（优雅降级）。失败模式：cache 返回过期 / 个性化答案。Fix：缓存 key 要含个性化维度 + TTL。

**Logging and observability（日志与可观测）**：记 prompt / model / version、token、latency、cost、用的 retrieval、outcome（都要 redacted 脱敏）；端到端 trace（追踪）+ drift alert（漂移告警）。失败模式：日志里记 PII / 原始 prompt 没脱敏。Fix：redaction 是必选项，不是可选项。

**CI/CD for AI apps**：pipeline = lint / test prompts and code → eval regression（评估回归）→ stage deploy → canary（金丝雀）→ 回滚条件是 eval / cost / latency 回归。失败模式：prompt 改动没过 eval 门就上线。Fix：eval 进 CI/CD，回归就拦。

## 3. Multimodal：每种模态先各自验，再合

**Vision-Language Models / VLM（视觉语言模型）**：吃 image + text，回答 / 描述 / grounding（定位图中内容）。要用视觉检查验，不只文本 eval。失败模式：hallucinate 图中没有的细节。Fix：视觉校验。

**Image generation and editing（图像生成与编辑）**：按文本 prompt + 参考图生成 / 编辑，控 style、size、安全 filter；输出要验。失败模式：以为 prompt = 精确输出。Fix：把生成当草稿，验了再用。

**Speech integration（语音集成）**：speech-to-text → LLM → text-to-speech 链起来，配 streaming、interruption（打断）、latency budget（延迟预算），语音 UX 才成立。失败模式：轮次延迟太长，对话断了。Fix：延迟预算 + 流式 + 可打断。

**Multimodal integration（多模态集成）**：一个 flow 里路由 / 合并文本、图像、音频，共享 context，每种模态各自有检查 + fallback。失败模式：一个模态的错静默污染答案。Fix：跨模态失败处理，先各验再合。

## 4. Responsible AI：Defense in depth（纵深防御）

**Adversarial prompt safety（对抗性 prompt 安全）**：防 injection、jailbreak（越狱）、data exfiltration（数据外泄）：hierarchy（层级）+ 输入/输出检查 + least privilege + red-teaming（红队测试）。失败模式：把 user / tool 文本当指令信。Fix：对抗测试当日常。

**Data privacy and PII handling**：最小化收集、模型调用 / 日志前 detect / redact PII、尊重 retention / deletion（保留 / 删除）和 regional rules（地区规则）。失败模式：PII 漏进日志或训练数据。Fix：调用前脱敏 + 保留策略。

**Bias and fairness（偏见与公平）**：跨 group / prompt 测输出，看 skewed errors（偏斜错误），修 data / prompt / threshold，而不是只怪模型。失败模式：单侧测试集藏住伤害。Fix：多群测试。

**Guardrails and content moderation（护栏与内容审核）**：分层 input / output filter、policy check、refusal / abstain 路径、高风险人工 review；调 over-blocking vs under-blocking（拦太多 vs 拦太少）的平衡。失败模式：只有一个 filter 当唯一防线。Fix：纵深防御，没有单点。

## 5. 结尾：Build → Evaluate → Operate

收尾没有新技术，只给 loop：ship 一个小而有边界的系统 → measure（度量）→ harden（加固）→ 再扩展。先小、先测、先加固，再扩大。

## 这门课怎么教的（Modules 7–11）

Module 7 用“怎么知道它变好了？” hook，顺序是 eval 集 → 自动化 → 线上 A/B → 幻觉，最值得偷的是“eval 当回归测试”。小坑：幻觉放在 A/B 之后，而不是和 grounding / RAG 放一起。Module 8 用“模型包成 API，必须又稳又便宜” hook，顺序是 API 包装 → 成本控制 → 可观测 → CI/CD，最值得偷的是 API 边界 + 预算/告警的生产心智模型 —— 全课对 YouTube 讲解最有用的模块。小坑：CI/CD 课很压缩。Module 9 用“能看能听的 app” hook，每种模态一个 intro，最值得偷的是“合并前每种模态先各自验”。小坑：广度大于深度。Module 10 用“对抗用户 / 敏感数据” hook，顺序是攻击 → 隐私 → 偏见 → 护栏，最值得偷的是纵深防御分层。小坑：偏见课最抽象，需要一个具体 worked example。Module 11 只有 recap。

## 自测 4 问（带答案要点）
1. Eval 集要覆盖哪三类？LLM-as-judge 为什么必须校准？
   → success / edge / adversarial；没校准的 judge 不可信 + 人要抽查。
2. API wrapper 为什么是生产关键？边界处要收敛什么？
   → 稳定服务边界，换模型 / provider 不破客户端；收敛 provider 错误 / 形状，不直漏给客户端。
3. 成本控制三件套 + 降级？
   → 缓存（精确 / 语义重复）+ 限流 + 预算/告警；graceful degradation。
4. Guardrails 为什么不能只有一个 filter？纵深防御包括哪几层？
   → 单点会漏；input / output filter + policy check + refusal / abstain + 高风险人工 review。
